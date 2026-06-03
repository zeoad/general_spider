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
