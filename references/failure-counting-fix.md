# parse.php：只有解析成功才消耗次数

## Bug 发现

`/var/www/short-videos/api/parse.php` 中，密钥使用次数（`used_count`）在检查 API 返回结果**之前**就无条件 +1。这意味着：
- 解析超时 → 服务器返回 500 → 但次数已扣
- 上游 API 故障 → 无有效结果返回 → 但次数已扣
- 网络中断 → curl 返回 false → 但次数已扣

## 修复

**之前：** 日志记录 → 更新使用次数 → 检查结果 → 返回

**之后：** 日志记录（无论成功失败，用于调试） → 检查结果 → 失败立即返回不扣次数 → 成功才计次

```php
// 先记录日志（始终执行）
$stmt = $db->prepare("INSERT INTO usage_log ...");
$stmt->execute();

// 检查解析结果 — 只有成功才消耗次数
if ($http_code != 200 || !$parsed || !isset($parsed['code'])) {
    jsonExit(500, '解析服务异常');  // ← 直接退出，不计次
}

// 到这里才更新使用次数（确保解析成功）
$stmt = $db->prepare("UPDATE api_keys SET used_count = used_count + 1 WHERE id = ?");
$stmt->execute();
```

## 影响

- 测试密钥：失败不消耗每日限免次数
- 正式密钥：失败不消耗剩余次数，用户不会因为接口偶发故障损失额度
- 日志：仍保留所有请求（包括失败）的记录，用于监控和排查

## 相关文件

- `/var/www/short-videos/api/parse.php`（第 174-218 行附近）
- git commit: `d383fb3 fix: 只有解析成功才消耗次数，失败不计次`
