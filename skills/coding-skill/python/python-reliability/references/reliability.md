# Python可靠性规范详细参考

## 一、防御式编程

### 1.1 类型检查

使用类型注解和运行时类型检查确保代码健壮性：

**错误示例** - 无类型检查，运行时才发现错误：
```python
def process_data(data, threshold):
    return [item['value'] for item in data if item['value'] > threshold]
```

**正确示例** - 类型注解+运行时校验：
```python
from typing import TypeVar, Generic, Optional, List, Dict, Any
from dataclasses import dataclass

T = TypeVar('T')

def validate_type(value: Any, expected_type: type, name: str = "parameter") -> None:
    if not isinstance(value, expected_type):
        raise TypeError(
            f"{name} must be {expected_type.__name__}, "
            f"got {type(value).__name__}"
        )

def process_data(data: List[Dict[str, Any]], threshold: float) -> List[float]:
    validate_type(data, list, "data")
    validate_type(threshold, (int, float), "threshold")
    
    if not all(isinstance(item, dict) for item in data):
        raise TypeError("All items in data must be dictionaries")
    
    if threshold < 0 or threshold > 1:
        raise ValueError(f"threshold must be between 0 and 1, got {threshold}")
    
    return [item.get('value', 0.0) for item in data if item.get('value', 0) > threshold]
```

### 1.2 参数验证

```python
from typing import Callable, Any
import re

class ParameterValidator:
    @staticmethod
    def validate_string(value: str, min_len: int = 0, max_len: int = None,
                        pattern: str = None) -> str:
        if not isinstance(value, str):
            raise TypeError(f"Expected string, got {type(value).__name__}")
        if len(value) < min_len:
            raise ValueError(f"String length must be >= {min_len}")
        if max_len and len(value) > max_len:
            raise ValueError(f"String length must be <= {max_len}")
        if pattern and not re.match(pattern, value):
            raise ValueError(f"String does not match pattern: {pattern}")
        return value
    
    @staticmethod
    def validate_number(value: Any, min_val: float = None, 
                        max_val: float = None) -> float:
        if not isinstance(value, (int, float)):
            raise TypeError(f"Expected number, got {type(value).__name__}")
        if min_val is not None and value < min_val:
            raise ValueError(f"Value must be >= {min_val}")
        if max_val is not None and value > max_val:
            raise ValueError(f"Value must be <= {max_val}")
        return float(value)
```

### 1.3 None值处理

```python
from typing import Optional

def safe_get(data: Dict[str, Any], key: str, default: Any = None) -> Any:
    return data.get(key, default) if data is not None else default

def require_non_null(value: Optional[T], name: str = "value") -> T:
    if value is None:
        raise ValueError(f"{name} cannot be None")
    return value

class ConfigLoader:
    def __init__(self, config: Optional[Dict[str, Any]] = None):
        self._config = config or {}
    
    def get(self, key: str, default: Any = None) -> Any:
        keys = key.split('.')
        value = self._config
        for k in keys:
            if isinstance(value, dict):
                value = value.get(k)
            else:
                return default
        return value if value is not None else default
```

## 二、资源管理

### 2.1 with语句

**错误示例** - 手动管理资源，异常时资源泄漏：
```python
f = open(path, 'r', encoding='utf-8')
content = f.read()
f.close()
```

**正确示例** - 使用with自动管理资源：
```python
with open(path, 'r', encoding='utf-8') as f:
    content = f.read()
```

**错误示例** - 数据库连接未使用上下文管理：
```python
conn = create_connection(url)
cursor = conn.cursor()
cursor.execute(sql)
conn.commit()
conn.close()
```

