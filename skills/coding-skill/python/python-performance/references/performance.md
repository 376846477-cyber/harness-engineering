# Python高效性规范详细参考

## 1. 算法选择和数据结构

### 1.1 选择合适的数据结构

```python
# 差：使用列表进行成员检查 O(n)
items = [1, 2, 3, 4, 5]
if item in items:  # 线性查找
    pass

# 好：使用集合进行成员检查 O(1)
items = {1, 2, 3, 4, 5}
if item in items:  # 常数时间查找
    pass

# 差：使用列表查找键值对 O(n)
pairs = [('a', 1), ('b', 2)]
for key, value in pairs:
    if key == 'a':
        return value

# 好：使用字典 O(1)
mapping = {'a': 1, 'b': 2}
value = mapping.get('a')
```

### 1.2 标准库数据结构

```python
from collections import deque, defaultdict, Counter
from heapq import heappush, heappop, nlargest
import bisect

# deque：双端队列，两端操作都是O(1)
queue = deque([1, 2, 3])
queue.appendleft(0)  # 左端添加 O(1)
queue.pop()           # 右端弹出 O(1)

# defaultdict：避免键不存在异常
from collections import defaultdict
word_counts = defaultdict(int)
for word in words:
    word_counts[word] += 1

# Counter：计数器
from collections import Counter
counts = Counter(['a', 'b', 'a', 'c', 'a'])
# Counter({'a': 3, 'b': 1, 'c': 1})

# heapq：堆排序
import heapq
data = [3, 1, 4, 1, 5]
heapq.heapify(data)  # 原地建堆
smallest = heapq.nsmallest(3, data)

# bisect：二分查找（有序列表）
import bisect
sorted_list = [1, 3, 5, 7, 9]
index = bisect.bisect_left(sorted_list, 5)  # O(log n)
```

---

## 2. 生成器和迭代器

### 2.1 避免创建大列表

```python
# 差：创建大列表占用大量内存
def process_large_file(filename):
    with open(filename) as f:
        lines = f.readlines()  # 一次性加载所有行
    for line in lines:
        process(line)

# 好：使用生成器逐行处理
def process_large_file(filename):
    with open(filename) as f:
        for line in f:  # 惰性迭代
            process(line)
```

### 2.2 生成器函数

```python
# 差：返回完整列表
def fibonacci_list(n):
    result = []
    a, b = 0, 1
    for _ in range(n):
        result.append(a)
        a, b = b, a + b
    return result

# 好：使用生成器
def fibonacci_gen(n):
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

# 使用生成器表达式
sum_of_squares = sum(x * x for x in range(1000000))  # 不创建中间列表
```

### 2.3 itertools模块

```python
from itertools import islice, chain, cycle, repeat, product, permutations

# islice：切片迭代器
first_10 = islice(range(1000), 10)

# chain：连接多个迭代器
combined = chain([1, 2], [3, 4], [5, 6])

# cycle：无限循环
for i, item in enumerate(cycle(['A', 'B', 'C'])):
    if i >= 10:
        break

# product：笛卡尔积
for pair in product([1, 2], ['a', 'b']):
    print(pair)  # (1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')
```

---

## 3. 列表推导式 vs 循环

### 3.1 列表推导式通常更快

```python
# 差：传统循环
squares = []
for x in range(1000):
    squares.append(x * x)

# 好：列表推导式
squares = [x * x for x in range(1000)]

# 好：生成器表达式（内存更优）
squares_gen = (x * x for x in range(1000))

# 字典推导式
squared_dict = {x: x * x for x in range(10)}

# 集合推导式
unique_lengths = {len(word) for word in words}
```

### 3.2 复杂逻辑保持可读性

```python
# 差：过于复杂的推导式
result = [process(item) for sublist in data if sublist 
          for item in sublist if item is not None if validate(item)]

# 好：保持可读性，使用普通循环
result = []
for sublist in data:
    if not sublist:
        continue
    for item in sublist:
        if item is not None and validate(item):
            result.append(process(item))
```

---

## 4. 使用内置函数和标准库

### 4.1 内置函数比自定义循环快

```python
data = [1, 2, 3, 4, 5]

# 差：手动循环求和
total = 0
for x in data:
    total += x

# 好：使用内置sum
total = sum(data)

# 差：手动查找最大值
max_val = data[0]
for x in data:
    if x > max_val:
        max_val = x

# 好：使用内置max
max_val = max(data)

# 使用map和filter
numbers = range(10)
evens = list(filter(lambda x: x % 2 == 0, numbers))
squares = list(map(lambda x: x * x, numbers))

# 更好：使用推导式（通常更清晰）
evens = [x for x in numbers if x % 2 == 0]
squares = [x * x for x in numbers]
```

### 4.2 使用operator模块

