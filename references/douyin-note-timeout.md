# 抖音图文笔记解析超时故障排查

## 现象

调用解析接口处理抖音图文笔记（note/ 或 图文混合内容）时，请求持续 25-60 秒后返回失败或超时。

## 根本原因

抖音图文笔记需要后端多次到抖音服务器拉取图片资源和元数据，耗时显著高于普通视频：
- 普通视频/图片解析：3-10 秒
- 抖音图文笔记：25-60 秒
- 超时不消耗密钥次数（仅在 API 成功返回后计次）

## 修复链

```
AI 调 parse.php
  → 本地 DouyinParser（10s 超时）→ 失败
  → 降级到 BugPK 聚合器（sv1.php）
  → sv1.php 调 api.bugpk.com（需要 25-60s 才能返回）
  → curl 超时 30s → 刚好卡死
  → AI 等了 40s 没返回 → 放弃
```

## 修复

将两处 curl 超时从 30s 扩大到 60s：

```php
// sv1.php - requestUrl()
curl_setopt($ch, CURLOPT_TIMEOUT, 60);

// parse.php - 聚合器降级
CURLOPT_TIMEOUT => 60,  // 第 145、160 行
```

注意本地解析器（DouyinParser）的 10s 超时不动——它执行快慢取决于 Douyin 页面，和图文笔记无关。

## 排查方法

1. 直接调 BugPK API 测耗时：`curl -w "%{time_total}" https://api.bugpk.com/api/douyin?url=...`
2. 检查 parse.php 的聚合器超时配置（30s 为故障阈值）
3. 检查 sv1.php 的 curl 超时
4. 客户端 AI 的 HTTP 超时也要设到 60s 以上
