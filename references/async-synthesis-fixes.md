# Async Synthesis Fixes (2026-07-07)

## Root Cause: External AI Retry Flood

**Symptom:** 5 parallel ffmpeg processes all synthesizing the same video, RAM exhausted, all jobs stuck forever.

**Chain of events:**
1. External AI calls `parse.php` for a live photo
2. `parse.php` spawns `async_job_handler.php` via `exec()` with a unique `job_id`
3. AI's HTTP request times out (30s polling window)
4. AI retries the same URL → `parse.php` spawns ANOTHER async job with a new `job_id`
5. Repeat 3-5 times → 5 `async_job_handler.php` processes running the SAME ffmpeg operations
6. All 5 ffmpeg processes compete for CPU and RAM, making each one take 5x longer
7. None finish within the 30s polling window → AI thinks synthesis failed
8. Deadlock: infinite retries spawn infinite jobs

## Fix 1: Dedup Lock (parse.php)

**File:** `/var/www/short-videos/api/parse.php`

**Approach:** Use atomic `mkdir()` as a POSIX lock keyed by `md5(video_url + image_url)`.

```php
$lock_key = md5($video_url . $image_url);
$lock_dir = "/tmp/synth_locks/{$lock_key}";
$lock_info = "/tmp/synth_locks/{$lock_key}.info";

// Check existing job within 5 minutes
if (is_dir($lock_dir)) {
    $existing_job = @file_get_contents($lock_info);
    $age = time() - filemtime($lock_dir);
    if ($existing_job && $age < 300) {
        // Reuse existing job_id — don't spawn another
        $job_id = trim($existing_job);
        // Check if it's still processing (202) or finished (200)
    }
}

// Only spawn if no lock or expired
if (@mkdir($lock_dir, 0755)) {
    file_put_contents($lock_info, $new_job_id);
    exec("php ... {$new_job_id} ... &");
}
```

**Key details:**
- Locks stored at `/tmp/synth_locks/` (owned by www-data)
- `mkdir()` is atomic on Linux — race-safe
- Lock TTL: 5 minutes (300s), auto-expires
- If lock exists and job is still processing (202), reuses the job_id
- If lock exists and job completed (200), spawns a new one (user wants fresh video)
- Cleaned up `/tmp/synth_locks/*` directory created on first use

## Fix 2: Audio Mixing — Missing No-Music Branch

**File:** `/var/www/short-videos/api/async_job_handler.php`

**Bug:** When `music.mp3` is empty/0 bytes (URL expired or unreachable), `$has_music` is false, but the code still used `-map "[aout]"` which doesn't exist in the filter graph.

**Original code (broken):**
```php
$audio_cmd .= ' -filter_complex "[1:a]volume=1.7,atrim=0:{$td}[orig]"';
if ($has_music) $audio_cmd .= ';[2:a]volume=1.0,...[aout]';
$audio_cmd .= ' -map "[aout]" -map 0:v ... final.mp4';
// ^^ [aout] doesn't exist when $has_music=false!
```

**Fixed code:**
```php
if ($has_music) {
    // 3 inputs: video.mp4 + live.mp4 + music.mp3
    $audio_cmd .= ' -filter_complex "...[orig];...[bgm];[orig][bgm]amix...[aout]"';
    $audio_cmd .= ' -map "[aout]" -map 0:v ... final.mp4';
} else {
    // 2 inputs: video.mp4 + live.mp4 (no music)
    $audio_cmd .= ' -filter_complex "[1:a]volume=1.7,atrim=0:{$td}[orig]"';
    $audio_cmd .= ' -map "[orig]" -map 0:v ... final.mp4';
}
```

## Observations

- Douyin music URLs (`lf26-music-east.douyinstatic.com`) can sometimes return empty/0-byte responses
- `filesize()` check after download: `$has_music = filesize("$work/music.mp3") > 0`
- When music is available, it mixes original audio (170%) + music (100%), weighted 1.7:1.0
- When music is not available, it just uses original audio at 170% volume
- The `2>/dev/null` at end of ffmpeg commands hides ALL errors — this is by design (avoids noise in PHP output), but makes debugging harder. To debug, remove `2>/dev/null` and check the PHP-FPM slow log.

## Fix 3: Multi-Segment Live Photo (All Entries)

**File:** `/var/www/short-videos/api/async_job_handler.php` (+ parse.php args changed)

**Bug:** `parse.php` only passed `live_photo[0]` (first entry) to the async handler. Videos with 3-4 live_photo entries only displayed the first one.

**Fix (parse.php):** Instead of extracting individual args, write ALL live_photo entries as a JSON file:

```php
file_put_contents("/tmp/synth_jobs/{$job_id}.photos.json", 
    json_encode($parsed['data']['live_photo'], JSON_UNESCAPED_UNICODE));
```

Then call the async handler with just `job_id`, `key`, and `music_url` (no individual video/image URLs):

```php
exec("php {$script} {$job_id} {$esc_key} {$esc_mus} > /dev/null 2>&1 &");
```

**Fix (async_job_handler.php):** Read the JSON file and iterate:

```php
$live_photos = json_decode(file_get_contents($photos_file), true);
$total = count($live_photos);

for ($pi = 0; $pi < $total; $pi++) {
    $video_url = $live_photos[$pi]['video'];
    $image_url = $live_photos[$pi]['image'];
    
    // Download each video + image pair
    // Create motion_loop + still_loop for each
    // Cut 1 segment pair per entry: m{$pi}.mp4 + s{$pi}.mp4
}

// Concat all segments: m0→s0→m1→s1→m2→s2...
```

**Result structure:**
```json
{
  "duration": 24.33,
  "segments": 4,  // number of live_photo entries
  "url": "http://101.32.98.240:8080/synth/synth_xxx.mp4"
}
```

**Performance note:** Each additional live_photo entry adds ~15-20s of ffmpeg time on a 2-core server, since each entry needs: motion loop creation + still loop creation + 2 segment cuts.

## Fix 4: `-movflags +faststart` for Mobile Playback

**File:** `/var/www/short-videos/api/async_job_handler.php`

**Bug:** ffmpeg by default puts the moov atom (metadata) at the end of the MP4 file. Desktop players tolerate this, but mobile video players (iOS/Android) may crash or force-stop playback mid-stream because they can't read the metadata in time.

**Fix:** Add `-movflags +faststart` to the audio mixing (final) ffmpeg command:

```php
$audio_cmd .= " -c:v copy -c:a aac -b:a 96k -movflags +faststart "
    . escapeshellarg("$work/final.mp4") . " 2>/dev/null";
```

**Verification:** After encoding, the moov atom is at the beginning of the file and the player can start streaming immediately without downloading the entire file.

## Related Config

PHP-FPM settings that affect this:
- `request_terminate_timeout = 300s` — must exceed total ffmpeg time
- `pm.max_children = 30` — enough for concurrent requests + background jobs
- `pm.max_requests = 200` — prevents memory leak accumulation

## Monitoring

- Check stuck jobs: `ls -lt /tmp/synth_jobs/*.status`
- Check job results: `cat /tmp/synth_jobs/<job_id>.result`
- Check ffmpeg count: `ps aux | grep ffmpeg | grep -v grep | wc -l`
- Check RAM: `free -h` (ffmpeg can use 300MB+ per instance on 720p)
