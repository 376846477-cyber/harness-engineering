# Python并发编程规范详细参考

## 1. GIL限制与并发选择

### 1.1 GIL基本概念

CPython的全局解释器锁（GIL）限制了多线程在CPU密集型任务上的并行执行能力。

**选择策略：**
- IO密集型任务：使用 `threading` 或 `asyncio`
- CPU密集型任务：使用 `multiprocessing`
- 混合型任务：使用 `concurrent.futures` 线程池+进程池组合

```python
import time
import threading
import multiprocessing

def cpu_bound_task(n):
    return sum(i * i for i in range(n))

def io_bound_task(duration):
    time.sleep(duration)
    return duration

def cpu_intensive():
    start = time.time()
    with multiprocessing.Pool() as pool:
        results = pool.map(cpu_bound_task, [10**6] * 4)
    print(f"Multiprocessing: {time.time() - start:.2f}s")

def io_intensive():
    start = time.time()
    threads = [threading.Thread(target=io_bound_task, args=(1,)) for _ in range(4)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    print(f"Threading: {time.time() - start:.2f}s")
```

### 1.2 GIL深度解析

#### 1.2.1 GIL存在的原因

**历史背景：**
GIL自1992年引入CPython，主要解决以下问题：
1. **内存管理非线程安全**：Python的引用计数机制需要GIL保护
2. **C扩展兼容性**：大量C扩展库假设GIL存在
3. **单线程性能**：GIL简化了实现，提升单线程性能

```python
# 引用计数示例 - 需要GIL保护
import sys

x = []
y = x  # 引用计数增加
print(sys.getrefcount(x))  # 3 (x, y, getrefcount参数)

del y  # 引用计数减少
# 这些操作在C层面是非原子的，需要GIL保护
```

#### 1.2.2 GIL何时释放

**规则：Python在特定时机自动释放GIL**

```python
import time
import dis

# 1. I/O操作自动释放GIL
def io_operation():
    with open('large_file.txt', 'r') as f:
        data = f.read()  # GIL在此期间释放
    time.sleep(1)  # GIL在此期间释放

# 2. 系统调用自动释放GIL
import os
def syscall_example():
    os.system('ls -la')  # GIL释放

# 3. CPU密集型操作（部分库）
import numpy as np
def numpy_operation():
    # NumPy在大量计算时释放GIL
    arr = np.random.rand(1000000)
    result = np.dot(arr, arr)  # GIL释放期间执行

# 4. 查看字节码 - 理解GIL粒度
def simple_loop():
    total = 0
    for i in range(1000):
        total += i
    return total

dis.dis(simple_loop)
# 每个字节码执行期间持有GIL
# 字节码之间可能释放GIL（Python 3.2+的check interval机制）
```

#### 1.2.3 GIL的Check Interval机制

```python
import sys

# Python 3.2+ 使用时间而非指令数
# 默认：每5ms检查是否切换线程
print(f"Switch interval: {sys.getswitchinterval()}")  # 0.005 (5ms)

# 调整切换间隔（影响线程调度频率）
sys.setswitchinterval(0.001)  # 更频繁的线程切换

# 注意：更短的interval意味着更多上下文切换开销
# 对于CPU密集型任务，可能反而降低性能
```

#### 1.2.4 GIL对性能的实际影响

```python
import time
import threading

def cpu_bound(n):
    """CPU密集型任务 - GIL导致串行执行"""
    total = 0
    for i in range(n):
        total += i * i
    return total

def measure_cpu_bound():
    n = 10**7
    
    # 单线程
    start = time.time()
    result1 = cpu_bound(n)
    single_time = time.time() - start
    
    # 多线程（无加速，甚至更慢）
    start = time.time()
    t1 = threading.Thread(target=cpu_bound, args=(n//2,))
    t2 = threading.Thread(target=cpu_bound, args=(n//2,))
    t1.start()
    t2.start()
    t1.join()
    t2.join()
    multi_time = time.time() - start
    
    print(f"单线程: {single_time:.2f}s")
    print(f"多线程: {multi_time:.2f}s")
    print(f"加速比: {single_time/multi_time:.2f}x")
    # 输出示例：
    # 单线程: 1.23s
    # 多线程: 1.45s  (更慢！因为GIL)
    # 加速比: 0.85x

def io_bound(n):
    """IO密集型任务 - GIL期间释放，多线程有效"""
    import time
    time.sleep(n)
    return n

def measure_io_bound():
    # 单线程
    start = time.time()
    io_bound(1)
    io_bound(1)
    single_time = time.time() - start
    
    # 多线程
    start = time.time()
    t1 = threading.Thread(target=io_bound, args=(1,))
    t2 = threading.Thread(target=io_bound, args=(1,))
    t1.start()
    t2.start()
    t1.join()
    t2.join()
    multi_time = time.time() - start
    
    print(f"单线程: {single_time:.2f}s")
    print(f"多线程: {multi_time:.2f}s")
    print(f"加速比: {single_time/multi_time:.2f}x")
    # 输出示例：
    # 单线程: 2.00s
    # 多线程: 1.00s  (接近2x加速)
    # 加速比: 2.00x
```

