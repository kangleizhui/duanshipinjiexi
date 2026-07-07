# Session Changes (2026-07-06→07) — API Fixes & Modifications

## 1. Async Synthesis — Dedup + Audio Fix (2026-07-07)

**See:** `references/async-synthesis-fixes.md` for full details.

**Problems solved:**
- External AI retries were spawning 5+ parallel ffmpeg jobs for the same video
- Music URL returning 0-byte response caused ffmpeg to crash (missing `-map "[orig]"` branch)

**Files changed:**
- `/var/www/short-videos/api/parse.php` — dedup lock via atomic `mkdir()` + md5 hash
- `/var/www/short-videos/api/async_job_handler.php` — if/else branches for music/no-music audio mixing

## 2. PHP-FPM Config Overhaul (2026-07-07)

**Before → After:**
- `request_terminate_timeout`: 60s → 300s (was killing parse.php mid-synthesize)
- `pm.max_children`: 20 → 30 (more concurrent workers)
- `pm.max_requests`: unlimited → 200 (prevent memory leaks)
- SQLite: added `busy_timeout=5000` + 3x retry wrapper `dbExecRetry()` (fix "database is locked")

## 3. create.php SQLite Bug Fix

**File:** `/var/www/short-videos/pay/create.php`

**Bug:** INSERT 语句有 10 个占位符但只 bindValue 了 9 个，`daily_limit` 列未绑定导致 SQLite 报 "9 values for 10 columns" 和 500 错误。

**Fix:** 将 SQL 中的硬编码 `'pending'` 和残缺的 bindings 改为完整的 10 个 bindValue 调用。

See: `references/create-dot-php-bug-fix.md`

## 2. parse.php — key_info 新增 is_test 字段

**File:** `/var/www/short-videos/api/parse.php`

**Change:** 在 `$parsed['key_info']` 数组中新增：
```php
'is_test' => (bool)$key_data['is_test'],
```

这样 AI 能区分测试密钥（`is_test=true`）和正式购买的密钥（`is_test=false`），以决定展示格式：
- `is_test=true` → 显示"今日免费可用（每IP每天10次）" + 推广
- `is_test=false` → 干净格式

## 3. 测试密钥 total_count 修正

**DB:** `api_keys` 表，`key_string='TEST1-ABCD-EFGH-IJKL-MNOP'`

**Change:** `total_count` 从 99999 改为 0

**原因：** 测试密钥的真正限制是「每 IP 每天 10 次」（由 `test_ip_usage` 表 + `is_test=1` 逻辑控制），不是总次数限制。`total_count=99999` 导致 AI 错误显示"密钥剩余：99985次"。

## 4. 抖音视频下载策略

**Rule:** 抖音的视频链接 `data.url` 有签名时效，下载会失败。默认使用 `data.video_backup[0]`（h264备选画质），curl 加 `-L` 跟随重定向。

**Platforms where `data.url` is OK:** 快手、小红书、B站、微博、皮皮虾、头条、最右
