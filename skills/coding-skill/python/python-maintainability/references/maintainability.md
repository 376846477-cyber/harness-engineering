# Python可维护性规范详细参考

## 1. 模块化设计

### 1.1 模块划分原则

每个模块应专注于单一职责，提供清晰的功能边界：

```python
# 推荐：按功能划分模块
myproject/
├── auth/           # 认证模块
│   ├── __init__.py
│   ├── jwt_handler.py
│   └── password_hasher.py
├── database/       # 数据库模块
│   ├── __init__.py
│   ├── connection.py
│   └── models.py
└── utils/          # 工具模块
    ├── __init__.py
    └── validators.py

# 不推荐：按类型划分（导致高耦合）
myproject/
├── models/         # 所有模型混在一起
├── views/          # 所有视图混在一起
└── controllers/    # 所有控制器混在一起
```

### 1.2 模块导入规范

```python
# 推荐：使用绝对导入
from myproject.auth.jwt_handler import create_token
from myproject.database.connection import get_connection

# 不推荐：使用相对导入（增加理解难度）
from .jwt_handler import create_token
from ..database.connection import get_connection
```

## 2. 包结构组织

### 2.1 标准包结构

```python
myproject/
├── pyproject.toml      # 项目配置
├── README.md
├── src/
│   └── myproject/
│       ├── __init__.py
│       ├── core/           # 核心功能
│       │   ├── __init__.py
│       │   └── config.py
│       ├── api/            # API层
│       │   ├── __init__.py
│       │   └── routes.py
│       └── tests/          # 测试
│           ├── __init__.py
│           └── test_api.py
└── config/
    ├── dev.yaml
    └── prod.yaml
```

### 2.2 循环导入解决方案

```python
# 问题：循环导入
# module_a.py
from module_b import process_b

def process_a():
    return process_b()

# module_b.py
from module_a import process_a  # 循环导入！

def process_b():
    return process_a()

# 解决方案1：延迟导入
# module_a.py
def process_a():
    from module_b import process_b  # 函数内导入
    return process_b()

# 解决方案2：重构依赖结构
# shared.py
class DataProcessor:
    pass

# module_a.py
from shared import DataProcessor

# module_b.py
from shared import DataProcessor
```

## 3. SOLID原则

### 3.1 单一职责原则（SRP）

```python
# 违反SRP：一个类做太多事情
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
    
    def save(self):
        # 保存到数据库
        pass
    
    def send_email(self, message):
        # 发送邮件
        pass
    
    def validate(self):
        # 验证数据
        pass

# 遵循SRP：职责分离
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

class UserRepository:
    def save(self, user: User) -> None:
        # 保存到数据库
        pass

class EmailService:
    def send(self, email: str, message: str) -> None:
        # 发送邮件
        pass

class UserValidator:
    def validate(self, user: User) -> bool:
        # 验证数据
        return bool(user.name and user.email)
```

### 3.2 开闭原则（OCP）

```python
# 违反OCP：每次新增类型都要修改类
class PaymentProcessor:
    def process(self, payment_type, amount):
        if payment_type == "credit":
            self._process_credit(amount)
        elif payment_type == "paypal":
            self._process_paypal(amount)

# 遵循OCP：使用策略模式
from abc import ABC, abstractmethod

class PaymentStrategy(ABC):
    @abstractmethod
    def process(self, amount: float) -> None:
        pass

class CreditPayment(PaymentStrategy):
    def process(self, amount: float) -> None:
        print(f"Processing credit payment: ${amount}")

class PayPalPayment(PaymentStrategy):
    def process(self, amount: float) -> None:
        print(f"Processing PayPal payment: ${amount}")

class PaymentProcessor:
    def __init__(self, strategy: PaymentStrategy):
        self.strategy = strategy
    
    def process(self, amount: float) -> None:
        self.strategy.process(amount)
```

### 3.3 里氏替换原则（LSP）

```python
# 违反LSP：子类改变了父类行为
class Bird:
    def fly(self):
        pass

class Penguin(Bird):
    def fly(self):
        raise Exception("Penguins can't fly!")

# 遵循LSP：合理设计继承层次
class Bird:
    pass

class FlyingBird(Bird):
    def fly(self):
        pass

class Penguin(Bird):
    def swim(self):
        pass
```

### 3.4 接口隔离原则（ISP）

```python
# 违反ISP：接口太大
class Worker(ABC):
    @abstractmethod
    def work(self):
        pass
    
    @abstractmethod
    def eat(self):
        pass

class Robot(Worker):
    def work(self):
        print("Working")
    
    def eat(self):
        pass  # 机器人不需要吃饭

# 遵循ISP：接口分离
class Workable(ABC):
    @abstractmethod
    def work(self):
        pass

class Feedable(ABC):
    @abstractmethod
    def eat(self):
        pass

class Human(Workable, Feedable):
    def work(self):
        print("Working")
    
    def eat(self):
        print("Eating")

class Robot(Workable):
    def work(self):
        print("Working")
```

### 3.5 依赖倒置原则（DIP）

