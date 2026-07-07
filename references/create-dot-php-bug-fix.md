# create.php Bug Fix — daily_limit SQLite Insert

## 问题
`/var/www/short-videos/pay/create.php` 在创建订单时返回 500 错误：
```
PHP Warning: SQLite3::prepare(): Unable to prepare statement: 9 values for 10 columns
PHP Fatal error: Call to a member function bindValue() on false
```

## 原因
INSERT 语句定义了 10 列（含 `daily_limit`）和 10 个占位符 `?`，但只调用了 9 次 `bindValue()`。第 8 个 `bindValue` 本应是 `daily_limit`，却绑成了 `note`（"{$plan_name} 订单"），导致 `daily_limit` 未绑定，SQLite 报 9 values for 10 columns。

## 修复
将硬编码在 SQL 里的 `'pending'` 和不完整的 bindings 改为完整的 10 个 bindValue 调用：

```php
$stmt = $db->prepare("INSERT INTO orders (order_id, plan_type, plan_name, amount, total_count, expires_days, key_type, daily_limit, note, status) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)");
$stmt->bindValue(1, $out_trade_no, SQLITE3_TEXT);
$stmt->bindValue(2, $plan_id, SQLITE3_TEXT);
$stmt->bindValue(3, $plan_name, SQLITE3_TEXT);
$stmt->bindValue(4, $price, SQLITE3_FLOAT);
$stmt->bindValue(5, $count, SQLITE3_INTEGER);
$stmt->bindValue(6, $days, SQLITE3_INTEGER);
$stmt->bindValue(7, $plan_type, SQLITE3_TEXT);
$stmt->bindValue(8, $daily_limit, SQLITE3_INTEGER);
$stmt->bindValue(9, "{$plan_name} 订单", SQLITE3_TEXT);
$stmt->bindValue(10, 'pending', SQLITE3_TEXT);
$stmt->execute();
```

## 验证
POST 携带 `daily_limit` 字段（即使为 0）即可正常创建订单：
```bash
curl -s -X POST http://101.32.98.240:8080/pay/create.php \
  -H 'Content-Type: application/json' \
  -d '{"id":"count_10","name":"🎯 体验套餐","type":"count","count":10,"days":0,"daily_limit":0,"price":0.5}'
```
