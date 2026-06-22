# Python命名规范详细参考（PEP 8）

本文档详细描述Python代码的命名约定，遵循PEP 8标准，确保代码可读性和一致性。

---

## 1. 模块名（Module Names）

### 规则
- 全部使用小写字母
- 单词之间使用下划线分隔
- 名称应简短、有意义
- 避免使用下划线开头（除非是内部模块）

### 正确示例
```python
# 文件名
user_service.py       # 用户服务模块
http_client.py        # HTTP客户端模块
data_processor.py     # 数据处理器
database_utils.py     # 数据库工具
```

### 错误示例
```python
UserService.py        # 错误：使用了大写
http-client.py        # 错误：使用了连字符
user_service_v2.py    # 不推荐：版本号应在模块内定义
```

---

## 2. 包名（Package Names）

### 规则
- 全部使用小写字母
- 尽量简短
- 不使用下划线（推荐）
- 名称应具有描述性

### 正确示例
```python
# 目录结构
project/
├── utils/           # 工具包
├── services/        # 服务包
├── models/          # 模型包
├── core/            # 核心包
└── external/        # 外部接口包
```

### 错误示例
```python
user_services/       # 不推荐：应简化为userservices
utils_package/       # 错误：应简化为utils
HTTPClient/          # 错误：使用了大写
```

---

## 3. 类名（Class Names）

### 规则
- 使用PascalCase（大驼峰命名法）
- 每个单词首字母大写，无分隔符
- 异常类应以`Exception`结尾
- 抽象基类可使用`Base`前缀或`ABC`后缀

### 正确示例
```python
class UserService:                    # 用户服务类
    pass

class HttpRequest:                    # HTTP请求类
    pass

class DatabaseConnection:             # 数据库连接类
    pass

class ValidationError(Exception):     # 验证异常
    pass

class BaseHandler:                    # 抽象基类
    pass

class DataSourceABC:                  # 抽象基类（使用ABC后缀）
    pass
```

### 错误示例
```python
class user_service:       # 错误：应使用PascalCase
class User_Service:       # 错误：不应使用下划线
class userservice:        # 错误：难以阅读
```

---

## 4. 函数和方法名（Function and Method Names）

### 规则
- 使用snake_case（小写字母+下划线）
- 名称应以动词开头表示操作
- 布尔返回值函数使用`is_`、`has_`、`can_`前缀
- 私有方法使用单下划线前缀
- 保护方法使用双下划线前缀（名称修饰）

### 正确示例
```python
def get_user_by_id(user_id):           # 获取用户
    pass

def calculate_total_price(items):      # 计算总价
    pass

def is_valid_email(email):             # 布尔判断
    return bool(re.match(pattern, email))

def has_permission(user, action):      # 权限检查
    return action in user.permissions

def can_access_resource(user, resource):  # 能力检查
    return user.role == 'admin'

def _validate_input(data):             # 私有方法
    pass

def __internal_process(self):          # 保护方法（名称修饰）
    pass
```

### 错误示例
```python
def GetUserById(id):      # 错误：应使用snake_case
def get_user(Id):         # 错误：参数应使用snake_case
def valid_email(email):   # 不推荐：布尔函数应有is/has前缀
```

---

## 5. 变量名（Variable Names）

### 规则
- 使用snake_case
- 避免使用单字母（循环变量除外）
- 避免与Python内置函数名冲突
- 变量名应具有描述性

### 正确示例
```python
user_name = "Alice"              # 用户名
record_count = 100               # 记录数量
item_list = []                   # 物品列表
database_connection = None       # 数据库连接

# 循环变量允许单字母
for i in range(10):
    for item in items:
        process(item)

# 避免与内置名冲突
class_ = "User"                  # 使用后缀下划线避免关键字
type_ = "admin"                  # 使用后缀下划线避免内置函数
list_ = [1, 2, 3]                # 使用后缀下划线避免内置类型
```

### 错误示例
```python
UserName = "Alice"               # 错误：应使用snake_case
user_name_ = "Alice"             # 错误：不应使用后缀下划线（除非避免冲突）
list = [1, 2, 3]                 # 错误：与内置函数冲突
x = "Alice"                      # 不推荐：缺乏描述性
```

---

## 6. 常量（Constants）

### 规则
- 使用UPPER_SNAKE_CASE（全大写+下划线）
- 在模块级别定义
- 常量值不应在运行时修改

### 正确示例
```python
# 模块级常量
MAX_CONNECTIONS = 100
DEFAULT_TIMEOUT = 30
PI = 3.14159265359
DATABASE_URL = "postgresql://localhost:5432/db"
API_VERSION = "v1.0.0"
HTTP_STATUS_OK = 200
MAX_RETRY_COUNT = 3

class Config:
    DEBUG = False
    TESTING = False
    SECRET_KEY = "your-secret-key"
```

### 错误示例
```python
maxConnections = 100             # 错误：应使用UPPER_SNAKE_CASE
MAX_CONNECTIONS = 100            # 在函数内部定义常量（不推荐）
```

---

## 7. 私有属性（Private Attributes）

### 规则
- 使用单下划线前缀`_`表示内部使用
- 双下划线前缀`__`触发名称修饰
- 名称修饰用于避免子类命名冲突

### 单下划线前缀（约定私有）

```python
class UserService:
    def __init__(self):
        self._cache = {}         # 约定私有，外部可访问但不推荐
        self._initialized = False
    
    def _validate(self, data):   # 约定私有方法
        return True

service = UserService()
print(service._cache)            # 可以访问，但不推荐
```