```python
# 违反DIP：高层模块依赖低层模块
class Switch:
    def __init__(self):
        self.light = LightBulb()  # 直接依赖具体实现
    
    def turn_on(self):
        self.light.on()

# 遵循DIP：依赖抽象
class Switchable(ABC):
    @abstractmethod
    def on(self):
        pass
    
    @abstractmethod
    def off(self):
        pass

class LightBulb(Switchable):
    def on(self):
        print("Light is on")
    
    def off(self):
        print("Light is off")

class Switch:
    def __init__(self, device: Switchable):
        self.device = device
    
    def turn_on(self):
        self.device.on()
```

## 4. 依赖注入

### 4.1 构造函数注入

```python
from typing import Protocol

class DatabaseProtocol(Protocol):
    def query(self, sql: str) -> list:
        ...

class UserRepository:
    def __init__(self, db: DatabaseProtocol):
        self.db = db  # 依赖注入
    
    def get_user(self, user_id: int) -> dict:
        return self.db.query(f"SELECT * FROM users WHERE id = {user_id}")

# 使用
class MySQLDatabase:
    def query(self, sql: str) -> list:
        return [{"id": 1, "name": "Alice"}]

db = MySQLDatabase()
repo = UserRepository(db)
```

### 4.2 使用依赖注入框架

```python
# 使用 dependency-injector 库
from dependency_injector import containers, providers

class Container(containers.DeclarativeContainer):
    config = providers.Configuration()
    
    database = providers.Singleton(
        MySQLDatabase,
        host=config.db.host,
        port=config.db.port
    )
    
    user_repository = providers.Factory(
        UserRepository,
        db=database
    )

# 使用
container = Container()
repo = container.user_repository()
```

## 5. 避免全局变量

### 5.1 问题示例

```python
# 不推荐：使用全局变量
cache = {}

def add_to_cache(key, value):
    cache[key] = value

def get_from_cache(key):
    return cache.get(key)
```

### 5.2 解决方案

```python
# 推荐：使用类封装状态
class Cache:
    def __init__(self):
        self._data = {}
    
    def add(self, key: str, value: any) -> None:
        self._data[key] = value
    
    def get(self, key: str, default=None):
        return self._data.get(key, default)

# 推荐：使用上下文管理器
from contextlib import contextmanager

@contextmanager
def temporary_cache():
    cache = {}
    yield cache
    # 自动清理

# 推荐：使用单例模式（谨慎使用）
class CacheSingleton:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._data = {}
        return cls._instance
```

## 6. 使用dataclass

### 6.1 基本用法

```python
from dataclasses import dataclass, field
from typing import List, Optional

@dataclass
class User:
    name: str
    age: int
    email: Optional[str] = None
    tags: List[str] = field(default_factory=list)
    
    def __post_init__(self):
        if self.age < 0:
            raise ValueError("Age cannot be negative")

# 自动生成 __init__, __repr__, __eq__
user = User(name="Alice", age=30)
print(user)  # User(name='Alice', age=30, email=None, tags=[])
```

### 6.2 不可变数据类

```python
@dataclass(frozen=True)
class Point:
    x: float
    y: float
    
    def distance_from_origin(self) -> float:
        return (self.x ** 2 + self.y ** 2) ** 0.5

point = Point(3.0, 4.0)
# point.x = 5.0  # 会抛出 FrozenInstanceError
```

### 6.3 继承

```python
@dataclass
class Animal:
    name: str
    age: int

@dataclass
class Dog(Animal):
    breed: str
    age: int = 0  # 可以覆盖父类字段

dog = Dog(name="Buddy", age=5, breed="Golden Retriever")
```

## 7. 类型注解

### 7.1 基本类型注解

```python
from typing import List, Dict, Optional, Union, Any

def process_data(
    items: List[str],
    config: Dict[str, Any],
    timeout: Optional[int] = None
) -> Dict[str, Union[int, str]]:
    result = {}
    for item in items:
        result[item] = len(item)
    return result

# 类型别名
UserId = int
UserName = str
UserDict = Dict[UserId, UserName]

def get_user_name(users: UserDict, user_id: UserId) -> Optional[UserName]:
    return users.get(user_id)
```

### 7.2 高级类型注解

```python
from typing import TypeVar, Generic, Callable, Type

T = TypeVar('T')

class Repository(Generic[T]):
    def __init__(self, model_class: Type[T]):
        self.model_class = model_class
    
    def find(self, id: int) -> Optional[T]:
        # 返回具体类型
        pass

# Protocol 类型
from typing import Protocol

class Serializable(Protocol):
    def to_dict(self) -> dict:
        ...

def save_to_json(obj: Serializable) -> str:
    import json
    return json.dumps(obj.to_dict())
```

### 7.3 类型检查工具

```python
# mypy 配置 (mypy.ini)
[mypy]
python_version = 3.10
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True

# 运行类型检查
# mypy myproject/
```

## 8. __init__.py组织

### 8.1 基本组织