**正确示例** - 自定义上下文管理器：
```python
from contextlib import contextmanager, closing
import socket
import threading

@contextmanager
def managed_file(path: str, mode: str = 'r'):
    f = None
    try:
        f = open(path, mode, encoding='utf-8')
        yield f
    finally:
        if f is not None:
            f.close()

@contextmanager
def managed_lock(lock: threading.Lock):
    acquired = lock.acquire(timeout=10.0)
    if not acquired:
        raise TimeoutError("Failed to acquire lock within timeout")
    try:
        yield
    finally:
        lock.release()

class DatabaseConnection:
    def __init__(self, connection_string: str):
        self._conn_string = connection_string
        self._connection = None
    
    def __enter__(self):
        self._connection = self._create_connection()
        return self._connection
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if self._connection:
            if exc_type is not None:
                self._connection.rollback()
            else:
                self._connection.commit()
            self._connection.close()
        return False
    
    def _create_connection(self):
        return None
```

### 2.2 contextmanager装饰器

```python
from contextlib import contextmanager, AbstractContextManager
import time
from typing import Generator, Any

@contextmanager
def timer(name: str) -> Generator[None, None, None]:
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        print(f"{name} took {elapsed:.4f} seconds")

@contextmanager
def change_directory(path: str) -> Generator[str, None, None]:
    import os
    old_dir = os.getcwd()
    try:
        os.chdir(path)
        yield path
    finally:
        os.chdir(old_dir)

class ResourcePool(AbstractContextManager):
    def __init__(self, max_size: int = 10):
        self._max_size = max_size
        self._resources: List[Any] = []
        self._in_use: set = set()
    
    def acquire(self) -> Any:
        if not self._resources and len(self._in_use) >= self._max_size:
            raise RuntimeError("Resource pool exhausted")
        if self._resources:
            resource = self._resources.pop()
        else:
            resource = self._create_resource()
        self._in_use.add(id(resource))
        return resource
    
    def release(self, resource: Any) -> None:
        resource_id = id(resource)
        if resource_id in self._in_use:
            self._in_use.remove(resource_id)
            self._resources.append(resource)
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        for resource in list(self._in_use):
            self.release(resource)
        return False
```

## 三、类型注解

### 3.1 typing模块使用

```python
from typing import (
    List, Dict, Set, Tuple, Optional, Union,
    Callable, TypeVar, Generic, Protocol, Any
)
from typing import runtime_checkable

T = TypeVar('T')
K = TypeVar('K')
V = TypeVar('V')

@runtime_checkable
class Serializable(Protocol):
    def to_dict(self) -> Dict[str, Any]: ...
    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> 'Serializable': ...

class Cache(Generic[K, V]):
    def __init__(self, max_size: int = 1000):
        self._data: Dict[K, V] = {}
        self._max_size = max_size
    
    def get(self, key: K) -> Optional[V]:
        return self._data.get(key)
    
    def set(self, key: K, value: V) -> None:
        if len(self._data) >= self._max_size:
            oldest = next(iter(self._data))
            del self._data[oldest]
        self._data[key] = value

def transform_data(
    items: List[Dict[str, Any]],
    converter: Callable[[Dict[str, Any]], T]
) -> List[T]:
    return [converter(item) for item in items]

def merge_configs(
    *configs: Dict[str, Any],
    overwrite: bool = True
) -> Dict[str, Any]:
    result: Dict[str, Any] = {}
    for config in configs:
        if overwrite:
            result.update(config)
        else:
            for k, v in config.items():
                if k not in result:
                    result[k] = v
    return result
```

## 四、输入验证

### 4.1 数据验证器

```python
from dataclasses import dataclass, field
from typing import Any, Dict, List, Type, Callable
import re

@dataclass
class ValidationResult:
    is_valid: bool
    errors: List[str] = field(default_factory=list)

class DataValidator:
    def __init__(self):
        self._validators: Dict[str, List[Callable]] = {}
    
    def add_validator(self, field_name: str, 
                      validator: Callable[[Any], bool],
                      error_msg: str = "Validation failed") -> None:
        if field_name not in self._validators:
            self._validators[field_name] = []
        self._validators[field_name].append((validator, error_msg))
    
    def validate(self, data: Dict[str, Any]) -> ValidationResult:
        errors = []
        for field_name, validators in self._validators.items():
            value = data.get(field_name)
            for validator, error_msg in validators:
                if not validator(value):
                    errors.append(f"{field_name}: {error_msg}")
        return ValidationResult(len(errors) == 0, errors)

def validate_email(email: str) -> bool:
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))

def validate_url(url: str) -> bool:
    pattern = r'^https?://[^\s/$.?#].[^\s]*$'
    return bool(re.match(pattern, url))

def validate_phone(phone: str) -> bool:
    pattern = r'^\+?1?\d{9,15}$'
    return bool(re.match(pattern, phone))
```

