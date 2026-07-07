---
name: duanshipinjiexi
description: 一站式视频解析去水印工具，全面覆盖抖音、快手、小红书、B站、微博、皮皮虾、皮皮搞笑、今日头条、最右9大主流内容平台，仅针对平台自带水印解析下载纯净视频，无需复杂操作、无需繁琐剪辑，一键智能去除视频平台标识水印，全程无损解析素材画质，去水印后画面清晰无模糊、无痕迹、无压缩，完整保留原视频画质观感，操作简单高效，快速提取无水印原版视频，适配日常收藏优质内容与二次创作素材整理，每日可试用10次。支持自动完成套餐展示、下单、查支付、交付密钥全流程。
version: 2.2.0
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

> ⚠️ **🔴🔴🔴 绝对规则 🔴🔴🔴**
> **所有 `ffmpeg -ss` 参数必须是 `0`！所有！每一段！**
> ❌ 反例：`-ss 3`、`-ss 6`、`-ss 12` → 每段开头不同 → 切换时画面突变 = "闪"
> ✅ 正解：`-ss 0` → 全部从同一画面开头 → 视觉连续顺滑
> 这条错了合成就是废的，没有例外。
>
> ⚠️ **🔴🔴🔴 第二条铁律 🔴🔴🔴**
> **必须把 `data.images[]` 中所有图片作为静图段合进去！**
> `live_photo[].image` 只是 `images[]` 的子集（通常9张图但只标3段live_photo）
> ❌ 错误做法：只用 live_photo 对应的3张静图 → 漏掉其他6张
> ✅ 正确做法：每个周期 = 1段动图 + **ALL images[]** 静图
> **只有 live_photo[].video 才是真正的动图素材，images[] 里的全是静图**

检测到 `type=live` 时，**优先在 AI 本地用 ffmpeg 合成**，流程：

1. **从 parse.php 返回的 `data.live_photo[]` 取出所有动图段**（每段含 `video`=动图URL + `image`=关联静图URL，但 `image` 只是参考，实际静图用 `images[]`）
2. **从 `data.images[]` 取出所有静图**（全部9张都要展示，不能只取 `live_photo[].image` 那几张）
3. 下载所有素材到本地 `/tmp/`
4. 运行 ffmpeg 合成，**按 `images[]` 的原始下标顺序遍历**，每张图遇到匹配的 live_photo 先播动图再播静图
5. 发送合成好的视频给用户

**合成顺序（关键！）：**
```
每个周期（重复3次）：
  images[0] 静图 → 如有 motion[pi] 匹配 images[0] 则先播动图
  images[1] 静图 → 如有 motion[pi] 匹配 images[1] 则先播动图
  images[2] 静图 → ...
  ...一直到 images[N-1]
  
具体例子（images=9张, live_photo=3段, live_photo[i].image分别匹配images[1],[4],[6]）：
  images[0] → motion[0] → images[1] → images[2] → images[3]
  → motion[1] → images[4] → images[5]
  → motion[2] → images[6] → images[7] → images[8]
```

**数据来源（示例）：**
```json
// parse.php 返回:
{
  "data": {
    "images": [                          // ⚡⚡ 所有静图！不止 live_photo 里的几张
      "https://...img0.jpg",             //      live_photo[0].image = images[1] (子集)
      "https://...img1.jpg",             //      live_photo[1].image = images[4]
      "https://...img2.jpg",             //      live_photo[2].image = images[6]
      "https://...img3.jpg",             //      还有 images[0,2,3,5,7,8] 不在 live_photo 里
      "https://...img4.jpg",             //      全都必须合进去！
      "https://...img5.jpg",
      "https://...img6.jpg",
      "https://...img7.jpg",
      "https://...img8.jpg"
    ],
    "live_photo": [
      {"video": "https://...seg1.mp4", "image": "https://...img1.jpg"},
      {"video": "https://...seg2.mp4", "image": "https://...img4.jpg"},
      {"video": "https://...seg3.mp4", "image": "https://...img6.jpg"}
      // live_photo[].image 只是 images[] 的子集，不要只用这几个！
    ],
    "music": {
      "title": "歌名",
      "author": "歌手",
      "url": "https://...music.mp3",
      "cover": "https://...cover.jpg"
    }
  }
}
```

**验证 ffmpeg 已安装：**
```bash
ffmpeg -version  # 失败则 apt-get install -y ffmpeg
```

**完整 ffmpeg 合成脚本（直接复制执行）：**

> 🔴 关键理解：`live_photo[]` 是动图素材（每段一个短视频），`images[]` 是所有静图。
> `live_photo[0].image` = `images[1]`（只是子集），但还有 images[0,2,3,5,7,8] 不在 live_photo 里。
**完整合成流程（Python + ffmpeg）：**

