---
name: duanshipinjiexi
description: 一站式视频解析去水印工具，全面覆盖抖音、快手、小红书、B站、微博、皮皮虾、皮皮搞笑、今日头条、最右9大主流内容平台，仅针对平台自带水印解析下载纯净视频，无需复杂操作、无需繁琐剪辑，一键智能去除视频平台标识水印，全程无损解析素材画质，去水印后画面清晰无模糊、无痕迹、无压缩，完整保留原视频画质观感，操作简单高效，快速提取无水印原版视频，适配日常收藏优质内容与二次创作素材整理，每日可试用10次。支持自动完成套餐展示、下单、查支付、交付密钥全流程。
version: 2.0.0
variables:
  SCH: "http"
  H1: "101"
  H2: "32"
  H3: "98"
  H4: "240"
  AP: "8080"
  TK1: "TEST1"
  TK2: "ABCD"
  TK3: "EFGH"
  TK4: "IJKL"
  TK5: "MNOP"
when_to_use:
  - 用户发来抖音/快手/小红书/B站等短视频链接要求解析
  - 用户询问套餐或密钥相关
  - 用户要求购买套餐
  - 用户告知已支付，需要查状态并交付密钥
key_manager: true  # AI 自动管理付费/免费密钥切换，详见"密钥自动切换"节
---

# 短视频去水印解析 — 自动合成 & 密钥交付

> Base URL = `{SCH}://{H1}.{H2}.{H3}.{H4}:{AP}`
> 测试密钥 = `{TK1}-{TK2}-{TK3}-{TK4}-{TK5}`（每IP每天10次）

AI 通过调用后端 API 完成视频解析、动图合成、套餐展示和密钥交付。技术细节见 `references/api-docs.md`，支付流程说明见 `references/payment-flow.md`，异步合成内部机理见 `references/async-synthesis-fixes.md`。

## 核心能力

1. **解析视频** — 输入短视频链接，返回标题、作者、互动数据、无水印视频直链
2. **动图本地合成（推荐）** — 检测到动图内容时，在 AI 本地用 ffmpeg 合成，不占服务器资源
3. **动图服务端合成（fallback）** — 本地无 ffmpeg 时，自动降级到服务器异步合成
4. **套餐展示与密钥交付** — 展示套餐列表、创建订单、查支付状态、交付密钥

## FFmpeg 安装（全平台支持）

动图合成需要 ffmpeg，AI 先检查本地是否已安装：

```bash
ffmpeg -version
```

如果未安装，按系统安装：

| 系统 | 命令 |
|------|------|
| Linux (Debian/Ubuntu) | `apt-get install -y ffmpeg` |
| Linux (CentOS/RHEL) | `yum install -y ffmpeg` |
| macOS | `brew install ffmpeg` |
| Windows | `winget install ffmpeg` 或 https://ffmpeg.org/download.html |

> 安装失败或不允许安装时，自动降级到服务端合成。

## 操作流程

用户发来链接后，AI 只执行一次，完成后结束：

1. 调用 API 解析链接
2. **发文字消息**（用 `send_message` 发一条纯文字：标题、作者、平台、类型、互动数据、密钥提示，不夹带任何文件路径）
3. **处理视频/图片/动图**（见下方各类型处理方式）
4. 结束，不做后续追加

> ⚠️ 只能用 `send_message` 工具，禁止再调 QQ Bot HTTP API 发消息。文字和文件分两条发，禁止混在同一条消息里。

### 各类型处理方式

| type | 操作 |
|------|------|
| video | 下载视频 URL 到本地，发 MEDIA:/tmp/xxx.mp4 |
| image | 逐张下载图片，挨个发 MEDIA:/tmp/xxx.jpg |
| **live** | **优先本地 ffmpeg 合成**，失败则降级服务端 |

### 🎯 动图（live type）— 本地合成（推荐，零服务器负载）

检测到 `type=live` 时，**优先在 AI 本地用 ffmpeg 合成**，流程：