## 五、错误处理策略

### 5.1 分层错误处理

```python
from enum import Enum
from typing import Optional, Any, Callable

class ErrorSeverity(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CRITICAL = 4

class ApplicationError(Exception):
    def __init__(self, message: str, severity: ErrorSeverity = ErrorSeverity.MEDIUM,
                 recoverable: bool = True, context: Optional[Dict] = None):
        super().__init__(message)
        self.severity = severity
        self.recoverable = recoverable
        self.context = context or {}

class ValidationError(ApplicationError):
    def __init__(self, field: str, message: str):
        super().__init__(
            f"Validation failed for {field}: {message}",
            severity=ErrorSeverity.LOW,
            recoverable=True
        )
        self.field = field

class ResourceError(ApplicationError):
    def __init__(self, resource: str, operation: str, cause: Optional[Exception] = None):
        super().__init__(
            f"Failed to {operation} resource {resource}",
            severity=ErrorSeverity.HIGH,
            recoverable=False,
            context={'resource': resource, 'operation': operation}
        )
        self.__cause__ = cause

class ErrorHandler:
    def __init__(self):
        self._handlers: Dict[ErrorSeverity, Callable] = {}
    
    def handle(self, error: ApplicationError) -> Any:
        handler = self._handlers.get(error.severity, self._default_handler)
        return handler(error)
    
    def register_handler(self, severity: ErrorSeverity, 
                        handler: Callable[[ApplicationError], Any]) -> None:
        self._handlers[severity] = handler
    
    def _default_handler(self, error: ApplicationError) -> None:
        print(f"[{error.severity.name}] {error}")
```

## 六、日志记录

### 6.1 结构化日志

```python
import logging
import json
from datetime import datetime
from typing import Any, Dict
from functools import wraps

class StructuredLogger:
    def __init__(self, name: str, level: int = logging.INFO):
        self._logger = logging.getLogger(name)
        self._logger.setLevel(level)
        if not self._logger.handlers:
            handler = logging.StreamHandler()
            handler.setFormatter(logging.Formatter('%(message)s'))
            self._logger.addHandler(handler)
    
    def log(self, level: int, message: str, **kwargs: Any) -> None:
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': logging.getLevelName(level),
            'message': message,
            **kwargs
        }
        self._logger.log(level, json.dumps(log_entry, default=str))
    
    def info(self, message: str, **kwargs: Any) -> None:
        self.log(logging.INFO, message, **kwargs)
    
    def error(self, message: str, exc_info: bool = False, **kwargs: Any) -> None:
        if exc_info:
            import traceback
            kwargs['traceback'] = traceback.format_exc()
        self.log(logging.ERROR, message, **kwargs)

def log_calls(logger: StructuredLogger):
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            logger.info(
                f"Calling {func.__name__}",
                args=str(args)[:200],
                kwargs=str(kwargs)[:200]
            )
            try:
                result = func(*args, **kwargs)
                logger.info(f"{func.__name__} completed successfully")
                return result
            except Exception as e:
                logger.error(f"{func.__name__} failed", exc_info=True, error=str(e))
                raise
        return wrapper
    return decorator
```

## 七、重试机制

### 7.1 自定义重试器

