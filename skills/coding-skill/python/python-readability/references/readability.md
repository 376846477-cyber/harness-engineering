# Python可读性规范详细参考

## 1. 名称传递信息

### 1.1 函数命名原则

函数名应清晰表达其行为和意图，遵循"动词+名词"结构。

```python
# 差：无法理解函数用途
def process(data):
    return data * 2

# 好：清晰表达行为和意图
def calculate_doubled_price(base_price: float) -> float:
    return base_price * 2

# 差：动词模糊
def handle_user(user_id):
    pass

# 好：动词具体
def deactivate_user_account(user_id: int) -> bool:
    pass
```

### 1.2 变量命名细节

变量名应包含重要信息，如单位、状态、类型。

```python
# 差：缺少上下文信息
timeout = 30
size = 1024
result = []

# 好：携带重要细节
timeout_seconds = 30
buffer_size_bytes = 1024
filtered_user_list = []

# 差：布尔变量命名不规范
flag = True
status = False

# 好：使用is/has/can前缀
is_authenticated = True
has_permission = False
can_delete = True
```

### 1.3 避免无意义名称

```python
# 差：无意义名称
data = get_users()
temp = process_items(items)
result = calculate(x, y)

# 好：描述性名称
user_list = get_users()
processed_items = process_items(items)
total_price = calculate(base_price, tax_rate)

# 差：单字母变量（除循环计数器）
x = get_data()
d = {}

# 好：有意义的名称
user_data = get_data()
user_age_map = {}
```

## 2. Pythonic代码风格

### 2.1 列表推导式

使用列表推导式替代显式循环append操作，使代码更简洁。

```python
# 差：传统循环方式
squares = []
for i in range(10):
    squares.append(i ** 2)

# 好：列表推导式
squares = [i ** 2 for i in range(10)]

# 差：带条件判断的循环
even_squares = []
for i in range(10):
    if i % 2 == 0:
        even_squares.append(i ** 2)

# 好：带条件的列表推导式
even_squares = [i ** 2 for i in range(10) if i % 2 == 0]
```

### 2.2 字典推导式

```python
# 差：传统方式构建字典
word_lengths = {}
for word in ['apple', 'banana', 'cherry']:
    word_lengths[word] = len(word)

# 好：字典推导式
word_lengths = {word: len(word) for word in ['apple', 'banana', 'cherry']}

# 差：带条件的字典构建
long_words = {}
for word in ['apple', 'banana', 'cherry']:
    if len(word) > 5:
        long_words[word] = len(word)

# 好：带条件的字典推导式
long_words = {word: len(word) for word in ['apple', 'banana', 'cherry'] if len(word) > 5}
```

### 2.3 生成器表达式

对于大数据集，使用生成器表达式节省内存。

```python
# 差：列表推导式占用内存
total = sum([i ** 2 for i in range(1000000)])

# 好：生成器表达式惰性计算
total = sum(i ** 2 for i in range(1000000))

# 差：创建中间列表
filtered_data = [x for x in large_dataset if x > 0]
result = sum(filtered_data)

# 好：链式生成器表达式
result = sum(x for x in large_dataset if x > 0)
```

### 2.4 集合推导式

```python
# 差：循环创建集合
unique_lengths = set()
for word in ['apple', 'banana', 'cherry']:
    unique_lengths.add(len(word))

# 好：集合推导式
unique_lengths = {len(word) for word in ['apple', 'banana', 'cherry']}
```

## 3. 解包操作

### 3.1 基本解包

```python
# 差：使用索引访问
point = (10, 20)
x = point[0]
y = point[1]

# 好：元组解包
point = (10, 20)
x, y = point

# 差：多变量赋值
a = 1
b = 2
c = 3

# 好：链式赋值
a, b, c = 1, 2, 3
```

### 3.2 扩展解包

```python
# 差：复杂的索引操作
data = [1, 2, 3, 4, 5]
first = data[0]
rest = data[1:]

# 好：星号解包
data = [1, 2, 3, 4, 5]
first, *rest = data

# 好：解包首尾元素
data = [1, 2, 3, 4, 5]
first, *middle, last = data
```

### 3.3 字典解包合并