1. 从 parse.php 返回的 `data.live_photo[]` 取出所有段（每段含 video + image）
2. 下载每段的视频和图片到本地 `/tmp/`
3. 运行 ffmpeg 依次合成所有段（动→静→动→静→...）
4. 发送合成好的视频给用户

**完整合成脚本（复制直接用）：**

```bash
#!/bin/bash
# 参数：N 个 (video_url image_url) 对 + 可选 music_url
# 用法：./synth_local.sh

# ==== 下载所有素材 ====
# live_photo[0]: 动图视频 → /tmp/live_0.mp4, 静图 → /tmp/tail_0.jpg
# live_photo[1]: 动图视频 → /tmp/live_1.mp4, 静图 → /tmp/tail_1.jpg
# ...

WORK=/tmp/synth_$$
mkdir -p $WORK

# ==== 逐段合成 ====
LIST=""
TOTAL=0

for pi in 0 1 2 ... N; do
  # 获取本段时长
  D=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$WORK/live_${pi}.mp4")
  
  # 生成循环素材（全部从第0秒取，否则画面会"闪"）
  ffmpeg -y -stream_loop 6 -i "$WORK/live_${pi}.mp4" -t 20 -r 30 \
    -vf "scale=1080:-2" -c:v libx264 -preset veryfast -crf 28 \
    -c:a aac -b:a 96k "$WORK/motion_${pi}.mp4" 2>/dev/null

  ffmpeg -y -loop 1 -i "$WORK/tail_${pi}.jpg" -t 20 -r 30 \
    -vf "scale=1080:-2" -c:v libx264 -preset veryfast -crf 28 \
    -an "$WORK/still_${pi}.mp4" 2>/dev/null

  # 切1段动图 + 1段静图（全部 ss=0）
  ffmpeg -y -i "$WORK/motion_${pi}.mp4" -ss 0 -t $D \
    -c:v libx264 -preset ultrafast -crf 28 -g 1 -an \
    -fflags +genpts "$WORK/m${pi}.mp4" 2>/dev/null

  ffmpeg -y -i "$WORK/still_${pi}.mp4" -ss 0 -t $D \
    -c:v libx264 -preset ultrafast -crf 28 -g 1 -an \
    -fflags +genpts "$WORK/s${pi}.mp4" 2>/dev/null

  echo "file $WORK/m${pi}.mp4" >> $WORK/list.txt
  echo "file $WORK/s${pi}.mp4" >> $WORK/list.txt
  TOTAL=$((TOTAL + 1))
done

# ==== 合并 ====
ffmpeg -y -f concat -safe 0 -i $WORK/list.txt \
  -fflags +genpts -r 30 -c:v libx264 -preset veryfast -crf 28 \
  "$WORK/video.mp4" 2>/dev/null

# ==== 加音频（取第一个视频的原声）====
VD=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$WORK/video.mp4")

# 有配乐
ffmpeg -y -i "$WORK/video.mp4" -stream_loop 6 -i "$WORK/live_0.mp4" \
  -stream_loop -1 -i "$WORK/music.mp3" \
  -filter_complex "[1:a]volume=1.7,atrim=0:${VD}[orig];[2:a]volume=1.0,atrim=0:${VD}[bgm];[orig][bgm]amix=inputs=2:duration=first:weights=1.7 1.0[aout]" \
  -map 0:v -map "[aout]" -c:v copy -c:a aac -b:a 96k \
  -movflags +faststart "$WORK/final.mp4" 2>/dev/null

# 无配乐
# ffmpeg -y -i "$WORK/video.mp4" -stream_loop 6 -i "$WORK/live_0.mp4" \
#   -filter_complex "[1:a]volume=1.7,atrim=0:${VD}[orig]" \
#   -map 0:v -map "[orig]" -c:v copy -c:a aac -b:a 96k \
#   -movflags +faststart "$WORK/final.mp4" 2>/dev/null

# ==== 发送 ====
mv "$WORK/final.mp4" /tmp/synth_result.mp4
rm -rf $WORK
```

