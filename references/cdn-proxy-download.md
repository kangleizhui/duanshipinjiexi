# CDN 视频代理下载 (svproxyurl.php) & 抖音地区限制

## svproxyurl.php 代理

项目内置了 `api/svproxyurl.php` 代理，用于解决部分 CDN 的 403 限制。

### API

```
GET /api/svproxyurl.php?proxyurl=<base64_encoded_url>&type=douyin
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `proxyurl` | 是 | 视频直链的 **base64 编码**（非明文） |
| `type` | 否 | `douyin`（默认）或 `weibo` |

### 使用方式

```python
import base64
video_url = "https://v3-dy-o.zjcdn.com/..."
encoded = base64.b64encode(video_url.encode()).decode()
proxy_url = f"http://101.32.98.240:8080/api/svproxyurl.php?proxyurl={encoded}&type=douyin"
```

## 抖音 CDN 地区限制（核心坑点）

这是最关键的已知问题。

### 现象

视频解析成功（返回完整数据），但 `data.url` 和 `data.video_backup[*].url` 中所有抖音 CDN 链接（`v3-dy-o.zjcdn.com`、`v6-dy-o.zjcdn.com` 等）在下载时都返回 **403 Forbidden**。

### 根因

抖音 (zjcdn.com) 的 CDN 架构：
1. CDN 由 **Tengine** (阿里巴巴 Web 服务器) 驱动
2. 实施**严格的地域/IP 白名单**——仅限中国大陆 IP 访问
3. 请求头中的 `Referer`、`User-Agent` 也会被检查
4. 服务器返回 `403 Forbidden` + 空 body 或 HTML 错误页，但 Content-Type 设为 `video/mp4`（抗解析技巧）

### 已验证不奏效的方案

| 方案 | 结果 |
|------|------|
| 本机 curl + 模拟 Referer/UA | ❌ 403 |
| PHP curl 代理 (svproxyurl.php) | ❌ 403 |
| QQ Bot API URL 上传（QQ 国内服务器转发） | ❌ "富媒体文件下载失败" 850026 |
| 全部 12 个 video_backup 画质（h264/h265） | ❌ 全部 403 |
| 使用不同的 CDN 域名 | ❌ 同为 zjcdn.com |

### 回退方案（唯一有效）

直接提供视频链接让用户自己点开看。用户在中国大陆，手机 QQ 中点开即可播放。

```python
# 解析成功后将最佳画质链接发给用户
best_url = parsed_data["data"]["video_backup"][0]["url"]
quality = parsed_data["data"]["video_backup"][0]["quality"]
# 发消息时直接把 URL 贴出来，用户可点击播放
```

**注意**：不要只发文字不附视频链接。用户要的是视频，不是文字描述。

### QQ Bot API 令牌获取

当需要直接调用 QQ Bot API 上传媒体时：

```python
import httpx
app_id = "1904088040"
client_secret = "p7Qj3OrLpKqMtRzY7hItV8lP3iO4lSAs"

resp = httpx.post(
    "https://bots.qq.com/app/getAppAccessToken",
    json={"appId": app_id, "clientSecret": client_secret},
    timeout=30,
)
token = resp.json()["access_token"]
```

> ⚠️ `curl` 直接调用此端点会返回 `"internal err"` 100002，必须用 Python `httpx` 或 `requests` 库。

上传视频文件（本地文件或 URL）：

```python
upload = httpx.post(
    f"https://api.sgroup.qq.com/v2/users/{user_id}/files",
    headers={"Authorization": f"QQBot {token}"},
    json={"file_type": 2, "url": video_url, "srv_send_msg": False},
)
file_info = upload.json().get("file_info")
# 然后发送媒体消息
msg = httpx.post(
    f"https://api.sgroup.qq.com/v2/users/{user_id}/messages",
    headers={"Authorization": f"QQBot {token}"},
    json={"msg_type": 7, "media": {"file_info": file_info}, "msg_seq": 1},
)
```