```python
# 差：显式复制合并
default_config = {'host': 'localhost', 'port': 8080}
user_config = {'port': 9090, 'debug': True}
config = default_config.copy()
config.update(user_config)

# 好：字典解包合并
default_config = {'host': 'localhost', 'port': 8080}
user_config = {'port': 9090, 'debug': True}
config = {**default_config, **user_config}
```

### 3.4 函数参数解包

```python
# 差：逐个传递参数
point = (10, 20)
result = calculate_distance(point[0], point[1])

# 好：解包传递
point = (10, 20)
result = calculate_distance(*point)

# 差：逐个传递关键字参数
options = {'verbose': True, 'timeout': 30}
result = run_task(verbose=options['verbose'], timeout=options['timeout'])

# 好：字典解包传递关键字参数
options = {'verbose': True, 'timeout': 30}
result = run_task(**options)
```

## 4. 上下文管理器

### 4.1 文件操作

```python
# 差：手动关闭文件
f = open('data.txt', 'r')
content = f.read()
f.close()

# 好：使用with自动关闭
with open('data.txt', 'r') as f:
    content = f.read()

# 差：多个文件嵌套
f1 = open('input.txt', 'r')
f2 = open('output.txt', 'w')
f1.close()
f2.close()

# 好：多个上下文管理器
with open('input.txt', 'r') as f1, open('output.txt', 'w') as f2:
    data = f1.read()
    f2.write(data)
```

### 4.2 自定义上下文管理器

```python
from contextlib import contextmanager

# 差：手动管理资源
import time
start = time.time()
try:
    result = expensive_operation()
finally:
    elapsed = time.time() - start
    print(f'耗时: {elapsed}秒')

# 好：使用上下文管理器
@contextmanager
def timer():
    start = time.time()
    yield
    elapsed = time.time() - start
    print(f'耗时: {elapsed}秒')

with timer():
    result = expensive_operation()
```

### 4.3 锁资源管理

```python
import threading

# 差：手动管理锁
lock = threading.Lock()
lock.acquire()
try:
    shared_resource.update()
finally:
    lock.release()

# 好：使用with管理锁
lock = threading.Lock()
with lock:
    shared_resource.update()
```

## 5. 函数长度控制

### 5.1 单一职责原则

每个函数应只做一件事，保持简短（建议不超过20行）。

```python
# 差：函数过长，职责不清
def process_user_data(data):
    # 验证数据
    if not data or 'email' not in data:
        raise ValueError('Invalid data')
    if '@' not in data['email']:
        raise ValueError('Invalid email')
    
    # 转换数据
    data['email'] = data['email'].lower()
    data['name'] = data['name'].strip().title()
    
    # 保存到数据库
    db.connect()
    db.insert('users', data)
    db.close()
    
    # 发送邮件
    send_email(data['email'], 'Welcome!')
    
    return data

# 好：拆分为多个小函数
def validate_user_data(data: dict) -> None:
    if not data or 'email' not in data:
        raise ValueError('Invalid data')
    if '@' not in data['email']:
        raise ValueError('Invalid email')

def normalize_user_data(data: dict) -> dict:
    return {
        'email': data['email'].lower(),
        'name': data['name'].strip().title()
    }

def save_user(data: dict) -> None:
    with db.connection():
        db.insert('users', data)

def process_user_data(data: dict) -> dict:
    validate_user_data(data)
    normalized = normalize_user_data(data)
    save_user(normalized)
    send_welcome_email(normalized['email'])
    return normalized
```

### 5.2 提取辅助函数

```python
# 差：复杂逻辑内联
def calculate_discount(user, order):
    if user.is_premium:
        if order.total > 1000:
            return order.total * 0.2
        else:
            return order.total * 0.1
    else:
        if user.age > 60:
            return order.total * 0.15
        else:
            return 0

# 好：提取辅助函数
def get_premium_discount_rate(order_total: float) -> float:
    return 0.2 if order_total > 1000 else 0.1

def get_senior_discount_rate() -> float:
    return 0.15

def calculate_discount(user, order) -> float:
    if user.is_premium:
        rate = get_premium_discount_rate(order.total)
    elif user.age > 60:
        rate = get_senior_discount_rate()
    else:
        rate = 0
    return order.total * rate
```