#### 1.2.5 绕过GIL的方法

**方法1：使用multiprocessing**

```python
from multiprocessing import Pool, cpu_count
import time

def heavy_computation(n):
    total = 0
    for i in range(n):
        total += i * i
    return total

def parallel_compute():
    n = 10**7
    num_workers = cpu_count()
    
    start = time.time()
    with Pool(num_workers) as pool:
        results = pool.map(heavy_computation, [n // num_workers] * num_workers)
    total = sum(results)
    
    print(f"Parallel: {time.time() - start:.2f}s")
    print(f"Total: {total}")
```

**方法2：使用释放GIL的C扩展**

```python
# NumPy释放GIL
import numpy as np

def numpy_intensive():
    large_arr = np.random.rand(10**7)
    # NumPy在底层C代码中释放GIL
    result = np.sum(large_arr ** 2)
    return result

# 使用ctypes调用C代码时释放GIL
import ctypes

def call_c_with_gil_released():
    lib = ctypes.CDLL('./mylib.so')
    
    # 方法1：调用前手动释放
    ctypes.pythonapi.PyEval_SaveThread()
    try:
        result = lib.heavy_computation()
    finally:
        ctypes.pythonapi.PyEval_RestoreThread(ctypes.pythonapi.PyEval_SaveThread())
    
    # 方法2：使用Py_BEGIN_ALLOW_THREADS宏（C扩展开发）
```

**方法3：使用Cython释放GIL**

```python
# mymodule.pyx
# cython: language_level=3

from cython.parallel import prange

def parallel_sum(int n):
    cdef long total = 0
    cdef int i
    
    # nogil上下文释放GIL
    with nogil:
        for i in prange(n, nogil=True):
            total += i * i
    
    return total

# 使用：
# from mymodule import parallel_sum
# result = parallel_sum(10**7)  # 无GIL并行执行
```

**方法4：使用替代Python实现**

```python
# Jython (JVM上的Python) - 无GIL
# IronPython (.NET上的Python) - 无GIL
# PyPy - 有GIL但优化更好

# 注意：这些实现与CPython不完全兼容
# 特别是C扩展库可能无法使用
```

#### 1.2.6 GIL相关的最佳实践

```python
import threading
import multiprocessing
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# 规则1：识别任务类型选择正确的并发模型
def choose_concurrency_model(task_type: str):
    """根据任务类型选择并发模型"""
    models = {
        'io_bound': 'threading 或 asyncio',      # 网络请求、文件IO
        'cpu_bound': 'multiprocessing',           # 数值计算、图像处理
        'mixed': 'ThreadPoolExecutor + ProcessPoolExecutor',
        'high_concurrency_io': 'asyncio',         # 大量并发连接
    }
    return models.get(task_type, 'single_thread')

# 规则2：避免在CPU密集型任务中使用多线程
def bad_practice():
    """错误示例：CPU密集型任务用多线程"""
    results = []
    lock = threading.Lock()
    
    def cpu_task():
        total = sum(i*i for i in range(10**6))  # CPU密集
        with lock:
            results.append(total)
    
    threads = [threading.Thread(target=cpu_task) for _ in range(4)]
    for t in threads: t.start()
    for t in threads: t.join()
    # 结果：比单线程更慢！

# 规则3：使用释放GIL的库
def good_practice_with_numpy():
    """正确示例：使用NumPy释放GIL"""
    import numpy as np
    
    # NumPy数组操作在C层面执行，释放GIL
    data = np.random.rand(10**6)
    result = np.dot(data, data)  # 并行执行

# 规则4：合理设置线程/进程数
import os

def optimal_worker_count():
    """确定最优工作线程/进程数"""
    cpu_count = os.cpu_count() or 4
    
    # IO密集型：线程数可以是CPU核心数的2-10倍
    io_workers = cpu_count * 5
    
    # CPU密集型：进程数等于CPU核心数
    cpu_workers = cpu_count
    
    return io_workers, cpu_workers

# 规则5：监控GIL争用
def monitor_gil_contention():
    """监控线程等待GIL的时间"""
    import sys
    import threading
    import time
    
    # 使用sys.setswitchinterval()调整检查频率
    # 使用threading.active_count()监控线程数
    # 复杂场景可使用性能分析工具：py-spy, viztracer
    
    print(f"Active threads: {threading.active_count()}")
    print(f"Switch interval: {sys.getswitchinterval()}s")
```

