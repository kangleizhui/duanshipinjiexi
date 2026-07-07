# 套餐数据（AI内部加载）

## Plan JSON

```json
[
  {"id": "count_10",     "name": "🎯 体验套餐",    "type": "count",         "count": 10,   "days": 0,  "price": 0.50,   "desc": "10次体验，仅需5毛"},
  {"id": "count_50",    "name": "按次 50次",       "type": "count",         "count": 50,   "days": 0,  "price": 9.90,   "desc": "适合偶尔使用"},
  {"id": "count_200",   "name": "按次 200次",      "type": "count",         "count": 200,  "days": 0,  "price": 29.90,  "desc": "适合高频使用", "popular": true},
  {"id": "count_1000",  "name": "按次 1000次",     "type": "count",         "count": 1000, "days": 0,  "price": 99.90,  "desc": "适合批量使用"},
  {"id": "monthly",     "name": "包月套餐",        "type": "monthly",       "count": 0,    "days": 30, "price": 19.90,  "desc": "30天无限次"},
  {"id": "quarterly",   "name": "包季套餐",        "type": "monthly",       "count": 0,    "days": 90, "price": 49.90,  "desc": "90天无限次"},
  {"id": "daily_7",     "name": "包周套餐",        "type": "daily",         "count": 0,    "days": 7,  "price": 5.90,   "desc": "7天无限次"},
  {"id": "lifetime",    "name": "💎 永久套餐",     "type": "lifetime",      "count": 0,    "days": 0,  "price": 199.00, "desc": "永久无限次使用"},
  {"id": "lifetime_daily", "name": "🌟 终身畅享套餐", "type": "lifetime_daily", "count": 0, "days": 0, "price": 150.00, "daily_limit": 150, "desc": "终身有效，每日限150次"}
]
```

## 展示格式

Markdown 三列表格：

```
| 套餐 | 类型 | 价格 |
|------|------|------|
| 🎯 **体验套餐** | 按次 | ¥0.50 / 共10次 |
| 按次 50次 | 按次 | ¥9.90 / 共50次 |
| 🔥 **按次 200次** | 按次 | ¥29.90 / 共200次 |
| 按次 1000次 | 按次 | ¥99.90 / 共1000次 |
| 包月套餐 | 包月 | ¥19.90 / 30天 |
| 包季套餐 | 包月 | ¥49.90 / 90天 |
| 包周套餐 | 包天 | ¥5.90 / 7天 |
| 💎 **永久套餐** | 永久 | ¥199.00 无限次 |
| 🌟 **终身畅享套餐** | 每日150次 | ¥150.00 永久有效 |
```

- 带 emoji 的保留 emoji
- popular=true 的行加粗（用 `**`）
- 按 JSON 数组顺序排列