```python
import time
import random
from typing import Callable, Type, Tuple, Optional
from functools import wraps

class RetryConfig:
    def __init__(
        self,
        max_attempts: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 60.0,
        exponential_base: float = 2.0,
        jitter: bool = True,
        retryable_exceptions: Tuple[Type[Exception], ...] = (Exception,)
    ):
        self.max_attempts = max_attempts
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.exponential_base = exponential_base
        self.jitter = jitter
        self.retryable_exceptions = retryable_exceptions

def retry(config: Optional[RetryConfig] = None):
    if config is None:
        config = RetryConfig()
    
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(config.max_attempts):
                try:
                    return func(*args, **kwargs)
                except config.retryable_exceptions as e:
                    last_exception = e
                    if attempt < config.max_attempts - 1:
                        delay = min(
                            config.base_delay * (config.exponential_base ** attempt),
                            config.max_delay
                        )
                        if config.jitter:
                            delay *= (0.5 + random.random())
                        time.sleep(delay)
            raise last_exception
        return wrapper
    return decorator

@retry(RetryConfig(max_attempts=3, base_delay=0.5))
def unreliable_operation() -> str:
    if random.random() < 0.7:
        raise ConnectionError("Random failure")
    return "success"
```

## 八、降级处理

### 8.1 多层降级策略

```python
from typing import Callable, Any, Optional, Dict
from abc import ABC, abstractmethod
import time

class FallbackStrategy(ABC):
    @abstractmethod
    def execute(self, *args, **kwargs) -> Any:
        pass

class DefaultValueFallback(FallbackStrategy):
    def __init__(self, default_value: Any):
        self._default = default_value
    
    def execute(self, *args, **kwargs) -> Any:
        return self._default

class CacheFallback(FallbackStrategy):
    def __init__(self, cache: Dict[str, Any], key_generator: Callable):
        self._cache = cache
        self._key_gen = key_generator
    
    def execute(self, *args, **kwargs) -> Any:
        key = self._key_gen(*args, **kwargs)
        if key in self._cache:
            return self._cache[key]
        raise KeyError(f"No cached value for key: {key}")

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, reset_timeout: float = 60.0):
        self._failure_count = 0
        self._failure_threshold = failure_threshold
        self._reset_timeout = reset_timeout
        self._last_failure_time: Optional[float] = None
        self._state = 'CLOSED'
    
    def can_execute(self) -> bool:
        if self._state == 'OPEN':
            if time.time() - self._last_failure_time > self._reset_timeout:
                self._state = 'HALF_OPEN'
                return True
            return False
        return True
    
    def record_success(self) -> None:
        self._failure_count = 0
        self._state = 'CLOSED'
    
    def record_failure(self) -> None:
        self._failure_count += 1
        self._last_failure_time = time.time()
        if self._failure_count >= self._failure_threshold:
            self._state = 'OPEN'

class ResilientFunction:
    def __init__(
        self,
        func: Callable,
        fallback: Optional[FallbackStrategy] = None,
        circuit_breaker: Optional[CircuitBreaker] = None
    ):
        self._func = func
        self._fallback = fallback
        self._breaker = circuit_breaker or CircuitBreaker()
    
    def __call__(self, *args, **kwargs) -> Any:
        if not self._breaker.can_execute():
            if self._fallback:
                return self._fallback.execute(*args, **kwargs)
            raise RuntimeError("Circuit breaker is open")
        
        try:
            result = self._func(*args, **kwargs)
            self._breaker.record_success()
            return result
        except Exception as e:
            self._breaker.record_failure()
            if self._fallback:
                return self._fallback.execute(*args, **kwargs)
            raise
```

## 九、资源泄漏防护

### 9.1 内存管理