### 双下划线前缀（名称修饰）

```python
class BaseService:
    def __init__(self):
        self.__internal_id = 1   # 实际名称：_BaseService__internal_id

class UserService(BaseService):
    def __init__(self):
        super().__init__()
        self.__internal_id = 2   # 实际名称：_UserService__internal_id
                                # 不会覆盖父类的同名属性

user = UserService()
# print(user.__internal_id)      # AttributeError
print(user._BaseService__internal_id)  # 1
print(user._UserService__internal_id)  # 2
```

---

## 8. 名称修饰（Name Mangling）

### 规则
- 双下划线前缀触发名称修饰
- 格式：`_ClassName__attribute`
- 主要用于避免继承中的名称冲突
- 不要滥用名称修饰，大多数情况使用单下划线即可

### 示例
```python
class Counter:
    def __init__(self):
        self.__count = 0         # 变为 _Counter__count
    
    def increment(self):
        self.__count += 1        # 变为 _Counter__count += 1
    
    def get_count(self):
        return self.__count

counter = Counter()
counter.increment()
print(counter._Counter__count)   # 1 - 通过修饰名访问
print(counter.get_count())       # 1 - 通过公共方法访问
```

---

## 9. 魔术方法（Magic Methods）

### 规则
- 使用双下划线包围`__method__`
- 也称为特殊方法或dunder方法
- 用于实现Python协议
- 不要自定义新的魔术方法

### 常用魔术方法

```python
class User:
    def __init__(self, name, age):      # 构造函数
        self.name = name
        self.age = age
    
    def __str__(self):                  # 字符串表示
        return f"User({self.name})"
    
    def __repr__(self):                 # 官方字符串表示
        return f"User(name='{self.name}', age={self.age})"
    
    def __eq__(self, other):            # 相等比较
        return self.name == other.name
    
    def __lt__(self, other):            # 小于比较
        return self.age < other.age
    
    def __len__(self):                  # 长度
        return len(self.name)
    
    def __getitem__(self, key):         # 索引访问
        return getattr(self, key)
    
    def __bool__(self):                 # 布尔值
        return self.age >= 18
    
    def __enter__(self):                # 上下文管理器
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        pass
```

---

## 10. 类型变量（Type Variables）

### 规则
- 使用PascalCase命名
- 通常使用简短的描述性名称
- 在`typing`模块中定义

### 示例
```python
from typing import TypeVar, Generic, List

# 简单类型变量
T = TypeVar('T')                    # 通用类型
K = TypeVar('K')                    # 键类型
V = TypeVar('V')                    # 值类型
T_co = TypeVar('T_co', covariant=True)    # 协变类型
T_contra = TypeVar('T_contra', contravariant=True)  # 逆变类型

# 约束类型变量
Number = TypeVar('Number', int, float)    # 必须是int或float

# 使用示例
class Stack(Generic[T]):
    def __init__(self):
        self._items: List[T] = []
    
    def push(self, item: T) -> None:
        self._items.append(item)
    
    def pop(self) -> T:
        return self._items.pop()
    
    def is_empty(self) -> bool:
        return len(self._items) == 0

stack: Stack[int] = Stack()
stack.push(1)
stack.push(2)
```

---

## 命名约定速查表

| 类型 | 命名风格 | 示例 |
|------|----------|------|
| 模块 | lower_case | `user_service.py` |
| 包 | lowercase | `services` |
| 类 | PascalCase | `UserService` |
| 函数 | snake_case | `get_user_by_id()` |
| 方法 | snake_case | `calculate_total()` |
| 变量 | snake_case | `user_name` |
| 常量 | UPPER_SNAKE_CASE | `MAX_CONNECTIONS` |
| 私有属性 | _snake_case | `_internal_cache` |
| 保护属性 | __snake_case | `__private_data` |
| 魔术方法 | __method__ | `__init__` |
| 类型变量 | PascalCase | `T`, `K`, `V` |

---

## 命名检查清单

### 模块和包
- [ ] 模块名是否全部小写？
- [ ] 模块名是否使用下划线分隔单词？
- [ ] 包名是否全部小写？
- [ ] 包名是否简短且无下划线？

### 类
- [ ] 类名是否使用PascalCase？
- [ ] 异常类是否以`Exception`结尾？
- [ ] 抽象基类是否使用`Base`前缀或`ABC`后缀？

### 函数和方法
- [ ] 函数名是否使用snake_case？
- [ ] 布尔函数是否使用`is_`、`has_`、`can_`前缀？
- [ ] 私有方法是否使用单下划线前缀？

### 变量
- [ ] 变量名是否使用snake_case？
- [ ] 是否避免了单字母变量名（循环除外）？
- [ ] 是否避免了与内置名冲突？
- [ ] 变量名是否具有描述性？

### 常量
- [ ] 常量是否使用UPPER_SNAKE_CASE？
- [ ] 常量是否在模块级别定义？

### 特殊命名
- [ ] 私有属性是否正确使用单下划线前缀？
- [ ] 是否正确理解双下划线名称修饰的作用？
- [ ] 是否正确使用魔术方法？
- [ ] 是否避免自定义新的魔术方法？

### 类型注解
- [ ] 类型变量是否使用PascalCase？
- [ ] 类型变量名称是否简短且有意义？

---

## 参考资料

- [PEP 8 -- Style Guide for Python Code](https://www.python.org/dev/peps/pep-0008/)
- [PEP 484 -- Type Hints](https://www.python.org/dev/peps/pep-0484/)
- [Python Naming Conventions](https://realpython.com/python-pep8/)
