# Playwright 进阶参考

## 同步 vs 异步

| | sync_api | async_api |
|---|---|---|
| 适用 | 简单脚本、单页操作 | 多页并发、高性能 |
| 写法 | `with sync_playwright() as p:` | `async with async_playwright() as p:` |
| 函数 | `def` | `async def` + `await` |

选择原则：爬 1-2 页用 sync，3 页以上用 async。

---

## 共享 Browser 模式（异步多页并发标准模板）

每页开一个新 browser 是最大性能杀手。正确做法是所有 page 共享一个 browser：

```python
import asyncio
from playwright.async_api import async_playwright

async def scrape_page(page, page_num):
    await page.goto(f'https://example.com/page/{page_num}')
    items = await page.query_selector_all('.item')
    results = []
    for item in items:
        title = await item.query_selector('.title')
        results.append(await title.inner_text())
    return results

async def main(pages):
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        semaphore = asyncio.Semaphore(5)  # 最多同时 5 个 page

        async def crawl(num):
            async with semaphore:
                page = await browser.new_page()
                try:
                    return await scrape_page(page, num)
                finally:
                    await page.close()

        tasks = [asyncio.create_task(crawl(i)) for i in range(1, pages + 1)]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        await browser.close()

    # 扁平化 + 过滤异常
    data = []
    for r in results:
        if isinstance(r, Exception):
            print(f"警告：{r}")
        else:
            data.extend(r)
    return data

asyncio.run(main(5))
```

---

## 鼠标轨迹模拟（滑块验证码）

```python
# 找到滑块元素
slider = page.query_selector('.slider')
box = await slider.bounding_box()

# 计算目标位置（通常需要先识别缺口位置）
target_x = box['x'] + 209  # 缺口偏移量
target_y = box['y'] + box['height'] / 2

# 模拟人工拖动（分段移动 + 微调）
await page.mouse.move(box['x'] + 5, target_y)
await page.mouse.down()
# 分段拖动模拟人类加速度
steps = [
    (target_x - box['x']) * 0.6,  # 快速段
    (target_x - box['x']) * 0.3,  # 减速段
    (target_x - box['x']) * 0.1,  # 微调段
]
for step in steps:
    await page.mouse.move(box['x'] + step, target_y + 2)
    await page.wait_for_timeout(100)
await page.mouse.up()
```

---

## CDP 网络监听

```python
# 拦截所有图片请求
await page.route('**/*.{png,jpg,jpeg}', lambda route: route.abort())

# 监听 API 响应
async def log_response(response):
    if 'api' in response.url:
        print(f'{response.url} → {response.status}')

page.on('response', log_response)
```

---

## 选择器最佳实践

| 优先级 | 方式 | 示例 | 适用场景 |
|--------|------|------|---------|
| 1 | CSS Selector | `#kw`、`.title`、`input[name="wd"]` | 大多数情况 |
| 2 | text= | `page.click('text=登录')` | 文字确定的按钮 |
| 3 | XPath | `//ul[@class="list"]/li[1]` | 复杂层级关系 |
| 4 | get_by_role | `page.get_by_role('button', name='提交')` | 无障碍友好的页面 |
