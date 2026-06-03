---
name: general_spider
version: 1.0.0
description: |
  全阶段 Python 爬虫助手。覆盖 HTTP 请求 → 数据提取 → 字体反爬 → JS 逆向 
  → Selenium → DrissionPage → Playwright → CloakBrowser 全链路。
  根据目标网站自动选工具栈、生成代码、排查报错。
triggers:
  - 爬虫
  - 爬取
  - 抓取
  - 反爬
  - 数据采集
  - web scraping
  - spider
  - crawler
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - WebSearch
  - WebFetch
---

# general_spider — 全阶段 Python 爬虫助手

## 工作流

当用户提出爬虫需求时，按以下步骤执行：

### 第1步：收集信息

缺什么问什么，每次只问一个问题：

1. 目标 URL 是什么？
2. 需要提取哪些数据字段？
3. 大概要爬多少页/条？
4. 需要登录吗？（需要 → 提供 cookie 或账号）

### 第2步：浏览器侦查

用 headless 模式快速访问目标页面一次，判断：

- 数据是直接 HTML 返回，还是 JS 动态渲染？
- 有没有 Cloudflare / 5 秒盾 / 验证码？
- 能不能直接抓到 API 接口？

### 第3步：选工具栈

按反爬强度递进选择：

| 场景 | 推荐工具 | 说明 |
|------|---------|------|
| 纯接口 / 静态 HTML，无反爬 | `requests` + `lxml` | 最快，零开销 |
| 纯接口，有 TLS 指纹检测 | `curl_cffi` | impersonate 一行搞定 |
| 需要 JS 渲染，无反爬 | `Playwright` (sync) | 简单页面够用 |
| 需要 JS 渲染，有中等反爬 | `Playwright` (async) | 多页并发效率高 |
| 国内站点，中等反爬 | `DrissionPage` | 中文文档好，API 简洁 |
| 强反爬（Cloudflare/DataDome） | `CloakBrowser` | C++ 源码级指纹伪装 |
| 大规模整站采集 | `Scrapy` / `Crawlee` | 内置队列、去重、反封锁 |
| 字体反爬 | `fontTools` + `ddddocr` | cmap 解密 + OCR |
| 滑块验证码 | `ddddocr.slide_match` + 动作链 | ddddocr 识别缺口位置 |
| JS 参数加密 | `execjs` / `page.evaluate` | Python 调 JS 加密逻辑 |

### 第4步：选解析方式

| 数据类型 | 工具 | 示例 |
|---------|------|------|
| HTML 表格/列表 | `lxml` XPath | `//ul[@class="list"]/li` |
| HTML 单个值 | CSS Selector | `#price` 或 `.title` |
| JSON 接口 | `jsonpath` | `$.data.list[*].name` |
| JS 内嵌数据 | `re` regex | `r'var data = (.*?);'` 非贪婪 |
| 字体加密文本 | `fontTools` cmap | 建立密文→真实字映射字典 |

### 第5步：选导出方式

| 格式 | 适用场景 | 关键配置 |
|------|---------|---------|
| CSV | Excel 查看 | `encoding='utf-8-sig'` 防乱码 |
| JSON | API 交互 | `ensure_ascii=False` |
| TXT | 日志/文本 | `encoding='utf-8'` |

### 第6步：生成代码

遵循以下规则（详见下文「代码生成规则」），生成后自动运行验证。

### 第7步：运行验证 → 报错 → 查调试清单 → 修复

运行生成的代码。如果报错，对照「调试清单」排查修复，最多 3 次迭代。3 次后仍未解决则上报用户。

### 第8步：确认结果

数据保存后告知用户：多少条数据、保存在哪、用时多少。

---

## 知识库速查表

### 阶段 1：HTTP 请求基础 (day01-day03)

| 概念 | 工具 | 代码骨架 |
|------|------|---------|
| GET 请求 | `requests.get()` | `requests.get(url, headers=headers, timeout=10)` |
| POST 请求 | `requests.post()` | `requests.post(url, data=data, headers=headers)` |
| UA 伪装 | `fake_useragent` | `UserAgent(platforms='desktop').random` |
| Cookie 持久化 | `requests.Session()` | `session.get()` / `session.post()` |
| URL 编码 | `urllib.parse.quote()` | `quote('中文关键词')` |

### 阶段 2：数据提取 (day06-day08)

| 数据类型 | 工具 | 代码骨架 |
|---------|------|---------|
| HTML XPath | `lxml.etree` | `tree.xpath('//ul/li/a/text()')` |
| HTML CSS | `lxml.cssselect` | `tree.cssselect('.price')` |
| JSON 路径 | `jsonpath` | `jsonpath(obj, '$.data[*].name')` |
| JS 内嵌数据 | `re` | `re.findall(r'var data = (.*?);', html)` |