## 6. 减少嵌套（早返回）

### 6.1 卫语句模式

使用早返回减少嵌套层级，提高可读性。

```python
# 差：深层嵌套
def process_payment(payment):
    if payment is not None:
        if payment.is_valid():
            if payment.amount > 0:
                if payment.has_sufficient_funds():
                    return process_transaction(payment)
                else:
                    return 'Insufficient funds'
            else:
                return 'Invalid amount'
        else:
            return 'Invalid payment'
    else:
        return 'No payment provided'

# 好：早返回（卫语句）
def process_payment(payment):
    if payment is None:
        return 'No payment provided'
    
    if not payment.is_valid():
        return 'Invalid payment'
    
    if payment.amount <= 0:
        return 'Invalid amount'
    
    if not payment.has_sufficient_funds():
        return 'Insufficient funds'
    
    return process_transaction(payment)
```

### 6.2 循环中的早返回

```python
# 差：使用标志变量
def find_user_by_id(user_list, target_id):
    found_user = None
    for user in user_list:
        if user.id == target_id:
            found_user = user
            break
    return found_user

# 好：直接返回
def find_user_by_id(user_list, target_id):
    for user in user_list:
        if user.id == target_id:
            return user
    return None

# 差：嵌套的条件判断
def has_admin(users):
    result = False
    for user in users:
        if user.role == 'admin':
            if user.is_active:
                result = True
                break
    return result

# 好：使用any()和早返回
def has_admin(users):
    return any(user.role == 'admin' and user.is_active for user in users)
```

### 6.3 continue跳过不需要的迭代

```python
# 差：嵌套的条件处理
def process_items(items):
    results = []
    for item in items:
        if item.is_valid():
            if item.is_ready():
                results.append(item.process())
    return results

# 好：使用continue跳过
def process_items(items):
    results = []
    for item in items:
        if not item.is_valid():
            continue
        if not item.is_ready():
            continue
        results.append(item.process())
    return results
```

## 7. 避免魔法数字

### 7.1 使用命名常量

```python
# 差：魔法数字
def calculate_discount(price):
    if price > 1000:
        return price * 0.15
    elif price > 500:
        return price * 0.1
    return 0

def get_timeout():
    return 30

# 好：命名常量
PREMIUM_THRESHOLD = 1000
STANDARD_THRESHOLD = 500
PREMIUM_DISCOUNT_RATE = 0.15
STANDARD_DISCOUNT_RATE = 0.1
DEFAULT_TIMEOUT_SECONDS = 30

def calculate_discount(price):
    if price > PREMIUM_THRESHOLD:
        return price * PREMIUM_DISCOUNT_RATE
    elif price > STANDARD_THRESHOLD:
        return price * STANDARD_DISCOUNT_RATE
    return 0

def get_timeout():
    return DEFAULT_TIMEOUT_SECONDS
```

### 7.2 使用枚举

```python
from enum import Enum

# 差：使用数字常量
STATUS_PENDING = 0
STATUS_APPROVED = 1
STATUS_REJECTED = 2

def get_status_message(status):
    if status == 0:
        return 'Pending'
    elif status == 1:
        return 'Approved'
    elif status == 2:
        return 'Rejected'

# 好：使用枚举
class Status(Enum):
    PENDING = 'pending'
    APPROVED = 'approved'
    REJECTED = 'rejected'

def get_status_message(status: Status) -> str:
    messages = {
        Status.PENDING: 'Pending',
        Status.APPROVED: 'Approved',
        Status.REJECTED: 'Rejected'
    }
    return messages.get(status, 'Unknown')
```

### 7.3 配置文件集中管理

```python
# 差：散落在代码中的配置
def connect_database():
    return connect(host='localhost', port=5432, timeout=30)

def connect_cache():
    return connect(host='localhost', port=6379, timeout=10)

# 好：集中配置管理
class Config:
    DATABASE_HOST = 'localhost'
    DATABASE_PORT = 5432
    DATABASE_TIMEOUT_SECONDS = 30
    
    CACHE_HOST = 'localhost'
    CACHE_PORT = 6379
    CACHE_TIMEOUT_SECONDS = 10

def connect_database():
    return connect(
        host=Config.DATABASE_HOST,
        port=Config.DATABASE_PORT,
        timeout=Config.DATABASE_TIMEOUT_SECONDS
    )

def connect_cache():
    return connect(
        host=Config.CACHE_HOST,
        port=Config.CACHE_PORT,
        timeout=Config.CACHE_TIMEOUT_SECONDS
    )
```

