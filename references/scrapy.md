# Scrapy 进阶参考

## 定位

Scrapy 是 Python 最成熟的爬虫框架，适合大规模整站采集（万级到百万级页面）。

跟 `requests` + `lxml` 组合的区别：Scrapy 内置请求队列、去重、中间件管道、并发控制和错误重试，不需要手写调度逻辑。

---

## 安装和创建项目

```bash
pip install scrapy
scrapy startproject myproject
cd myproject
scrapy genspider example example.com
```

生成的目录结构：

```
myproject/
├── scrapy.cfg
└── myproject/
    ├── spiders/
    │   └── example.py     # 爬虫逻辑
    ├── items.py           # 数据模型
    ├── middlewares.py     # 中间件（UA、代理、重试）
    ├── pipelines.py       # 数据管道（保存、清洗）
    └── settings.py        # 全局配置
```

---

## 第一个爬虫

```python
import scrapy

class BookSpider(scrapy.Spider):
    name = "books"
    start_urls = ["http://books.toscrape.com/"]

    def parse(self, response):
        for book in response.css("article.product_pod"):
            yield {
                "title": book.css("h3 a::attr(title)").get(),
                "price": book.css(".price_color::text").get(),
                "link": book.css("h3 a::attr(href)").get(),
            }
        # 翻页
        next_page = response.css(".next a::attr(href)").get()
        if next_page:
            yield response.follow(next_page, self.parse)
```

运行：

```bash
scrapy crawl books -o books.csv   # 导出 CSV
scrapy crawl books -o books.json  # 导出 JSON
```

---

## 核心概念

| 组件 | 作用 | 说明 |
|------|------|------|
| Spider | 定义爬取逻辑 | parse 方法解析响应，yield 数据或新请求 |
| Item | 数据容器 | 定义字段结构，比字典更规范 |
| Pipeline | 数据处理 | 清洗、去重、存数据库 |
| Middleware | 请求/响应拦截 | UA 轮换、代理切换、重试 |
| Settings | 全局配置 | 并发数、延迟、下载超时 |

---

## 关键配置（settings.py）

```python
# 并发请求数（默认 16，适当调低避免被封）
CONCURRENT_REQUESTS = 16

# 下载延迟（秒）
DOWNLOAD_DELAY = 1

# 自动限速（推荐开启，替代固定延迟）
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 1
AUTOTHROTTLE_MAX_DELAY = 10

# 是否遵循 robots.txt
ROBOTSTXT_OBEY = True

# 默认请求头
DEFAULT_REQUEST_HEADERS = {
    "User-Agent": "Mozilla/5.0 ...",
    "Accept": "text/html,application/xhtml+xml",
    "Accept-Language": "zh-CN,zh;q=0.9",
}

# 中间件
DOWNLOADER_MIDDLEWARES = {
    "myproject.middlewares.MyUserAgentMiddleware": 543,
    "myproject.middlewares.MyProxyMiddleware": 544,
}

# 数据管道
ITEM_PIPELINES = {
    "myproject.pipelines.MyPipeline": 300,
}
```

---

## 常见中间件写法

### UA 轮换

```python
# middlewares.py
from fake_useragent import UserAgent

class MyUserAgentMiddleware:
    def __init__(self):
        self.ua = UserAgent(platforms="desktop")

    def process_request(self, request, spider):
        request.headers["User-Agent"] = self.ua.random
```

### 代理轮换

```python
class MyProxyMiddleware:
    def __init__(self):
        self.proxies = ["http://ip1:port", "http://ip2:port"]
        self.index = 0

    def process_request(self, request, spider):
        request.meta["proxy"] = self.proxies[self.index % len(self.proxies)]
        self.index += 1
```

---

## 数据管道写法

```python
# pipelines.py
import csv
from scrapy.exceptions import DropItem

class CsvPipeline:
    def open_spider(self, spider):
        self.file = open("output.csv", "w", newline="", encoding="utf-8-sig")
        self.writer = csv.writer(self.file)
        self.writer.writerow(["title", "price"])

    def process_item(self, item, spider):
        # 过滤空数据
        if not item.get("title"):
            raise DropItem("Missing title")
        self.writer.writerow([item["title"], item["price"]])
        return item

    def close_spider(self, spider):
        self.file.close()
```

---

## Scrapy 异步引擎

Scrapy 基于 Twisted 异步引擎，天然支持高并发，不需要手写 `asyncio`：

```python
# 多个请求并发执行，Scrapy 自动管理
def parse_list(self, response):
    detail_urls = response.css(".item a::attr(href)").getall()
    for url in detail_urls:
        yield response.follow(url, self.parse_detail)
    # 所有 detail 页会自动并发请求
```

---

## 什么时候用 Scrapy

| 场景 | 用 Scrapy | 用 Playwright |
|------|----------|--------------|
| 万级页面采集 | 首选 | 太重 |
| 需要 JS 渲染 | 配合 scrapy-playwright 插件 | 原生支持 |
| 纯数据 API | 首选 | 不需要浏览器 |
| 整站爬取 | 内置去重+队列 | 需手写调度 |
| 增量采集 | 需自己实现 | 需自己实现 |

---

## Scrapy + Playwright 混合方案

```python
# 安装
pip install scrapy-playwright

# settings.py
DOWNLOAD_HANDLERS = {
    "http": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
    "https": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
}
TWISTED_REACTOR = "twisted.internet.asyncioreactor.AsyncioSelectorReactor"

# spider
class JSSpider(scrapy.Spider):
    name = "js_spider"

    def start_requests(self):
        yield scrapy.Request(
            url="https://example.com",
            meta={"playwright": True},  # 这个请求用浏览器
        )

    async def parse(self, response):
        # 正常解析即可，Scrapy 已处理 JS 渲染
        for item in response.css(".item"):
            yield {"title": item.css("h2::text").get()}
```