```python
import weakref
import gc
from typing import Any, Callable, Optional, Dict
import threading

class ObjectPool:
    def __init__(self, factory: Callable[[], Any], max_size: int = 100):
        self._factory = factory
        self._max_size = max_size
        self._pool: List[Any] = []
        self._lock = threading.Lock()
        self._weak_refs: weakref.WeakSet = weakref.WeakSet()
    
    def acquire(self) -> Any:
        with self._lock:
            if self._pool:
                obj = self._pool.pop()
            else:
                obj = self._factory()
            self._weak_refs.add(obj)
            return obj
    
    def release(self, obj: Any) -> None:
        with self._lock:
            if len(self._pool) < self._max_size:
                self._pool.append(obj)
    
    def cleanup(self) -> int:
        collected = gc.collect()
        with self._lock:
            self._pool = [obj for obj in self._pool 
                         if obj in self._weak_refs]
        return collected

class ResourceManager:
    _instance: Optional['ResourceManager'] = None
    _lock = threading.Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance._resources: Dict[int, Any] = {}
                    cls._instance._handlers: Dict[str, Callable] = {}
        return cls._instance
    
    def register(self, resource: Any, cleanup_handler: Callable) -> int:
        resource_id = id(resource)
        self._resources[resource_id] = resource
        self._handlers[resource_id] = cleanup_handler
        return resource_id
    
    def unregister(self, resource_id: int) -> None:
        if resource_id in self._resources:
            handler = self._handlers.pop(resource_id, None)
            if handler:
                try:
                    handler(self._resources[resource_id])
                except Exception:
                    pass
            del self._resources[resource_id]
    
    def cleanup_all(self) -> None:
        for resource_id in list(self._resources.keys()):
            self.unregister(resource_id)
```

### 9.2 文件句柄管理

```python
import os
import atexit
from typing import Set, Optional
import psutil

class FileHandleTracker:
    _instance: Optional['FileHandleTracker'] = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._handles: Set[int] = set()
            cls._instance._warnings: Set[int] = set()
            atexit.register(cls._instance._cleanup_on_exit)
        return cls._instance
    
    def track(self, file_obj) -> None:
        self._handles.add(id(file_obj))
    
    def untrack(self, file_obj) -> None:
        self._handles.discard(id(file_obj))
    
    def get_open_count(self) -> int:
        process = psutil.Process(os.getpid())
        try:
            return len(process.open_files())
        except psutil.AccessDenied:
            return -1
    
    def _cleanup_on_exit(self) -> None:
        import warnings
        if self._handles:
            warnings.warn(
                f"Resource leak detected: {len(self._handles)} "
                "file handles may not be properly closed",
                ResourceWarning
            )

def safe_open(file_path: str, mode: str = 'r', **kwargs):
    tracker = FileHandleTracker()
    file_obj = open(file_path, mode, **kwargs)
    tracker.track(file_obj)
    
    class TrackedFile:
        def __init__(self, f):
            self._file = f
        
        def __getattr__(self, name):
            return getattr(self._file, name)
        
        def __enter__(self):
            return self._file.__enter__()
        
        def __exit__(self, *args):
            tracker.untrack(self._file)
            return self._file.__exit__(*args)
        
        def close(self):
            tracker.untrack(self._file)
            return self._file.close()
    
    return TrackedFile(file_obj)
```

---

## 检查清单

### 防御式编程
- [ ] 所有公共函数都有参数类型检查
- [ ] 使用类型注解标注函数签名
- [ ] 边界条件已处理（空值、边界值）
- [ ] None值检查在关键路径上
- [ ] 使用 dataclass 或 pydantic 进行数据验证

### 资源管理
- [ ] 文件操作使用 with 语句
- [ ] 数据库连接、网络连接使用上下文管理器
- [ ] 锁和信号量正确释放
- [ ] 自定义资源类实现了 __enter__ 和 __exit__
- [ ] 使用 atexit 注册清理函数

### 类型注解
- [ ] 所有函数参数和返回值有类型注解
- [ ] 使用 Optional 标注可选参数
- [ ] 复杂类型使用 typing 模块（List, Dict, Union 等）
- [ ] 泛型类使用 TypeVar 和 Generic
- [ ] 协议使用 Protocol 定义

### 输入验证
- [ ] 外部输入都经过验证
- [ ] 使用验证器模式处理复杂验证逻辑
- [ ] 字符串输入检查长度和格式
- [ ] 数值输入检查范围
- [ ] 文件路径验证安全性