## 8. 使用enumerate/zip

### 8.1 enumerate替代range(len())

```python
# 差：使用range(len())
items = ['apple', 'banana', 'cherry']
for i in range(len(items)):
    print(f'{i}: {items[i]}')

# 好：使用enumerate
items = ['apple', 'banana', 'cherry']
for index, item in enumerate(items):
    print(f'{index}: {item}')

# 好：指定起始索引
for index, item in enumerate(items, start=1):
    print(f'{index}: {item}')

# 差：需要索引修改元素
numbers = [1, 2, 3, 4, 5]
for i in range(len(numbers)):
    numbers[i] = numbers[i] * 2

# 好：enumerate更清晰
numbers = [1, 2, 3, 4, 5]
for i, num in enumerate(numbers):
    numbers[i] = num * 2
```

### 8.2 zip并行迭代

```python
# 差：使用索引并行迭代
names = ['Alice', 'Bob', 'Charlie']
ages = [25, 30, 35]
for i in range(len(names)):
    print(f'{names[i]} is {ages[i]} years old')

# 好：使用zip
names = ['Alice', 'Bob', 'Charlie']
ages = [25, 30, 35]
for name, age in zip(names, ages):
    print(f'{name} is {age} years old')

# 差：创建字典
keys = ['name', 'age', 'city']
values = ['Alice', 25, 'Beijing']
result = {}
for i in range(len(keys)):
    result[keys[i]] = values[i]

# 好：使用zip创建字典
keys = ['name', 'age', 'city']
values = ['Alice', 25, 'Beijing']
result = dict(zip(keys, values))

# 好：多个序列并行迭代
names = ['Alice', 'Bob', 'Charlie']
ages = [25, 30, 35]
cities = ['Beijing', 'Shanghai', 'Shenzhen']
for name, age, city in zip(names, ages, cities):
    print(f'{name}, {age}, lives in {city}')

# 注意：zip会在最短序列结束时停止
short = [1, 2]
long = [10, 20, 30, 40]
list(zip(short, long))  # [(1, 10), (2, 20)]

# 使用zip_longest填充缺失值
from itertools import zip_longest
list(zip_longest(short, long, fillvalue=0))  # [(1, 10), (2, 20), (0, 30), (0, 40)]
```

## 9. 可读性检查清单

### 9.1 命名检查

- [ ] 所有变量名是否描述了其用途？
- [ ] 函数名是否使用了动词+名词结构？
- [ ] 布尔变量是否使用is/has/can前缀？
- [ ] 是否避免了缩写和单字母变量（循环计数器除外）？
- [ ] 常量是否使用全大写下划线命名？

### 9.2 Pythonic检查

- [ ] 是否使用列表/字典推导式替代显式循环？
- [ ] 大数据集是否使用生成器表达式？
- [ ] 是否使用enumerate替代range(len())？
- [ ] 是否使用zip并行迭代多个序列？
- [ ] 是否使用解包替代索引访问？
- [ ] 是否使用f-string格式化字符串？

### 9.3 结构检查

- [ ] 函数是否保持简短（<20行）？
- [ ] 每个函数是否只做一件事？
- [ ] 是否使用早返回减少嵌套？
- [ ] 嵌套层级是否不超过3层？
- [ ] 是否避免了魔法数字？

### 9.4 资源管理检查

- [ ] 文件操作是否使用with语句？
- [ ] 锁资源是否使用上下文管理器？
- [ ] 数据库连接是否正确关闭？
- [ ] 临时资源是否被正确清理？

### 9.5 代码审查示例

```python
# 审查前
def proc(d):
    r = []
    for i in range(len(d)):
        if d[i] > 0:
            if d[i] < 100:
                r.append(d[i] * 2)
    return r

# 审查后
def filter_and_double_positive_small_values(numbers: list[int]) -> list[int]:
    return [num * 2 for num in numbers if 0 < num < 100]
```
