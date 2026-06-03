# 异步并发模式

## 什么时候用异步

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 单页/少量请求 | 同步（requests） | 异步反而增加复杂度 |
| 批量 HTTP 请求（50+） | `aiohttp` + `asyncio` | 异步 IO 不阻塞等待 |
| 多页浏览器并发 | `Playwright async` | 共享 browser 实例 |
| CPU 密集（解析大文件） | 线程池 `concurrent.futures` | 异步不适配 CPU 密集 |

---

## Semaphore 限流标准模式

```python
import asyncio
from aiohttp import ClientSession

async def fetch(session, semaphore, url):
    async with semaphore:                    # 控制并发数
        async with session.get(url) as resp:
            return await resp.text()

async def main(urls):
    semaphore = asyncio.Semaphore(10)        # 最多同时 10 个请求
    async with ClientSession() as session:
        tasks = [asyncio.create_task(fetch(session, semaphore, url)) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    # 过滤失败的
    return [r for r in results if not isinstance(r, Exception)]
```

Semaphore 值建议：
- HTTP 请求：10-20
- 浏览器 page：3-5
- 纯 API 小请求：20-50

---

## gather 错误处理（关键）

```python
# ❌ 错误写法 — 一个 task 失败全部白干
results = await asyncio.gather(*tasks)

# ✅ 正确写法 — 失败的不影响成功的
results = await asyncio.gather(*tasks, return_exceptions=True)

# 过滤异常并告警
success = []
for r in results:
    if isinstance(r, Exception):
        print(f"警告：某任务失败 — {r}")
    else:
        success.append(r)
```

---

## 扁平化 gather 结果

gather 返回 `[[[task1结果]], [[task2结果]], ...]`，多了一层：

```python
# ❌ 直接使用 — 三维列表
all_pages = await asyncio.gather(*task_list, return_exceptions=True)
# all_pages = [[['书名','价格'],...], [['书名','价格'],...], ...]

# ✅ 扁平化为二维
flat = [item for page in all_pages for item in page]
# flat = [['书名','价格'], ['书名','价格'], ...]
```

---

## 生产者-消费者模式

```python
import asyncio

async def producer(queue, urls):
    for url in urls:
        await queue.put(url)
    # 发送结束信号
    for _ in range(WORKERS):
        await queue.put(None)

async def consumer(queue, worker_id, results):
    while True:
        url = await queue.get()
        if url is None:
            break
        data = await fetch(url)
        results.append(data)
        queue.task_done()

async def main(urls):
    queue = asyncio.Queue()
    results = []
    workers = [asyncio.create_task(consumer(queue, i, results)) for i in range(5)]
    await producer(queue, urls)
    await asyncio.gather(*workers)
    return results
```

---

## 线程 vs 协程

| | `threading.Thread` | `asyncio` |
|---|---|---|
| 适用 | IO 密集（网络请求） | IO 密集（网络请求） |
| 开销 | 高（每个线程 ~8MB 栈） | 低（每个协程几 KB） |
| 并发量 | 几十 | 几百到几千 |
| 调试 | 简单（正常的函数调用栈） | 复杂（async/await 链） |
| 学习曲线 | 低 | 中 |

选型原则：日常用 `asyncio`，已经用 threading 的项目不用改。
