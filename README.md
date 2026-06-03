# general_spider

全阶段 Python 爬虫助手 — Claude Code Skill。覆盖从 HTTP 请求到 CloakBrowser 反检测的完整爬虫学习链路。

## 功能

- **向导式代码生成**：描述目标网站，自动选择工具栈并生成可运行代码
- **知识库速查**：10 个学习阶段的核心概念和代码模板
- **调试清单**：8 种高频报错的根因分析和修复方案
- **工具进阶参考**：Playwright、CloakBrowser、DrissionPage、Scrapy、异步并发、反检测综述、APP 抓包

## 安装

复制到 Claude Code skills 目录：

```bash
mkdir -p ~/.claude/skills/general_spider
cp -r * ~/.claude/skills/general_spider/
```

重启 Claude Code，Skill 自动加载。

## 使用示例

### 场景 1：简单抓取

```
帮我爬当当图书畅销榜前 3 页，导出 CSV
```

Skill 会：侦查页面 → 选 Playwright → 生成异步并发代码 → 运行 → 保存 dangdang_books.csv。

### 场景 2：反爬咨询

```
这个网站有 Cloudflare 5 秒盾，用什么方案？
```

Skill 会：读取 `references/anti-detection.md` → 推荐 CloakBrowser → 给出具体参数配置。

### 场景 3：代码改造

```
把这段 requests 代码改成异步 aiohttp
```

Skill 会：读取 `references/async-patterns.md` → 按用户代码风格改写 → 加上 Semaphore 和错误处理。

### 场景 4：APP 抓包

```
帮我抓这个 APP 的数据接口
```

Skill 会：读取 `references/mobile-scraping.md` → 指导你 MuMu 模拟器 + Reqable 抓包 → 分析 HTTPS 请求 → 用 Python 复现。

## 前置要求

- Python 3.9+
- 虚拟环境（推荐 `.venv`）
- 按需安装：`playwright`、`cloakbrowser`、`DrissionPage`、`curl_cffi`、`aiohttp`、`aiofiles` 等

## 文件结构

```
general_spider/
├── SKILL.md                    # 主文件：决策树 + 知识库 + 调试清单
├── references/
│   ├── playwright.md           # Playwright 进阶
│   ├── cloakbrowser.md         # CloakBrowser 进阶
│   ├── drissionpage.md         # DrissionPage 进阶
│   ├── scrapy.md               # Scrapy 大规模采集
│   ├── async-patterns.md       # 异步并发模式
│   ├── anti-detection.md       # 反检测手段综述
│   └── mobile-scraping.md      # APP 抓包爬取（MuMu + Reqable）
├── README.md
└── LICENSE
```

## 许可

MIT License — 完全开源，可自由使用、修改、分发。