### 阶段 3：字体反爬 (day09-day10)

```python
from fontTools.ttLib import TTFont
font = TTFont('font.woff')          # 下载字体文件
cmap = font.getBestCmap()           # 获取 cmap 映射表
mapping = {chr(k): str(v) for k, v in cmap.items()}  # unicode → 真实字
# 将页面中加密的字符按 mapping 替换
```

### 阶段 4：JS 逆向 (day11-day12)

```python
import execjs
ctx = execjs.compile(open('encrypt.js', encoding='utf-8').read())
encrypted = ctx.call('encrypt_function', '参数')
```

### 阶段 5-6：浏览器自动化 (day13-day16)

| 操作 | Selenium | DrissionPage |
|------|----------|--------------|
| 启动 | `webdriver.Chrome()` | `ChromiumPage()` |
| 无头模式 | `Options().add_argument('--headless')` | `ChromiumOptions().headless()` |
| 元素定位 | `find_element(By.XPATH, '//div')` | `ele('xpath://div')` |
| 点击 | `element.click()` | `ele.click()` |
| 输入 | `element.send_keys('text')` | `ele.input('text')` |
| iframe | `driver.switch_to.frame(el)` | 直接 `ele('tag:iframe')` |
| 网络监听 | 不支持 | `page.listen.start()` |
| 滑块验证码 | `ActionChains(driver).click_and_hold().move_by_offset().release()` | `page.actions.hold().move().release()` |

### 阶段 7：异步并发 (day_19-day_20)

```python
import asyncio
from aiohttp import ClientSession

async def fetch(session, semaphore, url):
    async with semaphore:
        async with session.get(url) as resp:
            return await resp.text()

async def main(urls):
    semaphore = asyncio.Semaphore(10)
    async with ClientSession() as session:
        tasks = [asyncio.create_task(fetch(session, semaphore, url)) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    return [r for r in results if not isinstance(r, Exception)]
```

### 阶段 8：Playwright / CloakBrowser (day_21)

| | Playwright | CloakBrowser |
|---|-----------|-------------|
| 同步启动 | `sync_playwright().chromium.launch()` | `launch()` |
| 异步启动 | `async_playwright().chromium.launch()` | `launch_async()` |
| 反检测 | 手动 `page.add_init_script()` | 源码级内置 |
| 迁移成本 | — | 改一行 import |

---

## 代码生成规则

生成代码时必须遵守：

1. **匹配用户风格**：观察用户项目中的变量命名（如 `page_book_info`、`book_name_element`）、注释风格（中文注释说明意图）和缩进规范，保持一致
2. **Headers 完整**：必须包含 `User-Agent`、`Accept`、`Accept-Language`、`sec-ch-ua`、`sec-fetch-*` 系列字段
3. **CSV 编码**：统一使用 `encoding='utf-8-sig'`，确保 Excel 直接打开不乱码
4. **异步安全**：并发代码必须带 `asyncio.Semaphore` 限流，`gather` 必须加 `return_exceptions=True`
5. **浏览器复用**：多页并发时共享一个 browser 实例，用完 `try/finally` 关闭 page
6. **CloakBrowser 无需反检测 JS**：指纹伪装已内置在 C++ 层，不要添加 `add_init_script`
7. **使用 .venv 解释器**：运行爬虫代码时使用项目虚拟环境 Python 解释器

---

## 调试清单

遇到报错按此表排查，按频率排序：

| # | 典型报错 | 原因 | 修复 |
|---|---------|------|------|
| 1 | `ModuleNotFoundError: No module named 'xxx'` | 未安装 | `pip install xxx` |
| 2 | `AttributeError: 'NoneType' object has no attribute 'xxx'` | XPath/CSS 写错，或页面结构变了 | F12 重新确认 selector，检查是否在 iframe 中 |
| 3 | `Element is not an <input>, <textarea>, <select>` | `.fill()` 选中了 div 容器而非内部 input | 用 `#chat-textarea` 或 `//input[@id='kw']` 精确定位到表单元素 |
| 4 | 列表为空 / URL 404 | 参数名与 Playwright `page` 对象同名导致 URL 拼错 | 将页码参数名改为 `page_num`，避免覆盖 |
| 5 | `SSL: CERTIFICATE_VERIFY_FAILED` | Windows 下 Python 证书链不完整 | `pip install pip-system-certs` |
| 6 | 三维列表 / CSV 写不进去 | `asyncio.gather()` 返回结果多包了一层 | `[item for page in all_pages for item in page]` 扁平化 |
| 7 | CSV 用 Excel 打开中文乱码 | 默认写入 gbk 编码 | 加 `encoding='utf-8-sig'` |
| 8 | 一个 task 报错导致全部数据丢失 | `gather()` 默认抛异常 | 加 `return_exceptions=True`，然后用 `isinstance(result, Exception)` 过滤 |

