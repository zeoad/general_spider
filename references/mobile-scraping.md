# 移动端 APP 抓包爬虫

## 架构总览

```
手机 APP → 代理/抓包工具 → 电脑 → Python 复现请求
```

核心思路：
1. 模拟器里运行 APP，流量经过代理
2. 电脑上 Reqable 拦截 HTTPS 请求
3. 分析请求格式和加密参数
4. 用 Python 代码复现请求

---

## 推荐工具链

| 环节 | 推荐工具 | 说明 |
|------|---------|------|
| 安卓环境 | **MuMu 模拟器** | 网易出品，性能好、ROOT 方便 |
| 抓包 | **Reqable** | 国产免费，GUI 美观，支持 Windows/Mac/Linux |
| HTTPS 解密 | Reqable CA 证书 | App 内一键安装 |
| 绕过 SSL Pinning | Xposed + JustTrustMe | Hook SSL 校验 |
| API 分析 | Reqable 面板 | 直观展示请求/响应，支持复制为 cURL |

---

## 实操步骤

### 1. 安装 MuMu 模拟器

```
下载: https://mumu.163.com/
```

安装后启动 → 设置里开启 ROOT 权限。

### 2. 安装 Reqable

```
下载: https://reqable.com/
```

打开 Reqable → 默认端口 9000。界面分左右两栏：左边请求列表，右边请求/响应详情。

### 3. 模拟器配代理

MuMu 模拟器 → 设置 → WLAN → 长按当前 WiFi → 修改网络 → 手动代理：

```
主机名：你电脑的局域网 IP（命令行 ipconfig 查 IPv4 地址）
端口：9000（Reqable 默认）
```

### 4. 安装 HTTPS 证书

Reqable 菜单 → 证书管理 → 安装根证书到本机。

模拟器浏览器访问 `http://你的IP:9000/cert` 或扫描 Reqable 显示的证书二维码下载安装。安装后会提示设置锁屏密码。

如果 APP 检测到用户证书不算数，用 ADB 推到系统证书目录：

```bash
# MuMu 默认 ADB 端口 7555
adb connect 127.0.0.1:7555

# Push 证书到系统目录（需要 ROOT）
adb root
adb remount
adb push cert.cer /system/etc/security/cacerts/
adb reboot
```

### 5. 绕过 SSL Pinning

有些 APP 固定了证书指纹，即使装系统证书也不认代理。绕过去：

**Xposed + JustTrustMe（推荐）：**

在 MuMu 模拟器里安装 Xposed 框架 → 安装 JustTrustMe 模块 → 激活 → 重启。该模块 Hook 所有 SSL 校验函数，强制信任任意证书。

**Frida（高级备用）：**

```bash
pip install frida-tools
# 模拟器里装 frida-server，然后
frida -U -l bypass-ssl.js com.example.app
```

### 6. 抓包分析

Reqaable 开起来 → MuMu 里启动目标 APP → 看 Reqable 的请求列表：

- 找到数据接口（通常是 `.json` 或 `api/` 路径）
- 看请求头（Authorization、Cookie、签名参数）
- 看请求体（加密参数、分页参数）
- 右键 → 复制为 cURL → 导入 Python

### 7. 用 Python 复现请求

```python
import requests

url = "https://api.example.com/v1/list"
headers = {
    "User-Agent": "okhttp/4.9.0",  # APP 的 UA
    "Authorization": "Bearer xxx",
    "X-Sign": "md5_of_something",   # 需要逆向
}
payload = {
    "page": 1,
    "size": 20,
}

resp = requests.post(url, json=payload, headers=headers)
data = resp.json()
```

### 8. Reqable + Python 自动化

Reqable 支持导出 HAR 文件，可以用 Python 解析：

```python
import json

with open("reqable_export.har", "r", encoding="utf-8") as f:
    har = json.load(f)

for entry in har["log"]["entries"]:
    request = entry["request"]
    url = request["url"]
    method = request["method"]
    headers = {h["name"]: h["value"] for h in request["headers"]}
    # 筛选目标接口
    if "api" in url:
        print(f"{method} {url}")
        print(f"Headers: {headers}")
```

---

## 移动端特有反爬

| 反爬手段 | 表现 | 应对 |
|---------|------|------|
| SSL Pinning | 代理抓不到包 | Xposed + JustTrustMe / Frida |
| 代理检测 | APP 弹窗"检测到代理" | VPN 模式（Postern）绕过代理检测 |
| 签名校验 | API 返回 signature error | jadx 反编译 APK，找签名算法 |
| 设备指纹 | 每个请求带 device_id | 分析 device_id 生成逻辑 |
| 时间戳校验 | 时间差过大返回错误 | 同步时间，添加正确时间戳参数 |
| 双向证书校验 | 客户端也需要证书 | 从 APK 提取客户端证书 |

---

## APK 逆向工具

| 库 | 用途 |
|---|------|
| `jadx` | 反编译 dex 查看 Java 源码 |
| `frida` | 动态调试、Hook 函数 |
| `apktool` | APK 解包/重打包 |
| `unidbg` | 模拟执行 so 中的加密函数（无需真机） |

---

## 完整技术栈

| 层级 | 推荐方案 |
|------|---------|
| 环境 | MuMu 模拟器 + ROOT + Xposed |
| 抓包 | Reqable |
| HTTPS | Reqable CA 证书 + JustTrustMe |
| 逆向 | jadx（Java）+ Frida（Hook）+ IDA（so） |
| 复现 | Python requests / curl_cffi |
| 存储 | CSV / SQLite / MongoDB |