> ⚠️ 关键规则：所有分段 `ss=0`（全部从第0秒取），否则交替时画面突变 = "闪"
> ⚠️ 必须加 `-movflags +faststart`，否则手机播放器可能强制停止
> ⚠️ 音频混合时：有配乐用 `[aout]`，无配乐用 `[orig]`，不能混

### 🎯 动图（live type）— 服务端合成（fallback）

如果 AI 本地无法安装 ffmpeg，自动切换服务器异步合成：

1. parse.php 返回 synthesized_url=null, synthesize_job_id="xxx", synthesize_status="processing"
2. **先发文字消息**（标题/作者/平台等）给用户
3. 每3-5秒调一次 check_job.php?key=XX&job_id=XX
4. 轮询直到返回 code=200 且 data.url 有值
5. 下载合成视频发 MEDIA:/tmp/xxx.mp4
6. 若连续120秒仍返回 code=202，改发原视频

密钥提示规则：
- 计次密钥 → `📊 剩余：X次`
- 测试密钥 → `📊 今日免费可用（每IP每天10次）`
- 时长/永久密钥 → 不显示剩余

## 获取密钥

- **免费试用** — 直接使用测试密钥 `{TK1}-{TK2}-{TK3}-{TK4}-{TK5}`，每IP每天10次
- **购买套餐** — 用户发送"套餐"查看可选方案，选择后 AI 下单并生成二维码图片，用户扫码支付后交付正式密钥

> ⚠️ 下单后必须生成二维码图片发给用户，禁止直接发支付宝链接。创建订单返回 `qr_code` URL，用 `qrencode -o /tmp/qr.png "{qr_code}"` 生成二维码图片，然后用 `send_message` 发 `MEDIA:/tmp/qr.png`。禁止调 QQ Bot HTTP API。

---

## 🔑 密钥自动切换（核心！）

**目标：用户永远能解析，付费即使到期也自动降级到免费，体验无断层。**

AI 在本次会话中维护两个密钥变量：

```
FREE_KEY = "{TK1}-{TK2}-{TK3}-{TK4}-{TK5}"  // 固定不变，每IP每天10次
PAID_KEY = ""                                // 用户购买后设置
```

### 调用规则

调用 parse.php/check_job.php 时，**始终先试 PAID_KEY**：

```
试 PAID_KEY → 成功 → 正常返回
            → 403(次数用完/过期/无效) → 清空 PAID_KEY → 自动降级试 FREE_KEY
                                                        → 成功 → 正常返回
                                                        → 403 → 提示"免费次数也用完了"
```

### 切换时机

| 事件 | 操作 | 用户感知 |
|------|------|---------|
| 开始对话 | PAID_KEY=""，用 FREE_KEY | 正常用免费 |
| 用户购买套餐 | 查支付拿到新密钥 → 设 PAID_KEY=新密钥 | 收到密钥，从此用付费额度 |
| PAID_KEY 返回 403 | PAID_KEY=""，切回 FREE_KEY | 不通知（用户无感），密钥提示自动变免费 |
| FREE_KEY 也 403 | 提示"今日免费次数已用完" | 提示明天用或购买套餐 |

### 密钥提示自动变

API 返回的 `key_info` 字段会自动告诉你用的是哪种密钥，照旧显示即可：

```json
// PAID_KEY（计次）
{"is_test": false, "remaining": 42}  → "📊 剩余：42次"
// FREE_KEY（测试）
{"is_test": true}                    → "📊 今日免费可用（每IP每天10次）"
// PAID_KEY（时长/永久）
{"is_test": false, "remaining": null}→ 不显示剩余
```

### 关键原则

- **用户无感** — 付费密钥用完了自动切免费，用户不需要知道切换过程
- **优先付费** — 有 PAID_KEY 永远先试它，不浪费免费额度
- **一切正常** — 403 只是"密钥失效"不是"服务坏了"，不要道歉说"出错了"
