---
name: python-concurrency
description: Python编码规范 - 并发规范。适用于所有Python代码开发场景：(1) 编写多线程/多进程代码；(2) asyncio异步编程；(3) 共享变量访问控制；(4) 死锁预防。触发关键词：Python、线程、threading、asyncio、并发、concurrent、锁、lock、GIL、multiprocessing
---

# Python并发规范

基于Clean Code指导书

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 共享变量必须同步访问 |
| 规则1.2 | 优先使用asyncio而非线程 |
| 规则1.3 | 避免死锁：固定加锁顺序 |
| 规则1.4 | 使用queue实现线程间通信 |
| 规则1.5 | CPU密集型用multiprocessing |
| 规则1.6 | 理解GIL何时释放，选择正确的并发模型 |

## 线程安全示例

```python
import threading
from concurrent.futures import ThreadPoolExecutor

class SafeCounter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()

    def increment(self):
        with self._lock:
            self._value += 1

    @property
    def value(self):
        with self._lock:
            return self._value
```

## asyncio示例

```python
import asyncio

async def fetch_data(url: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

async def fetch_all(urls: list[str]) -> list[dict]:
    tasks = [fetch_data(url) for url in urls]
    return await asyncio.gather(*tasks)
```

## GIL深度解析

```python
import sys
import multiprocessing

# GIL释放时机
# 1. I/O操作：自动释放（文件、网络、sleep）
# 2. 系统调用：自动释放
# 3. C扩展：可手动释放（NumPy等）
# 4. Check interval：每5ms检查线程切换

# 绕过GIL的方法
# 1. multiprocessing：多进程绕过GIL
# 2. NumPy/Cython：C扩展释放GIL
# 3. asyncio：单线程异步避免GIL争用

# 选择并发模型
# IO密集型 → threading / asyncio（GIL期间释放）
# CPU密集型 → multiprocessing（多进程无GIL）
# 大量并发IO → asyncio（单线程高效）

print(f"GIL switch interval: {sys.getswitchinterval()}s")
```

## 详细规范

见 [references/concurrency.md](references/concurrency.md)

## 检查清单

- [ ] 共享变量是否使用锁保护？
- [ ] 是否选择了正确的并发模型？（threading/asyncio/multiprocessing）
- [ ] CPU密集型任务是否避免了多线程？
- [ ] 是否理解GIL何时释放？
- [ ] 是否避免了死锁？
