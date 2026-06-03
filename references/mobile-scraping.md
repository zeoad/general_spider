# 移动端 APP 抓包爬虫

## 架构总览

```
手机 APP → 代理/抓包工具 → 电脑 → Python 复现请求
```

核心思路：
1. 手机上运行 APP，流量经过代理
2. 电脑上抓包工具拦截 HTTPS 请求
3. 分析请求格式和加密参数
4. 用 Python 代码复现请求

---

## 工具链

| 环节 | 方案 A（体验最好） | 方案 B（轻量） |
|------|-------------------|---------------|
| 安卓环境 | MEmu（逍遥）/ MuMu 模拟器 | 真机 USB 连接 |
| 抓包 | Charles（收费，GUI 直观） | mitmproxy（免费，命令行） |
| HTTPS 解密 | 安装 Charles CA 证书 | 安装 mitmproxy CA 证书 |
| 绕过 SSL Pinning | Xposed + JustTrustMe | Frida 脚本 |
| API 分析 | 人肉看 Charles 面板 | 导出 har 文件到 Python 解析 |

---

## 实操步骤

### 1. 安装模拟器

```
MEmu: https://www.memuplay.com/
MuMu: https://mumu.163.com/
```

安装后启动，模拟器里就是一个安卓手机。开启 ROOT 权限（设置里勾选）。

### 2. 安装抓包工具

**Charles（推荐）：**

```
下载: https://www.charlesproxy.com/download/
```

打开后：Proxy → Proxy Settings → HTTP Proxy → Port: 8888

**mitmproxy（免费替代）：**

```bash
pip install mitmproxy
mitmweb  # 浏览器打开 http://127.0.0.1:8081 查看
```

### 3. 模拟器配代理

模拟器 WiFi 设置 → 长按当前 WiFi → 修改网络 → 手动代理：

```
主机名：你电脑的局域网 IP（命令行 ipconfig 查 IPv4）
端口：8888（Charles）或 8080（mitmproxy）
```

### 4. 安装 HTTPS 证书

模拟器浏览器访问：
```
http://chls.pro/ssl     （Charles）
http://mitm.it           （mitmproxy）
```

下载证书 → 安装 → 会提示设置锁屏密码。只装用户证书不够时，用 ADB 推到系统证书目录：

```bash
# 获取模拟器 ADB 连接
adb connect 127.0.0.1:21503  # MEmu 默认端口

# Push 证书到系统目录（需要 ROOT）
adb root
adb remount
adb push cert.cer /system/etc/security/cacerts/
adb reboot
```

### 5. 绕过 SSL Pinning

有些 APP 在代码里固定了证书指纹，即使装了系统证书也不信任代理。需要：

**方案 A：Xposed + JustTrustMe**

在模拟器里安装 Xposed 框架 → 安装 JustTrustMe 模块 → 激活 → 重启。该模块会 Hook 所有 SSL 校验函数，强制信任任意证书。

**方案 B：Frida**

```bash
pip install frida-tools
# 模拟器里安装 frida-server，然后
frida -U -l bypass-ssl.js com.example.app
```

### 6. 抓包分析

打开 Charles / mitmweb → 模拟器里启动目标 APP → 看 HTTPS 请求流：

- 找到数据接口（通常是 `.json` 或 `api/` 路径）
- 看请求头（Authorization、Cookie、签名参数）
- 看请求体（加密参数、分页参数）
- 复制为 cURL → 导入 Python

### 7. 用 Python 复现请求

```python
import requests

url = "https://api.example.com/v1/list"
headers = {
    "User-Agent": "okhttp/4.9.0",  # APP 的 UA
    "Authorization": "Bearer xxx",
    "X-Sign": "md5_of_something",   # 需要逆向（见下文）
}
payload = {
    "page": 1,
    "size": 20,
}

resp = requests.post(url, json=payload, headers=headers)
data = resp.json()
```

---

## 移动端特有反爬

| 反爬手段 | 表现 | 应对 |
|---------|------|------|
| SSL Pinning | 代理抓不到包 | Xposed + JustTrustMe / Frida |
| 代理检测 | APP 弹窗"检测到代理" | VPN 模式（Postern）绕过代理检测 |
| 签名校验 | API 返回 signature error | 逆向 APP APK，找签名算法 |
| 设备指纹 | 每个请求带 device_id | 分析 device_id 生成逻辑 |
| 时间戳校验 | 时间差过大返回错误 | 同步手机时间，添加时间戳参数 |
| 双向证书校验 | 客户端也要证书 | 从 APK 提取客户端证书 |

---

## Python 常用移动端逆向库

| 库 | 用途 |
|---|------|
| `frida` | 动态调试、Hook 函数 |
| `androlib` / `apktool` | APK 反编译 |
| `jadx` | 反编译 dex 查看 Java 源码 |
| `unidbg` | 模拟执行 so 中的加密函数 |
| `mitmproxy` | 可编程抓包（Python 脚本自动化） |

---

## 实战：mitmproxy Python 脚本

不用手动分析，直接写脚本自动化：

```python
# auto_capture.py
from mitmproxy import http

def response(flow: http.HTTPFlow):
    # 只关注 API 接口
    if "api" in flow.request.pretty_url:
        url = flow.request.url
        status = flow.response.status_code
        body = flow.response.text
        print(f"[{status}] {url}")
        # 自动保存到文件
        with open("api_log.txt", "a", encoding="utf-8") as f:
            f.write(f"=== {url} ===\n{body}\n\n")
```

运行：
```bash
mitmproxy -s auto_capture.py -p 8080
```

---

## 完整技术栈推荐

| 层级 | 工具 |
|------|------|
| 环境 | MEmu/MuMu + ROOT + Xposed |
| 抓包 | Charles（初级）/ mitmproxy + Python 脚本（高级） |
| HTTPS | 系统 CA 证书 + JustTrustMe |
| 逆向 | jadx（看 Java 代码）+ Frida（Hook）+ IDA（看 so） |
| 复现 | Python requests / curl_cffi |
| 存储 | CSV（简单）/ SQLite（增量）/ MongoDB（非结构化） |