```python
# mypackage/__init__.py
"""
My Package - A sample package for demonstration.
"""

from .core import main_function
from .utils import helper

__version__ = "1.0.0"
__author__ = "Your Name"
__all__ = ["main_function", "helper"]
```

### 8.2 延迟导入

```python
# mypackage/__init__.py
def __getattr__(name):
    if name == "LargeClass":
        from .large_module import LargeClass
        return LargeClass
    raise AttributeError(f"module {__name__!r} has no attribute {name!r}")

# 使用时才导入，减少启动时间
from mypackage import LargeClass  # 此时才导入
```

### 8.3 命名空间包

```python
# 不需要 __init__.py，多个包可以共享命名空间
# package1/mypackage/module_a.py
# package2/mypackage/module_b.py

# 两者都属于 mypackage 命名空间
from mypackage.module_a import func_a
from mypackage.module_b import func_b
```

## 9. 配置与代码分离

### 9.1 使用配置文件

```python
# config.yaml
database:
  host: localhost
  port: 5432
  name: mydb

api:
  timeout: 30
  retry: 3

# config.py
import yaml
from dataclasses import dataclass
from typing import Optional

@dataclass
class DatabaseConfig:
    host: str
    port: int
    name: str

@dataclass
class ApiConfig:
    timeout: int
    retry: int

@dataclass
class Config:
    database: DatabaseConfig
    api: ApiConfig
    
    @classmethod
    def from_yaml(cls, path: str) -> "Config":
        with open(path, 'r') as f:
            data = yaml.safe_load(f)
        return cls(
            database=DatabaseConfig(**data['database']),
            api=ApiConfig(**data['api'])
        )

# 使用
config = Config.from_yaml('config.yaml')
```

### 9.2 环境变量

```python
import os
from dataclasses import dataclass

@dataclass
class Config:
    database_url: str
    api_key: str
    debug: bool = False
    
    @classmethod
    def from_env(cls) -> "Config":
        return cls(
            database_url=os.getenv('DATABASE_URL', 'sqlite:///default.db'),
            api_key=os.getenv('API_KEY', ''),
            debug=os.getenv('DEBUG', 'false').lower() == 'true'
        )

# 使用 pydantic 进行验证
from pydantic import BaseSettings

class Settings(BaseSettings):
    database_url: str
    api_key: str
    debug: bool = False
    
    class Config:
        env_file = '.env'

settings = Settings()
```

### 9.3 多环境配置

```python
# config/
# ├── base.yaml      # 基础配置
# ├── development.yaml
# ├── staging.yaml
# └── production.yaml

import os
import yaml
from typing import Dict, Any

class ConfigManager:
    def __init__(self, config_dir: str = 'config'):
        self.config_dir = config_dir
        self.env = os.getenv('ENV', 'development')
        self._config = self._load_config()
    
    def _load_config(self) -> Dict[str, Any]:
        # 先加载基础配置
        with open(f'{self.config_dir}/base.yaml') as f:
            config = yaml.safe_load(f)
        
        # 再加载环境特定配置
        env_config_path = f'{self.config_dir}/{self.env}.yaml'
        if os.path.exists(env_config_path):
            with open(env_config_path) as f:
                env_config = yaml.safe_load(f)
                config = self._deep_merge(config, env_config)
        
        return config
    
    def _deep_merge(self, base: dict, override: dict) -> dict:
        result = base.copy()
        for key, value in override.items():
            if key in result and isinstance(result[key], dict) and isinstance(value, dict):
                result[key] = self._deep_merge(result[key], value)
            else:
                result[key] = value
        return result
    
    def get(self, key: str, default=None):
        keys = key.split('.')
        value = self._config
        for k in keys:
            if isinstance(value, dict):
                value = value.get(k)
            else:
                return default
        return value if value is not None else default

# 使用
config = ConfigManager()
db_host = config.get('database.host')
```

## 检查清单

### 模块化设计
- [ ] 每个模块职责单一
- [ ] 模块间依赖关系清晰
- [ ] 无循环导入
- [ ] 使用绝对导入

### 包结构组织
- [ ] 遵循标准项目结构
- [ ] __init__.py 合理导出公共 API
- [ ] 配置文件与代码分离
- [ ] 测试代码独立目录

### SOLID原则
- [ ] 单一职责：每个类只做一件事
- [ ] 开闭原则：对扩展开放，对修改关闭
- [ ] 里氏替换：子类可替换父类
- [ ] 接口隔离：接口小而专一
- [ ] 依赖倒置：依赖抽象不依赖具体

### 依赖管理
- [ ] 使用依赖注入
- [ ] 避免全局变量
- [ ] 明确依赖关系
- [ ] 使用类型注解标注依赖

### 数据类型
- [ ] 使用 dataclass 定义数据类
- [ ] 完整的类型注解
- [ ] 使用类型检查工具验证
- [ ] 避免 Any 类型滥用

### 配置管理
- [ ] 配置与代码分离
- [ ] 支持多环境配置
- [ ] 敏感信息使用环境变量
- [ ] 配置验证