## 2. threading模块

### 2.1 基本线程创建

```python
import threading
from threading import Thread, Lock, RLock, Condition, Event, Semaphore

class WorkerThread(Thread):
    def __init__(self, name, task_queue, result_queue):
        super().__init__(name=name)
        self.task_queue = task_queue
        self.result_queue = result_queue
        self._stop_event = Event()
    
    def run(self):
        while not self._stop_event.is_set():
            try:
                task = self.task_queue.get(timeout=1)
                result = self.process_task(task)
                self.result_queue.put(result)
            except queue.Empty:
                continue
    
    def process_task(self, task):
        return f"Processed: {task}"
    
    def stop(self):
        self._stop_event.set()
        self.join(timeout=5)
```

### 2.2 线程安全机制

**Lock（互斥锁）：**
```python
import threading

class Counter:
    def __init__(self):
        self._value = 0
        self._lock = Lock()
    
    def increment(self):
        with self._lock:
            self._value += 1
            return self._value
    
    def get_value(self):
        with self._lock:
            return self._value

counter = Counter()

def safe_increment():
    for _ in range(1000):
        counter.increment()
```

**RLock（可重入锁）：**
```python
class ReentrantExample:
    def __init__(self):
        self._rlock = RLock()
        self._data = []
    
    def add_item(self, item):
        with self._rlock:
            self._data.append(item)
            self._log(f"Added: {item}")
    
    def _log(self, message):
        with self._rlock:
            print(f"[LOG] {message}")
    
    def get_data(self):
        with self._rlock:
            return self._data.copy()
```

**Semaphore（信号量）：**
```python
class ConnectionPool:
    def __init__(self, max_connections=5):
        self._semaphore = Semaphore(max_connections)
        self._connections = []
    
    def acquire_connection(self):
        self._semaphore.acquire()
        return self._create_connection()
    
    def release_connection(self, conn):
        self._destroy_connection(conn)
        self._semaphore.release()
    
    def _create_connection(self):
        return {"id": id(self), "active": True}
    
    def _destroy_connection(self, conn):
        conn["active"] = False

pool = ConnectionPool(max_connections=3)

def use_connection():
    conn = pool.acquire_connection()
    try:
        time.sleep(1)
    finally:
        pool.release_connection(conn)
```

### 2.3 Condition和Event

**Condition（条件变量）：**
```python
class BoundedBuffer:
    def __init__(self, capacity):
        self._buffer = []
        self._capacity = capacity
        self._condition = Condition(Lock())
    
    def put(self, item):
        with self._condition:
            while len(self._buffer) >= self._capacity:
                self._condition.wait()
            self._buffer.append(item)
            self._condition.notify_all()
    
    def get(self):
        with self._condition:
            while len(self._buffer) == 0:
                self._condition.wait()
            item = self._buffer.pop(0)
            self._condition.notify_all()
            return item
```

**Event（事件）：**
```python
class ShutdownController:
    def __init__(self):
        self._shutdown_event = Event()
    
    def trigger_shutdown(self):
        self._shutdown_event.set()
    
    def wait_for_shutdown(self, timeout=None):
        return self._shutdown_event.wait(timeout)
    
    def is_shutdown(self):
        return self._shutdown_event.is_set()
    
    def reset(self):
        self._shutdown_event.clear()
```

## 3. queue模块 - 线程安全队列

### 3.1 Queue、LifoQueue、PriorityQueue