```python
from operator import itemgetter, attrgetter, methodcaller

# itemgetter：获取字典字段
people = [{'name': 'Alice', 'age': 30}, {'name': 'Bob', 'age': 25}]
sorted_people = sorted(people, key=itemgetter('age'))
names = map(itemgetter('name'), people)

# attrgetter：获取对象属性
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

people = [Person('Alice', 30), Person('Bob', 25)]
sorted_people = sorted(people, key=attrgetter('age'))

# methodcaller：调用方法
strings = ['hello', 'world']
upper_strings = list(map(methodcaller('upper'), strings))
```

---

## 5. 避免全局变量

### 5.1 局部变量访问更快

```python
# 差：使用全局变量
total = 0

def accumulate(numbers):
    global total
    for n in numbers:
        total += n

# 好：使用局部变量和返回值
def accumulate(numbers):
    total = 0
    for n in numbers:
        total += n
    return total

# 好：函数内部缓存常用全局引用
import math

def compute_angles(degrees):
    # 将math.radians缓存为局部变量
    radians_func = math.radians
    sin_func = math.sin
    return [sin_func(radians_func(d)) for d in degrees]
```

---

## 6. 字符串拼接

### 6.1 使用join而非+

```python
# 差：使用+拼接
result = ''
for word in words:
    result += word + ' '

# 好：使用join
result = ' '.join(words)

# 差：循环中拼接字符串
html = ''
for item in items:
    html += '<li>' + item + '</li>'

# 好：收集后join
parts = ['<li>' + item + '</li>' for item in items]
html = ''.join(parts)

# 更好：使用io.StringIO
from io import StringIO
buffer = StringIO()
for item in items:
    buffer.write(f'<li>{item}</li>')
html = buffer.getvalue()
```

---

## 7. 使用functools.lru_cache缓存

### 7.1 函数结果缓存

```python
from functools import lru_cache

# 递归函数缓存
@lru_cache(maxsize=None)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# 带参数限制的缓存
@lru_cache(maxsize=128)
def expensive_computation(n, m):
    return n ** m

# 清除缓存
fibonacci.cache_clear()

# 查看缓存信息
print(fibonacci.cache_info())

# cached_property：属性缓存
from functools import cached_property

class DataSet:
    def __init__(self, data):
        self.data = data
    
    @cached_property
    def statistics(self):
        # 只计算一次，之后缓存
        return {
            'mean': sum(self.data) / len(self.data),
            'max': max(self.data),
            'min': min(self.data)
        }
```

---

## 8. 异步编程（asyncio）

### 8.1 IO密集型任务使用异步

```python
import asyncio
import aiohttp

# 异步HTTP请求
async def fetch(session, url):
    async with session.get(url) as response:
        return await response.text()

async def fetch_all(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)

# 运行异步任务
urls = ['http://example.com'] * 10
results = asyncio.run(fetch_all(urls))
```

### 8.2 异步数据库操作

```python
import asyncio
import aiomysql

async def query_database():
    conn = await aiomysql.connect(
        host='localhost',
        user='user',
        password='password',
        db='database'
    )
    async with conn.cursor() as cur:
        await cur.execute("SELECT * FROM users")
        result = await cur.fetchall()
    conn.close()
    return result

# 并发执行多个查询
async def parallel_queries():
    results = await asyncio.gather(
        query_users(),
        query_orders(),
        query_products()
    )
    return results
```

---

## 9. 多进程 vs 多线程（GIL影响）

### 9.1 GIL限制

```python
# Python的GIL（全局解释器锁）限制了多线程的并行执行
# CPU密集型任务：使用多进程
# IO密集型任务：使用多线程或异步

import threading
import multiprocessing
import time

# CPU密集型任务
def cpu_bound(n):
    return sum(i * i for i in range(n))

# 多线程：对CPU密集型任务无提升（GIL限制）
def threaded_cpu_bound():
    threads = []
    for _ in range(4):
        t = threading.Thread(target=cpu_bound, args=(1000000,))
        threads.append(t)
        t.start()
    for t in threads:
        t.join()

# 多进程：真正并行
def multiprocess_cpu_bound():
    processes = []
    for _ in range(4):
        p = multiprocessing.Process(target=cpu_bound, args=(1000000,))
        processes.append(p)
        p.start()
    for p in processes:
        p.join()
```

### 9.2 使用ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor

def cpu_intensive_task(n):
    return sum(i * i for i in range(n))

# CPU密集型：使用进程池
with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(cpu_intensive_task, range(10)))

# IO密集型：使用线程池
def io_intensive_task(url):
    import requests
    return requests.get(url).text

with ThreadPoolExecutor(max_workers=10) as executor:
    results = list(executor.map(io_intensive_task, urls))
```

---

## 10. 性能分析工具

### 10.1 cProfile性能分析

```python
import cProfile
import pstats

# 方式1：直接运行
cProfile.run('my_function()')