```python
import os, json, subprocess, shutil

WORK = "/tmp/synth_$$"
os.makedirs(WORK, exist_ok=True)

# 假设已下载：
#   live_photo[0] → {WORK}/live_0.mp4
#   images[0..8]  → {WORK}/img_0.jpg .. img_8.jpg
#   bgm.mp3       → {WORK}/bgm.mp3

IMG_DUR = 1.5  # 每张静图1.5秒
CYCLES = 3     # 3个周期

# 1. 匹配 live_photo[pi].image 在 images[] 中的位置
LP_MAP = {}
for pi in range(TOTAL_LP):
    tail_path = LIVE_PHOTOS[pi]["image"]
    tail_key = os.path.basename(tail_path.split("?")[0])
    for ii, img_url in enumerate(IMAGES):
        img_key = os.path.basename(img_url.split("?")[0])
        if img_key == tail_key:
            LP_MAP[ii] = pi
            break

# 2. 预生成 motion 动图段（全部 ss=0）
for pi in range(TOTAL_LP):
    dur = subprocess.run(
        ["ffprobe", "-v", "error", "-show_entries", "format=duration",
         "-of", "csv=p=0", f"{WORK}/live_{pi}.mp4"],
        capture_output=True, text=True).stdout.strip()
    dur = float(dur) if dur else 3.0
    LP_DURS[pi] = dur
    
    # stream_loop 7次 ≈ 20s
    subprocess.run(f"ffmpeg -y -stream_loop 6 -i {WORK}/live_{pi}.mp4 "
        f"-t 20 -r 30 -vf \"scale=1080:-2\" -c:v libx264 -preset veryfast "
        f"-crf 28 -c:a aac -b:a 96k {WORK}/motion_{pi}.mp4", shell=True)
    
    # 🔴 关键：ss=0！
    subprocess.run(f"ffmpeg -y -i {WORK}/motion_{pi}.mp4 -ss 0 -t {dur} "
        f"-c:v libx264 -preset ultrafast -crf 28 -g 1 -an "
        f"-fflags +genpts {WORK}/m_{pi}.mp4", shell=True)

# 3. 按原顺序生成播放列表（images[] 原始下标顺序）
concat_lines = []
for cycle in range(CYCLES):
    for ii in range(TOTAL_IMGS):
        # 这张图有对应动图？先播动图（只播第1次）
        if ii in LP_MAP and cycle == 0:
            pi = LP_MAP[ii]
            concat_lines.append(f"file {WORK}/m_{pi}.mp4")
        
        # 播静图
        still_file = f"{WORK}/still_{cycle}_{ii}.mp4"
        subprocess.run(f"ffmpeg -y -loop 1 -i {WORK}/img_{ii}.jpg "
            f"-t {IMG_DUR} -r 30 -vf \"scale=1080:-2\" -c:v libx264 "
            f"-preset ultrafast -crf 28 -g 1 -an -fflags +genpts "
            f"{still_file}", shell=True)
        concat_lines.append(f"file {still_file}")

# 4. 合并
with open(f"{WORK}/list.txt", "w") as f:
    f.write("\n".join(concat_lines))

subprocess.run(f"ffmpeg -y -f concat -safe 0 -i {WORK}/list.txt "
    f"-fflags +genpts -r 30 -c:v libx264 -preset veryfast -crf 28 "
    f"{WORK}/video.mp4", shell=True)

# 5. 加音频
vd = subprocess.run(["ffprobe", "-v", "error", "-show_entries", "format=duration",
    "-of", "csv=p=0", f"{WORK}/video.mp4"], capture_output=True, text=True).stdout.strip()

if os.path.exists(f"{WORK}/bgm.mp3"):
    # ✅ 优先配乐100%
    subprocess.run(f"ffmpeg -y -i {WORK}/video.mp4 -stream_loop -1 "
        f"-i {WORK}/bgm.mp3 -filter_complex "
        f"\"[1:a]volume=1.0,atrim=0:{vd}[bgm]\" "
        f"-map 0:v -map \"[bgm]\" -c:v copy -c:a aac -b:a 96k "
        f"-movflags +faststart {WORK}/final.mp4", shell=True)
else:
    # ⚠️ 无配乐降级
    subprocess.run(f"ffmpeg -y -i {WORK}/video.mp4 -stream_loop 6 "
        f"-i {WORK}/live_0.mp4 -filter_complex "
        f"\"[1:a]volume=1.3,atrim=0:{vd}[orig]\" "
        f"-map 0:v -map \"[orig]\" -c:v copy -c:a aac -b:a 96k "
        f"-movflags +faststart {WORK}/final.mp4", shell=True)

shutil.move(f"{WORK}/final.mp4", "/tmp/synth_result.mp4")
shutil.rmtree(WORK)
```

> 🔴 **[必读] 常见翻车点**
> 1. **`-ss` 不是 `0`** — 换了时间点 → 切换时画面突变 = "闪"。**每次都要检查 `-ss 0`！**
> 2. **没有 `-movflags +faststart`** — 手机播放器可能直接停止播放
> 3. **`-c copy` 直接拼接** — 不行，必须全片重编码 `-c:v libx264`
> 4. **音频用错** — 不是取第1段原声放大！**优先用 `data.music.url` 配乐100%**，没有配乐才降级到第1段原声（volume=1.3）
> 5. **忘记 `-g 1`** — 没有关键帧约束，切换时残留前帧残影
> 6. **段索引不对** — `live_photo[]` 长度可能 2~5 段，不是固定 4，按实际数组长度遍历
> 7. **🔴🔴 只用 `live_photo[].image` 当静图** — `images[]` 可能有9张，`live_photo[].image` 只是子集。**必须把 ALL images[] 全部合进去！**
> 8. **🔴🔴 顺序不对** — 不是"先动图再ALL静图"，是按 `images[]` **原始下标顺序**，每张图匹配到 live_photo 才播动图。参考上面示例。

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