```python
import queue
from queue import Queue, LifoQueue, PriorityQueue
from dataclasses import dataclass, field
from typing import Any

@dataclass(order=True)
class PrioritizedItem:
    priority: int
    item: Any = field(compare=False)

class TaskScheduler:
    def __init__(self):
        self._task_queue = PriorityQueue()
        self._result_queue = Queue()
        self._workers = []
    
    def add_task(self, priority, task):
        prioritized = PrioritizedItem(priority, task)
        self._task_queue.put(prioritized)
    
    def start_workers(self, num_workers=4):
        for i in range(num_workers):
            worker = Thread(target=self._worker_loop, args=(i,))
            worker.daemon = True
            worker.start()
            self._workers.append(worker)
    
    def _worker_loop(self, worker_id):
        while True:
            try:
                prioritized = self._task_queue.get(timeout=1)
                result = self._process(prioritized.item)
                self._result_queue.put((worker_id, result))
                self._task_queue.task_done()
            except queue.Empty:
                continue
    
    def _process(self, task):
        return f"Result of {task}"

queue_example = Queue(maxsize=100)
lifo_queue = LifoQueue()
priority_queue = PriorityQueue()
```

## 4. multiprocessing模块

### 4.1 进程创建与通信

```python
import multiprocessing
from multiprocessing import Process, Pool, Queue, Manager, Value, Array, Pipe

def worker_process(name, task_queue, result_queue):
    while True:
        try:
            task = task_queue.get(timeout=5)
            if task is None:
                break
            result = f"Process {name}: {task}"
            result_queue.put(result)
        except:
            break

def run_multiprocessing():
    task_queue = Queue()
    result_queue = Queue()
    
    for i in range(10):
        task_queue.put(f"Task-{i}")
    
    processes = []
    for i in range(4):
        p = Process(target=worker_process, args=(i, task_queue, result_queue))
        p.start()
        processes.append(p)
    
    for p in processes:
        task_queue.put(None)
    
    for p in processes:
        p.join()
    
    results = []
    while not result_queue.empty():
        results.append(result_queue.get())
    
    return results
```

### 4.2 共享内存与Manager

```python
def shared_memory_example():
    counter = Value('i', 0)
    data_array = Array('i', [0] * 10)
    
    def increment(counter, array, index):
        with counter.get_lock():
            counter.value += 1
        array[index] = counter.value
    
    processes = [
        Process(target=increment, args=(counter, data_array, i))
        for i in range(10)
    ]
    
    for p in processes:
        p.start()
    for p in processes:
        p.join()
    
    return counter.value, list(data_array)

def manager_example():
    with Manager() as manager:
        shared_dict = manager.dict()
        shared_list = manager.list()
        
        def worker(d, l, i):
            d[f"key-{i}"] = i * 10
            l.append(i)
        
        processes = [
            Process(target=worker, args=(shared_dict, shared_list, i))
            for i in range(5)
        ]
        
        for p in processes:
            p.start()
        for p in processes:
            p.join()
        
        return dict(shared_dict), list(shared_list)
```

### 4.3 进程池

```python
def cpu_intensive_task(n):
    return sum(i ** 2 for i in range(n))

def pool_example():
    with Pool(processes=4) as pool:
        results = pool.map(cpu_intensive_task, [10**5] * 8)
    
    with Pool(processes=4) as pool:
        results = pool.map_async(cpu_intensive_task, [10**5] * 8)
        results.wait()
    
    with Pool(processes=4) as pool:
        for result in pool.imap(cpu_intensive_task, [10**5] * 8):
            print(result)
    
    return results
```

## 5. asyncio异步编程

### 5.1 协程基础

```python
import asyncio
from asyncio import Lock, Semaphore, Queue, Event, Condition

async def fetch_data(url, delay):
    await asyncio.sleep(delay)
    return f"Data from {url}"

async def process_data(data):
    await asyncio.sleep(0.1)
    return f"Processed: {data}"

async def main_async():
    urls = ["url1", "url2", "url3"]
    delays = [1, 2, 1.5]
    
    tasks = [fetch_data(url, delay) for url, delay in zip(urls, delays)]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    for result in results:
        if isinstance(result, Exception):
            print(f"Error: {result}")
        else:
            processed = await process_data(result)
            print(processed)

async def timed_operation():
    try:
        result = await asyncio.wait_for(
            fetch_data("slow-url", 10),
            timeout=2.0
        )
    except asyncio.TimeoutError:
        print("Operation timed out")

async def run_with_timeout():
    task = asyncio.create_task(fetch_data("url", 5))
    try:
        result = await asyncio.wait_for(task, timeout=2)
    except asyncio.TimeoutError:
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            print("Task cancelled")
```

### 5.2 异步同步原语