# 方式2：命令行
# python -m cProfile -s cumtime my_script.py

# 方式3：程序化分析
profiler = cProfile.Profile()
profiler.enable()

# 运行代码
my_function()

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)  # 打印前20个最耗时的函数
```

### 10.2 timeit精确计时

```python
import timeit

# 测试单条语句
time = timeit.timeit('"-".join(str(n) for n in range(100))', number=10000)
print(f"Time: {time:.4f}s")

# 测试函数
def test_function():
    return [x * x for x in range(1000)]

time = timeit.timeit(test_function, number=1000)

# 使用lambda传递参数
def add(a, b):
    return a + b

time = timeit.timeit(lambda: add(1, 2), number=1000000)
```

### 10.3 memory_profiler内存分析

```python
# pip install memory-profiler

from memory_profiler import profile

@profile
def my_function():
    a = [1] * (10 ** 6)
    b = [2] * (2 * 10 ** 7)
    del b
    return a

# 运行: python -m memory_profiler my_script.py
```

---

## 11. 使用numpy/pandas处理大数据

### 11.1 NumPy向量化操作

```python
import numpy as np

# 差：Python循环
def python_sum(n):
    total = 0
    for i in range(n):
        total += i * i
    return total

# 好：NumPy向量化
def numpy_sum(n):
    arr = np.arange(n)
    return np.sum(arr * arr)

# 向量化操作示例
data = np.random.rand(1000000)

# 批量操作
mean = np.mean(data)
std = np.std(data)
normalized = (data - mean) / std

# 条件筛选
filtered = data[data > 0.5]

# 矩阵运算
matrix_a = np.random.rand(1000, 1000)
matrix_b = np.random.rand(1000, 1000)
result = np.dot(matrix_a, matrix_b)  # 矩阵乘法
```

### 11.2 Pandas高效数据处理

```python
import pandas as pd

# 读取大数据
df = pd.read_csv('large_file.csv', chunksize=10000)
for chunk in df:
    process(chunk)

# 向量化操作
df['new_column'] = df['col1'] + df['col2']

# 分组聚合
result = df.groupby('category')['value'].agg(['mean', 'sum', 'count'])

# 避免逐行操作
# 差：apply
df['processed'] = df['text'].apply(lambda x: process(x))

# 好：向量化（如果可能）
df['length'] = df['text'].str.len()

# 使用query过滤
filtered = df.query('value > 100 and category == "A"')
```

---

## 12. Cython加速

### 12.1 Cython基础

```python
# my_module.pyx
def sum_squares(int n):
    cdef int i
    cdef long total = 0
    for i in range(n):
        total += i * i
    return total

# setup.py
from setuptools import setup
from Cython.Build import cythonize

setup(
    ext_modules=cythonize("my_module.pyx")
)

# 编译: python setup.py build_ext --inplace
```

### 12.2 类型声明加速

```python
# 普通Python（慢）
def python_function(n):
    result = 0
    for i in range(n):
        result += i ** 2
    return result

# Cython优化版本
# cython_module.pyx
def cython_function(int n):
    cdef int i
    cdef long result = 0
    for i in range(n):
        result += i * i
    return result

# 使用cdef定义C级别函数（更快）
cdef long _fast_sum(int n):
    cdef int i
    cdef long result = 0
    for i in range(n):
        result += i * i
    return result

def fast_sum(n):
    return _fast_sum(n)
```

---

## 性能优化检查清单

### 数据结构
- [ ] 使用set/dict进行O(1)查找
- [ ] 使用deque处理双端队列
- [ ] 使用heapq进行堆操作
- [ ] 使用bisect进行有序列表查找

### 内存优化
- [ ] 使用生成器替代大列表
- [ ] 使用生成器表达式替代列表推导
- [ ] 逐行读取大文件
- [ ] 使用__slots__减少类内存占用

### 循环优化
- [ ] 使用列表推导式替代append循环
- [ ] 将全局变量缓存到局部
- [ ] 将函数调用移出循环
- [ ] 避免循环内的重复计算

### 字符串处理
- [ ] 使用join替代+拼接
- [ ] 使用StringIO处理大量字符串
- [ ] 使用f-string格式化（Python 3.6+）

### 函数优化
- [ ] 使用@lru_cache缓存函数结果
- [ ] 使用@cached_property缓存属性
- [ ] 避免默认可变参数

### 并发优化
- [ ] IO密集型使用asyncio或多线程
- [ ] CPU密集型使用多进程
- [ ] 使用ProcessPoolExecutor/ThreadPoolExecutor

### 第三方库
- [ ] 使用NumPy进行数值计算
- [ ] 使用Pandas进行数据处理
- [ ] 考虑Cython加速热点代码

### 分析工具
- [ ] 使用cProfile定位瓶颈
- [ ] 使用timeit精确测量
- [ ] 使用memory_profiler分析内存
