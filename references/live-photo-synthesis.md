# Live Photo (动图) 合成完整工作流

## 触发条件

解析抖音链接返回 `data.type === "live"` 且 `data.live_photo` 非空。

## 自动合成（服务端异步，无需FFmpeg）

服务端 `parse.php` 已内置自动合成逻辑。检测到 `type=live` 时：

1. 服务端将 **所有 live_photo 段**（不再是仅第1个）写入后台任务
2. 返回 `synthesized_url=null` + `synthesize_job_id`
3. AI 轮询 `check_job.php` 获取合成结果

**AI流程：**

1. 调用 `GET /api/parse.php?key=密钥&url=抖音链接`
2. 检查返回的 `synthesize_job_id`
3. 每3-5秒轮询 `GET /api/check_job.php?key=密钥&job_id=xxx`
4. `code=200` 时下载 `data.url` 的视频文件
5. 用 MEDIA 发送

> ⚠️ 走自动合成就行，AI 不需要装 FFmpeg。服务端后台跑，不阻塞 parse.php 响应。
> ⚠️ 4段素材合成约需 60-90秒（2核/4G服务器）。轮询逾时建议发原视频不等。

### 多段合成

每个 `live_photo[]` 条目独立对待：video→image→video→image→... 依次轮播。

| 场景 | 效果 |
|------|------|
| 1个 live_photo | 1段动图 + 1段静图 交替，1个循环 |
| 4个 live_photo | 4段动图各自配对应静图 依次轮播，4个循环 |

### 防重复

parse.php 内置视频去重锁（md5(video_url+image_url) + 原子mkdir），同一视频 5分钟内只合成一次，不会堆叠多个 ffmpeg 进程。

## 多素材处理（多个 live_photo[] + images[]）

服务端自动处理所有 live_photo 条目：
- 每个条目独立下载素材
- 逐段生成动图/静图循环
- 全部 concat 为单一视频
- 用第一个视频的原声做背景音（可选叠加配乐）

## Root Cause：为什么"闪"

**症状**：动图合成后，在动图片段→静图片段→动图片段的交替切换处出现画面突然变化（类似"闪一帧残影"）。

**根因**：之前每个片段从不同时间点取内容。
- `seg_m0` → ss=0（第0秒开头）
- `seg_m1` → ss=6（第6秒的画面）
- `seg_m2` → ss=12（第12秒的画面）

每段开头画面不同 → 播放器在交替拼接时画面突变 = 用户感知的"闪"。

**修复**：全部 `ss=0`，每段都是同一段源的开头。
- 动图分段全部 ss=0 → 无论第几次切到动图，开头都一样
- 静图分段全部 ss=0 → 静图始终是同一帧

## 关键规则（踩坑总结）

| 规则 | 说明 |
|------|------|
| **所有分段 `ss=0`** | 全部从第0秒取同一段源。反例：不同时间点 → 画面突变 = "闪" |
| **统一30fps** | 动图和静图都必须 `-r 30`，否则拼接后播放器卡死 |
| **全片重编码** | 禁止 `-c copy`，必须 `-c:v libx264` 重编码为单一流 |
| **视频分段用 `-an`** | 音频不经过分段，从连续源单独混音 |
| **`-fflags +genpts`** | concat 时添加 genpts 确保 PTS 干净 |
| **`-g 1`** | 每帧都是关键帧，切换时不会残留前一帧残影 |
| **无配乐也正常合成** | 配乐 URL 空时自动用 `-map "[orig]"` 代替 `-map "[aout]"` |
| **多段互不冲突** | 每段独立下载 `live_{$pi}.mp4` / `tail_{$pi}.jpg`，互不覆盖 |

## 服务端文件

| 文件 | 作用 |
|------|------|
| `api/async_job_handler.php` | 后台 ffmpeg 合成，不占 FPM 进程 |
| `api/check_job.php` | AI 轮询接口 |
| `/tmp/synth_jobs/{id}.photos.json` | 多段素材列表 |
| `/tmp/synth_locks/{hash}` | 原子去重锁 |
| `/var/www/short-videos/synth/{id}.mp4` | 合成输出 |
