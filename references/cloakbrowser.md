# CloakBrowser 进阶参考

## 从 Playwright 迁移

迁移零成本，只改两行：

```python
# Playwright 旧写法
from playwright.async_api import async_playwright
async with async_playwright() as p:
    browser = await p.chromium.launch(headless=False)

# CloakBrowser 新写法
from cloakbrowser import launch_async
browser = await launch_async(headless=False)
# 其余代码（page.goto / query_selector / fill / click）全部不变
```

---

## API 一览

| 模式 | 函数 | 返回 |
|------|------|------|
| 同步 | `launch()` | `Browser` |
| 异步 | `launch_async()` | `Browser` |
| 持久化 | `launch_persistent_context('./profile')` | `BrowserContext` |

---

## 核心参数

### humanize — 模拟真人行为

```python
# 标准人性化
browser = await launch_async(humanize=True)

# 慢速谨慎模式（高反爬站点）
browser = await launch_async(humanize=True, human_preset="careful")
```

开启后的行为变化：
- 鼠标移动：贝塞尔曲线轨迹
- 键盘输入：逐字符延迟 + 随机停顿
- 滚动：加速 → 匀速 → 减速
- 错误模拟：5% 概率打错字并自动纠正
- 空闲微动作：点击间的微小鼠标晃动

### proxy — 代理

```python
# HTTP 代理
launch(proxy="http://user:pass@proxy:8080")

# SOCKS5 代理
launch(proxy="socks5://user:pass@proxy:1080")

# 配合 GeoIP 自动对齐时区/语言
launch(proxy="http://proxy:8080", geoip=True)
```

### 持久化上下文

```python
ctx = launch_persistent_context("./my-profile", headless=False)
# cookies、localStorage、session 自动保存到 ./my-profile/
# 下次启动恢复，保持登录态
```

---

## 指纹控制

CloakBrowser 支持通过命令行标志精确控制指纹，这些标志在底层 Chromium 的 C++ 层生效：

| 标志 | 作用 |
|------|------|
| `--fingerprint=seed` | 主种子，控制 canvas/WebGL/音频/字体指纹 |
| `--fingerprint-platform` | navigator.platform（windows/macos/linux） |
| `--fingerprint-gpu-vendor` | WebGL 厂商名 |
| `--fingerprint-gpu-renderer` | WebGL 渲染器名 |
| `--fingerprint-hardware-concurrency` | CPU 核心数 |
| `--fingerprint-device-memory` | 设备内存（GB） |
| `--fingerprint-screen-width/height` | 屏幕尺寸 |
| `--fingerprint-locale` | 语言区域 |
| `--fingerprint-timezone` | 时区 |
| `--fingerprint-webrtc-ip` | WebRTC IP（auto 或 指定 IP） |

---

## 检测测试成绩

| 检测服务 | Stock Playwright | CloakBrowser |
|---------|-----------------|-------------|
| reCAPTCHA v3 | 0.1 (bot) | 0.9 (human) |
| Cloudflare Turnstile | FAIL | PASS |
| FingerprintJS | DETECTED | PASS |
| BrowserScan | DETECTED | NORMAL (4/4) |
| `navigator.webdriver` | `true` | `false` |

---

## 注意事项

1. CloakBrowser 防止验证码出现，不解决已出现的验证码
2. 首次运行自动下载 ~200MB 隐形 Chromium 二进制
3. 合理控制请求频率，即使绕过检测也不要打爆目标站
4. 遵守 robots.txt 和服务条款
