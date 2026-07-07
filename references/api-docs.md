# Base URL = `{SCH}://{H1}.{H2}.{H3}.{H4}:{AP}`

## Base URL

```
{SCH}://{H1}.{H2}.{H3}.{H4}:{AP}
```

## 测试密钥

密钥格式以实际交付为准，测试密钥值见 SKILL.md variables：`{TK1}-{TK2}-{TK3}-{TK4}-{TK5}`

## 1. 解析视频

```
GET {SCH}://{H1}.{H2}.{H3}.{H4}:{AP}/api/parse.php?key={密钥}&url={视频链接}
```

### 响应 JSON

```json
{
  "code": 200,
  "data": {
    "type": "video",          // video / image / live
    "title": "标题",
    "author": {"name": "作者名"},
    "url": "https://...",
    "video_id": "v0d00fg...",
    "video_backup": [{"url": "..."}],
    "images": ["https://..."],
    "synthesized_url": null,          // ⚡ live时=null，需轮询拿
    "synthesize_job_id": "xxx",       // ⚡ 异步合成任务ID
    "synthesize_status": "processing",// ⚡ processing / downloading / synthesizing
    "synthesized_duration": 18.13,
    "stats": {"liked": "12.3w", "comment": "4567", "share": "890", "collect": "123"}
  },
  "key_info": {
    "is_test": true,
    "remaining": 8
  }
}
```

### 关键字段说明

| 字段 | 说明 |
|------|------|
| data.type | video=视频, image=图片, live=动图 |
| data.url | 视频直链（抖音可能403） |
| data.live_photo[] | 动图素材数组（每个元素有 video + image），用于本地 ffmpeg 合成 |
| data.video_id | 用于构造抖音 CDN 链接 |
| data.synthesized_url | live时=null（服务端异步合成时用） |
| data.synthesize_job_id | 异步合成任务ID，传给 check_job.php |
| data.synthesize_status | processing/downloading/synthesizing |
| key_info.remaining | 计次密钥剩余次数 |

### 处理规则

| type | 操作 |
|------|------|
| video | 下载后用 `send_message` 发 `MEDIA:/tmp/xxx.mp4` |
| image | 逐张下载后挨个用 `send_message` 发 `MEDIA:/tmp/xxx.jpg` |
| live | **优先本地 ffmpeg 合成**（用 `data.live_photo[]` 所有段），本地无 ffmpeg 时降级服务端 check_job.php 轮询 |

### 错误码

| HTTP | msg | 说明 |
|------|-----|------|
| 400 | 缺少参数: key | 未传密钥 |
| 403 | 密钥无效 | 密钥不存在 |
| 403 | 密钥已过期 | 已过期 |
| 403 | 密钥次数已用完 | 计次耗尽 |
| 429 | 今日调用次数已达上限 | 测试密钥每IP每天10次 |
| 500 | 解析服务异常 | 内部错误 |

### 动图（live）异步合成

type=live 时，parse.php 返回 synthesized_url=null + synthesize_job_id。服务端将所有 live_photo 段合成为完整视频（不再仅用第1段）。

```
GET {SCH}://{H1}.{H2}.{H3}.{H4}:{AP}/api/check_job.php?key={密钥}&job_id={synthesize_job_id}
```

**轮询响应：**

| code | msg | 操作 |
|------|-----|------|
| 202 | processing | 仍处理中，3-5秒后重试 |
| 200 | 合成成功 | data.url 即合成视频链接，下载后发 MEDIA |
| 500 | 合成失败 | 放弃轮询，发原视频 |
| 404 | 任务不存在 | 已过期（10分钟），放弃 |

**轮询建议：** 每3-5秒一次，多段素材（如4个live_photo）约需60-90秒。超过90秒仍返回202 → 发原视频，不等了。

---

## 2. 下单

```
POST {SCH}://{H1}.{H2}.{H3}.{H4}:{AP}/pay/create.php
Content-Type: application/json

{
  "id": "count_10",
  "name": "体验套餐",
  "type": "count",
  "count": 10,
  "days": 0,
  "price": 0.50,
  "daily_limit": 0
}
```

**成功响应：**
```json
{
  "success": true,
  "order_id": "SV20260707XXXX",
  "qr_code": "https://qr.alipay.com/xxx",
  "amount": 0.50
}
```

> ⚠️ **关键：必须生成二维码图片，禁止直接发支付宝链接！**
> 收到 `qr_code` URL 后，用 `qrencode` 命令转换为图片，再用 `send_message` 发 `MEDIA:/tmp/qr.png`：
> ```bash
> qrencode -o /tmp/qr.png "https://qr.alipay.com/xxx"
> ```
> 若系统无 qrencode，先安装：`apt install -y qrencode`
> ⚠️ 只能用 `send_message` 工具发文件，禁止再调 QQ Bot HTTP API。

## 3. 查支付状态

```
GET {SCH}://{H1}.{H2}.{H3}.{H4}:{AP}/pay/query.php?order_id=SV20260707XXXX
```

**已支付：** status=paid, api_key 有值
**未支付：** status=pending, api_key=null

## 4. 套餐数据

```json
[
  {"id": "count_10", "name": "体验套餐", "type": "count", "count": 10, "price": 0.50},
  {"id": "count_50", "name": "按次 50次", "type": "count", "count": 50, "price": 9.90},
  {"id": "count_200", "name": "按次 200次", "type": "count", "count": 200, "price": 29.90, "popular": true},
  {"id": "count_1000", "name": "按次 1000次", "type": "count", "count": 1000, "price": 99.90},
  {"id": "monthly", "name": "包月套餐", "type": "monthly", "count": 0, "days": 30, "price": 19.90},
  {"id": "quarterly", "name": "包季套餐", "type": "monthly", "count": 0, "days": 90, "price": 49.90},
  {"id": "daily_7", "name": "包周套餐", "type": "daily", "count": 0, "days": 7, "price": 5.90},
  {"id": "lifetime", "name": "永久套餐", "type": "lifetime", "count": 0, "days": 0, "price": 199.00},
  {"id": "lifetime_daily", "name": "终身畅享套餐", "type": "lifetime_daily", "count": 0, "days": 0, "price": 150.00, "daily_limit": 150}
]
```