```python
class AsyncCounter:
    def __init__(self):
        self._value = 0
        self._lock = Lock()
    
    async def increment(self):
        async with self._lock:
            self._value += 1
            return self._value
    
    async def get_value(self):
        async with self._lock:
            return self._value

class AsyncConnectionPool:
    def __init__(self, max_connections=5):
        self._semaphore = Semaphore(max_connections)
        self._connections = []
    
    async def acquire(self):
        await self._semaphore.acquire()
        return await self._create_connection()
    
    async def release(self, conn):
        await self._destroy_connection(conn)
        self._semaphore.release()
    
    async def _create_connection(self):
        await asyncio.sleep(0.1)
        return {"id": id(self), "active": True}
    
    async def _destroy_connection(self, conn):
        conn["active"] = False

class AsyncProducerConsumer:
    def __init__(self, buffer_size=10):
        self._queue = Queue(maxsize=buffer_size)
        self._stop_event = Event()
    
    async def produce(self, item):
        await self._queue.put(item)
    
    async def consume(self):
        while not self._stop_event.is_set():
            try:
                item = await asyncio.wait_for(self._queue.get(), timeout=1)
                yield item
            except asyncio.TimeoutError:
                continue
    
    def stop(self):
        self._stop_event.set()
```

### 5.3 异步上下文管理器

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def async_resource(name):
    print(f"Acquiring {name}")
    await asyncio.sleep(0.1)
    try:
        yield {"name": name, "active": True}
    finally:
        print(f"Releasing {name}")
        await asyncio.sleep(0.1)

async def use_async_resource():
    async with async_resource("db-connection") as conn:
        await asyncio.sleep(1)
        print(f"Using {conn['name']}")
```

## 6. concurrent.futures

### 6.1 ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor, as_completed

def io_task(url):
    time.sleep(1)
    return f"Result from {url}"

def thread_pool_example():
    urls = [f"url-{i}" for i in range(10)]
    
    with ThreadPoolExecutor(max_workers=5) as executor:
        future_to_url = {executor.submit(io_task, url): url for url in urls}
        
        results = []
        for future in as_completed(future_to_url):
            url = future_to_url[future]
            try:
                result = future.result(timeout=10)
                results.append((url, result))
            except Exception as e:
                print(f"{url} generated exception: {e}")
    
    return results

def thread_pool_map_example():
    urls = [f"url-{i}" for i in range(10)]
    
    with ThreadPoolExecutor(max_workers=5) as executor:
        results = list(executor.map(io_task, urls, timeout=10))
    
    return results
```

### 6.2 ProcessPoolExecutor

```python
def cpu_task(n):
    return sum(i ** 2 for i in range(n))

def process_pool_example():
    numbers = [10**5] * 8
    
    with ProcessPoolExecutor(max_workers=4) as executor:
        results = list(executor.map(cpu_task, numbers))
    
    return results

def mixed_executor_example():
    with ThreadPoolExecutor(max_workers=4) as io_pool:
        with ProcessPoolExecutor(max_workers=4) as cpu_pool:
            io_futures = [io_pool.submit(io_task, f"url-{i}") for i in range(5)]
            cpu_futures = [cpu_pool.submit(cpu_task, 10**5) for _ in range(4)]
            
            all_futures = as_completed(io_futures + cpu_futures)
            results = [f.result() for f in all_futures]
    
    return results
```

## 7. 死锁预防

### 7.1 死锁原因与预防策略

**死锁四个必要条件：**
1. 互斥条件
2. 持有并等待
3. 不可抢占
4. 循环等待

**预防策略：**
- 固定加锁顺序（破坏循环等待）
- 使用超时（避免无限等待）
- 避免嵌套锁（减少持有等待）
- 使用with语句确保锁释放

### 7.2 死锁预防实现

