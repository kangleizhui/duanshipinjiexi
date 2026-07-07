# 套餐页面抓取 — 实时价格数据

AI 实时抓取套餐页面获取最新价格，不在 SKILL.md 中硬编码。

## 抓取命令

```bash
curl -s http://101.32.98.240:8080/pay/index.php
```

## HTML 解析规则

页面返回纯 HTML，套餐卡片结构：

```html
<div class="plan">
  <span class="tag tag-count">按次</span>
  <div class="name">🎯 体验套餐</div>
  <div class="price">¥0.50 <small>/ 共10次</small></div>
  <div class="desc">10次体验，仅需5毛</div>
</div>
```

## 解析参考

```python
import re, html

html_content = "..."  # curl 返回的 HTML

# 提取所有套餐卡片
plans_raw = re.findall(
    r'<div class="plan[^>]*>(.*?)</div>\s*</div>',
    html_content, re.DOTALL
)

plans = []
for card in plans_raw:
    name = re.search(r'class="name"[^>]*>(.*?)</div>', card)
    price = re.search(r'class="price"[^>]*>(.*?)</div>', card)
    desc = re.search(r'class="desc"[^>]*>(.*?)</div>', card)
    tag = re.search(r'class="tag[^"]*"[^>]*>(.*?)</', card)
    popular = re.search(r'badge-popular[^>]*>(.*?)<', card)
    if name:
        plans.append({
            "name": html.unescape(name.group(1)).strip(),
            "price": html.unescape(price.group(1)).strip() if price else "",
            "desc": html.unescape(desc.group(1)).strip() if desc else "",
            "type": html.unescape(tag.group(1)).strip() if tag else "",
            "popular": html.unescape(popular.group(1)).strip() if popular else "",
        })

# 格式化展示给用户
for p in plans:
    popular_badge = f" {p['popular']}" if p.get('popular') else ""
    output = f"{p['name']}{popular_badge}\n  价格: {p['price']}\n  说明: {p['desc']}"
```

## 预期输出（截至 2026-07-07）

| 名称 | 价格 | 说明 |
|------|------|------|
| 🎯 体验套餐 | ¥0.50 / 共10次 | 10次体验，仅需5毛 |
| 按次 50次 | ¥9.90 / 共50次 | 适合偶尔使用 |
| 按次 200次 🔥推荐 | ¥29.90 / 共200次 | 适合高频使用 |
| 按次 1000次 | ¥99.90 / 共1000次 | 适合批量使用 |
| 包月套餐 | ¥19.90 / 30天 | 30天无限次 |
| 包季套餐 | ¥49.90 / 90天 | 90天无限次 |
| 包周套餐 | ¥5.90 / 7天 | 7天无限次 |
| 💎 永久套餐 | ¥199.00 / 永久 | 永久无限次使用 |
| 🌟 终身畅享套餐 | ¥150.00 / 每日限150次 | 终身有效，每日限150次 |
