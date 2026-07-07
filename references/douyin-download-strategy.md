# 抖音视频下载策略

## 问题

- `parse.php` 返回的 `data.url` 和 `data.video_backup[].url` 都在 `zjcdn.com` CDN
- `zjcdn.com` 有 IP 地区限制（境外 403），且自带签名时效
- 直接从 QQ Bot API 传 URL 给 QQ 平台也下不动（同样 geoblocked）

## 解决方案

1. 获取 `video_id`（从 parse 响应的 `data.video_id` 字段或从 `iesdouyin.com` 页面解析）
2. 使用 `snssdk.com` CDN 下载：
   ```
   https://aweme.snssdk.com/aweme/v1/play/
      ?line=0
      &ratio=720p          # 720p h264 最佳平衡
      &video_id=<video_id>
      &watermark=0         # 无水印
   ```
3. 用 `iesdouyin.com` 获取 video_id：
   ```
   https://www.iesdouyin.com/share/video/<aweme_id>/
   ```
   页面 JSON 中包含 `play_addr.url_list[0]` 的 `aweme.snssdk.com` URL

## 请求头

```bash
User-Agent: Mozilla/5.0 (Linux; Android 14) AppleWebKit/537.36
Referer: https://www.douyin.com/
```

## 关键路径

```
parse.php 返回 aweme_id (= extra.aweme_id)
  → 请求 iesdouyin.com/share/video/{aweme_id}/
  → 页面 JSON 提取 play_addr.url_list[0] (snssdk.com)
  → 修改 URL: playwm/ → play/, 加 &watermark=0
  → curl 下载
  → MEDIA 发送给用户
```
