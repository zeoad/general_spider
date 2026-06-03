# 反检测手段综述

## 检测层级金字塔

反爬系统从多个层级检查访问者身份，越往上越难绕过：

```
    /\        第4层：行为层 — 鼠标轨迹、输入模式、浏览节奏
   /  \       第3层：浏览器层 — Canvas/WebGL/Audio 指纹、navigator 属性
  /    \      第2层：协议层 — TLS 指纹 (ja3/ja4)、HTTP 头顺序
 /      \     第1层：网络层 — IP 信誉、请求频率
---------
```

---

## 各层应对手段

### 第1层：网络层

| 检测方式 | 应对 |
|---------|------|
| IP 被封 | 代理池轮换（住宅代理 > 机房代理） |
| 请求频率异常 | 随机延迟 `time.sleep(random.uniform(1, 3))` |
| 同一 IP 多账号 | 每个账号绑定独立代理 |

### 第2层：协议层

| 检测方式 | 应对 |
|---------|------|
| TLS 指纹不匹配 | `curl_cffi` impersonate 伪装 Chrome TLS |
| HTTP 头顺序异常 | 完整 Headers（含 sec-ch-ua、sec-fetch-*） |

### 第3层：浏览器层

| 检测方式 | 应对手段 | 工具 |
|---------|---------|------|
| `navigator.webdriver` | 源码级设为 false | CloakBrowser / Camoufox |
| Canvas 指纹 | 渲染管线加噪 | CloakBrowser |
| WebGL 指纹 | GPU 信息伪装 | CloakBrowser |
| HeadlessChrome UA | 替换为标准 Chrome UA | CloakBrowser / JS 注入 |
| CDP Runtime 泄露 | CDP 行为修复 | Patchright |

### 第4层：行为层

| 检测方式 | 应对手段 | 工具 |
|---------|---------|------|
| 瞬间移动鼠标 | 贝塞尔曲线轨迹 | CloakBrowser `humanize=True` |
| 匀速滚动 | 加速-匀速-减速 | CloakBrowser `humanize=True` |
| 瞬时输入 | 逐字延迟 + 打字错误 | CloakBrowser `humanize=True` |
| 无操作间隔 | 添加随机空闲微动作 | CloakBrowser `humanize=True` |

---

## 工具选型决策矩阵

| 反爬强度 | 特征 | HTTP Client | 浏览器 | 终极方案 |
|---------|------|------------|--------|---------|
| 无 | 静态 HTML，无任何验证 | `requests` | — | — |
| 低 | 只检查 UA | `requests` + UA 伪装 | — | — |
| 中 | TLS 指纹检测 | `curl_cffi` | Playwright + JS 注入 | — |
| 较高 | JS 渲染 + 轻度反爬 | — | Playwright / DrissionPage | — |
| 高 | Cloudflare/DataDome | — | CloakBrowser | Camoufox |
| 极高 | 验证码频繁弹出 | — | CloakBrowser + 住宅代理 | 商业方案 |

---

## 常见反爬系统识别

| 系统 | 特征 | 绕过难度 | 推荐方案 |
|------|------|---------|---------|
| Cloudflare | 5 秒盾 "Checking your browser" | 高 | CloakBrowser |
| DataDome | 拦截页 "You've been blocked" | 高 | CloakBrowser |
| PerimeterX | 静默指纹采集 | 中 | CloakBrowser |
| 阿里云 WAF | 滑块验证码 | 中 | ddddocr + 动作链 |
| 腾讯验证码 | 滑动/点选 | 中 | ddddocr + Playwright |
| reCAPTCHA v3 | 后台评分 | 中 | CloakBrowser (评分 0.9) |

---

## 实战原则

1. **渐进式升级**：先用最简单的方案（requests），遇到阻碍再升级。别一上来就上 CloakBrowser
2. **代理 > 指纹**：IP 信誉是第1关，IP 烂的话指纹伪装再好也没用
3. **行为是最终 boss**：过了前三层，请求节奏控制和鼠标行为是最后的分水岭
4. **爬取不是攻击**：合理频率、遵守 robots.txt、不爬敏感数据