### 错误处理
- [ ] 使用自定义异常类区分错误类型
- [ ] 异常包含足够的上下文信息
- [ ] 不要捕获过于宽泛的异常
- [ ] 正确处理异常链（使用 raise ... from）
- [ ] 关键操作有错误恢复策略

### 日志记录
- [ ] 关键操作记录日志
- [ ] 异常记录完整堆栈信息
- [ ] 使用结构化日志格式
- [ ] 敏感信息已脱敏
- [ ] 日志级别使用正确

### 重试机制
- [ ] 不稳定操作有重试机制
- [ ] 重试有最大次数限制
- [ ] 使用指数退避策略
- [ ] 只重试可重试的异常
- [ ] 重试有超时限制

### 降级处理
- [ ] 关键功能有降级方案
- [ ] 使用熔断器防止级联故障
- [ ] 缓存作为降级数据源
- [ ] 默认值策略合理
- [ ] 降级状态可监控

### 资源泄漏防护
- [ ] 使用 weakref 避免循环引用
- [ ] 对象池限制最大数量
- [ ] 定期清理不再使用的资源
- [ ] 监控文件句柄数量
- [ ] 使用上下文管理器保证资源释放

---

## 十、自动化工具推荐

### 可靠性检查工具

| 工具 | 用途 | 安装 |
|-----|------|------|
| **mypy** | 静态类型检查，发现类型相关可靠性隐患 | `pip install mypy` |
| **ruff** | 快速lint，检测资源泄漏等反模式 | `pip install ruff` |
| **pylint** | 深度分析，检测未关闭资源等 | `pip install pylint` |
| **bandit** | 安全扫描，检测硬编码密钥等 | `pip install bandit` |
| **pytest** | 单元测试，验证容错和恢复逻辑 | `pip install pytest` |
| **pytest-timeout** | 测试超时控制 | `pip install pytest-timeout` |
| **pytest-retry** | 失败重试测试 | `pip install pytest-retry` |

### mypy严格模式配置

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_any_generics = true
```

### ruff可靠性相关规则

```toml
# pyproject.toml
[tool.ruff.lint]
select = [
    "E",    # pycodestyle
    "B",    # flake8-bugbear (资源泄漏等)
    "SIM",  # flake8-simplify
    "RUF",  # ruff自有规则
    "S",    # flake8-bandit (安全)
    "C4",   # flake8-comprehensions
]
```

### 可靠性测试示例

```python
import pytest
from unittest.mock import patch, MagicMock

def test_retry_on_transient_failure():
    call_count = 0
    def flaky_operation():
        nonlocal call_count
        call_count += 1
        if call_count < 3:
            raise ConnectionError("临时故障")
        return "success"
    
    with pytest.raises(ConnectionError):
        flaky_operation()
    
    call_count = 0
    result = retry(RetryConfig(max_attempts=3, base_delay=0.01))(flaky_operation)()
    assert result == "success"

def test_circuit_breaker_opens():
    breaker = CircuitBreaker(failure_threshold=3, reset_timeout=1.0)
    for _ in range(3):
        breaker.record_failure()
    assert not breaker.can_execute()

def test_fallback_on_failure():
    fallback = DefaultValueFallback(default_value="cached")
    strategy = ResilientFunction(
        func=lambda: (_ for _ in ()).throw(RuntimeError("服务不可用")),
        fallback=fallback
    )
    assert strategy() == "cached"

def test_resource_cleanup_on_exception():
    with pytest.raises(ValueError):
        with managed_file("test.txt", "w") as f:
            raise ValueError("模拟异常")
    assert f.closed
```

### pre-commit配置

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy
        args: [--strict]
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.8
    hooks:
      - id: bandit
```

### CI集成建议

```yaml
# GitHub Actions示例
name: Reliability Check
on: [push, pull_request]
jobs:
  reliability:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install ruff mypy bandit pytest pytest-timeout pytest-cov
      - run: ruff check .
      - run: mypy --strict .
      - run: bandit -r .
      - run: pytest --timeout=30 --cov=src --cov-fail-under=80
```
