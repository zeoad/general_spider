# DrissionPage 进阶参考

## 定位

DrissionPage 是国人开发的浏览器自动化框架，API 比 Selenium 简洁，内置网络监听和反检测。

- Selenium 的重型替代
- Playwright 的轻量补充
- 特别适合国内中小站点

---

## 启动方式

```python
from DrissionPage import ChromiumPage, ChromiumOptions

# 基础启动
page = ChromiumPage()

# 无头模式
co = ChromiumOptions().headless()
page = ChromiumPage(co)

# 带代理
co = ChromiumOptions().set_proxy('http://127.0.0.1:7890')
page = ChromiumPage(co)
```

---

## 元素操作

```python
# 定位元素（支持 xpath、css、text 等多种方式）
ele = page.ele('xpath://input[@id="kw"]')   # XPath
ele = page.ele('#kw')                        # CSS Selector
ele = page.ele('text:百度一下')               # 按文字

# 输入和点击
ele.input('搜索关键词')
page.ele('#su').click()

# 获取属性
text = ele.text         # 可见文本
html = ele.html         # 内部 HTML
attr = ele.attr('href')  # 属性值
```

---

## 网络监听（核心优势）

Selenium 没有原生网络监听，DrissionPage 内置：

```python
from DrissionPage import ChromiumPage

page = ChromiumPage()
page.listen.start('api/detail')  # 监听包含 'api/detail' 的请求
page.get('https://example.com')

packet = page.listen.wait()       # 等待匹配的请求完成
print(packet.response.body)       # 直接拿到响应内容
```

比 Playwright 的 `page.route()` / `page.on('response')` 更直观。

---

## Session 和 Tab 混合

同一个对象可以发 HTTP 请求 + 控浏览器：

```python
page = ChromiumPage()
# 先发 HTTP 请求获取 token
resp = page.session.post('https://example.com/api/login', data={...})
# 再把 token 注入浏览器
page.run_js(f'localStorage.setItem("token", "{resp.json()["token"]}")')
# 然后浏览器正常访问
page.get('https://example.com/app')
```

---

## iframe 处理

```python
# 直接获取 iframe 元素，无需 switch_to
iframe = page.ele('tag:iframe')
# 在 iframe 内定位
inner = iframe.ele('#content')
```

---

## 反检测配置

```python
from DrissionPage import ChromiumPage, ChromiumOptions

co = ChromiumOptions()
co.set_argument('--disable-blink-features=AutomationControlled')
co.set_argument('--no-sandbox')
co.set_argument('--disable-gpu')
co.set_pref('excludeSwitches', 'enable-automation')

page = ChromiumPage(co)
```

---

## 与 Selenium 对比速查

| 操作 | Selenium | DrissionPage |
|------|----------|--------------|
| 启动耗时 | ~3s | ~1s |
| 元素定位 | `find_element(By.XPATH, '//...')` | `ele('xpath://...')` |
| 输入 | `send_keys('text')` | `input('text')` |
| iframe | `switch_to.frame(el)` | 直接 `ele('tag:iframe')` |
| 网络监听 | ❌ | ✅ `listen.start()` |
| 滑块操作 | `ActionChains` 链式调用 | `actions.hold().move().release()` |
| Cookie 操作 | `add_cookie()` 逐条加 | `set.cookies(dict)` 批量 |
| 截图 | `get_screenshot_as_file()` | `get_screenshot()` |