```python
import threading
from contextlib import contextmanager
from typing import Set

class DeadlockPreventingLock:
    _global_order = {}
    _order_lock = threading.Lock()
    _current_order = threading.local()
    
    def __init__(self, name):
        self._lock = Lock()
        self._name = name
        with DeadlockPreventingLock._order_lock:
            if name not in DeadlockPreventingLock._global_order:
                DeadlockPreventingLock._global_order[name] = len(DeadlockPreventingLock._global_order)
    
    def acquire(self, blocking=True, timeout=-1):
        if hasattr(self._current_order, 'held'):
            current_max = max(self._current_order.held) if self._current_order.held else -1
            my_order = DeadlockPreventingLock._global_order[self._name]
            if my_order <= current_max:
                raise RuntimeError(f"Lock order violation: {self._name}")
        
        acquired = self._lock.acquire(blocking, timeout)
        if acquired:
            if not hasattr(self._current_order, 'held'):
                self._current_order.held = set()
            self._current_order.held.add(DeadlockPreventingLock._global_order[self._name])
        return acquired
    
    def release(self):
        self._current_order.held.discard(DeadlockPreventingLock._global_order[self._name])
        self._lock.release()
    
    def __enter__(self):
        self.acquire()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()

class TimeoutLock:
    def __init__(self, default_timeout=5.0):
        self._lock = Lock()
        self._default_timeout = default_timeout
    
    @contextmanager
    def acquire_timeout(self, timeout=None):
        timeout = timeout or self._default_timeout
        acquired = self._lock.acquire(timeout=timeout)
        if not acquired:
            raise TimeoutError(f"Lock acquisition timed out after {timeout}s")
        try:
            yield
        finally:
            self._lock.release()

class ResourceHierarchy:
    def __init__(self):
        self._lock_a = Lock()
        self._lock_b = Lock()
        self._lock_order = {"A": 1, "B": 2}
    
    def operation_ab(self):
        with self._lock_a:
            time.sleep(0.1)
            with self._lock_b:
                return "AB"
    
    def operation_ba(self):
        with self._lock_a:
            time.sleep(0.1)
            with self._lock_b:
                return "BA"
    
    def safe_operation(self, resources):
        sorted_resources = sorted(resources, key=lambda r: self._lock_order[r])
        locks = {"A": self._lock_a, "B": self._lock_b}
        
        @contextmanager
        def acquire_all():
            for r in sorted_resources:
                locks[r].acquire()
            try:
                yield
            finally:
                for r in reversed(sorted_resources):
                    locks[r].release()
        
        with acquire_all():
            return f"Acquired {sorted_resources}"
```

## 8. 检查清单

### 8.1 threading检查清单

- [ ] 共享状态是否使用Lock/RLock保护
- [ ] 是否使用with语句管理锁
- [ ] 守护线程是否正确设置
- [ ] 线程退出是否优雅处理
- [ ] 是否使用线程安全队列（queue.Queue）
- [ ] 全局变量是否最小化使用
- [ ] 线程局部存储是否正确使用（threading.local）

### 8.2 multiprocessing检查清单

- [ ] 进程间通信是否使用Queue/Pipe
- [ ] 共享内存是否正确同步
- [ ] 进程池是否使用with语句管理
- [ ] 大数据是否避免序列化传输
- [ ] 子进程异常是否正确处理
- [ ] 是否正确设置进程启动方法
- [ ] 资源是否正确清理

### 8.3 asyncio检查清单

- [ ] 是否使用async/await语法
- [ ] 阻塞调用是否使用run_in_executor
- [ ] 异步资源是否使用async with管理
- [ ] 协程取消是否正确处理
- [ ] 异常是否正确传播
- [ ] 超时是否使用asyncio.wait_for
- [ ] 并发任务是否使用gather或TaskGroup

### 8.4 死锁预防检查清单

- [ ] 是否固定加锁顺序
- [ ] 是否使用超时获取锁
- [ ] 是否避免嵌套锁
- [ ] 锁是否使用with语句确保释放
- [ ] 是否避免在持锁时调用外部代码
- [ ] 是否最小化锁的持有时间
- [ ] 是否考虑使用无锁数据结构

### 8.5 concurrent.futures检查清单

- [ ] max_workers是否合理设置
- [ ] Future结果是否处理异常
- [ ] 是否使用as_completed处理结果
- [ ] 执行器是否使用with语句管理
- [ ] 任务取消是否正确处理
- [ ] 超时是否正确设置
- [ ] IO密集与CPU密集是否选择正确执行器

## 9. 最佳实践总结

1. **选择正确的并发模型：** IO密集用threading/asyncio，CPU密集用multiprocessing
2. **最小化共享状态：** 共享状态越少，并发问题越少
3. **使用高级抽象：** 优先使用Queue、Pool等高级抽象
4. **总是使用with语句：** 确保资源正确释放
5. **设置合理超时：** 避免无限等待
6. **正确处理异常：** 子线程/进程的异常要正确传播
7. **优雅关闭：** 实现优雅关闭机制，避免资源泄露
8. **测试并发代码：** 使用压力测试验证并发正确性
