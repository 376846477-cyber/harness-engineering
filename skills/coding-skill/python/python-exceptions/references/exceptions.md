# Python异常处理规范详细参考

## 一、try/except/finally 规范

### 1.1 基本结构

```python
try:
    result = risky_operation()
except ValueError as e:
    logger.error(f"参数错误: {e}")
    raise
except (IOError, OSError) as e:
    logger.error(f"IO错误: {e}")
    return default_value
except Exception as e:
    logger.exception("未知错误")
    raise
else:
    logger.info(f"操作成功: {result}")
    return result
finally:
    cleanup_resources()
```

### 1.2 关键规则

- **禁止裸except**：`except:` 会捕获所有异常包括系统退出信号
- **捕获具体异常**：明确指定异常类型，避免过度捕获
- **使用else分支**：无异常时执行的代码放else中
- **finally必须执行**：资源清理代码放finally，确保执行

### 1.3 错误/正确对比

**错误示例** - 裸except会捕获KeyboardInterrupt：
```python
try:
    do_something()
except:
    pass
```

**正确示例** - 捕获具体异常：
```python
try:
    do_something()
except ValueError as e:
    logger.warning(f"参数错误: {e}")
except IOError as e:
    logger.error(f"IO错误: {e}")
```

**错误示例** - 捕获范围过大：
```python
try:
    process_file(path)
except Exception:
    pass
```

**正确示例** - 捕获具体异常并记录：
```python
try:
    process_file(path)
except FileNotFoundError:
    logger.warning(f"文件不存在: {path}")
except PermissionError:
    logger.error(f"无权限访问: {path}")
```

**错误示例** - 忽略异常不记录：
```python
try:
    result = parse_config(config_str)
except ValueError:
    pass
```

**正确示例** - 记录异常并采取恢复措施：
```python
try:
    result = parse_config(config_str)
except ValueError as e:
    logger.warning(f"配置解析失败，使用默认配置: {e}")
    result = get_default_config()
```

**错误示例** - 异常信息泄露敏感数据：
```python
try:
    login(username, password)
except Exception as e:
    logger.error(f"登录失败: {e}")
    return f"错误: {e}"
```

**正确示例** - 异常信息脱敏：
```python
try:
    login(username, password)
except AuthenticationError:
    logger.error(f"用户 {username} 登录失败")
    return "用户名或密码错误"
except Exception as e:
    logger.exception("登录异常")
    return "登录服务暂时不可用"
```

## 二、自定义异常类

### 2.1 基本自定义异常

```python
class ApplicationError(Exception):
    """应用基础异常"""
    def __init__(self, message: str, code: int = None):
        self.message = message
        self.code = code
        super().__init__(self.message)
    
    def __str__(self):
        if self.code:
            return f"[{self.code}] {self.message}"
        return self.message


class ValidationError(ApplicationError):
    """数据校验异常"""
    def __init__(self, message: str, field: str = None):
        self.field = field
        super().__init__(message, code=400)


class BusinessError(ApplicationError):
    """业务逻辑异常"""
    def __init__(self, message: str, code: int = 500):
        super().__init__(message, code=code)


class DatabaseError(ApplicationError):
    """数据库操作异常"""
    def __init__(self, message: str, original_error: Exception = None):
        self.original_error = original_error
        super().__init__(message, code=500)
```

### 2.2 异常层次结构

```
Exception
└── ApplicationError (应用基础异常)
    ├── ValidationError (校验异常 - 400)
    ├── AuthenticationError (认证异常 - 401)
    ├── AuthorizationError (授权异常 - 403)
    ├── NotFoundError (资源不存在 - 404)
    ├── BusinessError (业务异常 - 500)
    └── DatabaseError (数据库异常 - 500)
```

### 2.3 使用示例

```python
def validate_user(user_data: dict):
    if not user_data.get("username"):
        raise ValidationError("用户名不能为空", field="username")
    if len(user_data["username"]) < 3:
        raise ValidationError("用户名至少3个字符", field="username")
    return True
```

## 三、异常链（raise from）

### 3.1 保留原始异常

```python
def load_config(path: str):
    try:
        with open(path, 'r') as f:
            return json.load(f)
    except FileNotFoundError as e:
        raise ConfigurationError(f"配置文件不存在: {path}") from e
    except json.JSONDecodeError as e:
        raise ConfigurationError(f"配置文件格式错误: {path}") from e
```

### 3.2 异常链的好处

```python
try:
    connect_database()
except ConnectionError as e:
    # 保留原始异常信息，便于调试
    raise DatabaseError("数据库连接失败") from e
    # 堆栈跟踪会显示两个异常的完整信息
```

### 3.3 显式抑制异常链

```python
try:
    convert_data(data)
except ValueError:
    # 转换为新异常，不需要保留原始异常
    raise ValidationError("数据格式错误") from None
```

## 四、上下文管理器

### 4.1 使用with语句

```python
# 文件操作
with open('data.txt', 'r') as f:
    content = f.read()

# 数据库连接
with get_db_connection() as conn:
    cursor = conn.cursor()
    cursor.execute(sql)

# 锁管理
with threading.Lock():
    shared_resource.modify()
```

### 4.2 使用contextlib装饰器

```python
from contextlib import contextmanager

@contextmanager
def timer(name: str):
    start = time.time()
    try:
        yield
    finally:
        elapsed = time.time() - start
        logger.info(f"{name} 耗时: {elapsed:.2f}秒")

# 使用
with timer("数据处理"):
    process_large_dataset(data)


@contextmanager
def db_transaction(conn):
    cursor = conn.cursor()
    try:
        yield cursor
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        cursor.close()
```

### 4.3 自定义上下文管理类

```python
class DatabaseConnection:
    def __init__(self, config):
        self.config = config
        self.conn = None
    
    def __enter__(self):
        self.conn = create_connection(self.config)
        return self.conn
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            self.conn.rollback()
            logger.error(f"事务回滚: {exc_val}")
        else:
            self.conn.commit()
        self.conn.close()
        return False  # 不抑制异常
```

## 五、异常层次（内置异常）

### 5.1 常用内置异常

| 异常类型 | 使用场景 |
|---------|---------|
| ValueError | 参数值不合法 |
| TypeError | 参数类型不正确 |
| KeyError | 字典键不存在 |
| IndexError | 序列索引越界 |
| FileNotFoundError | 文件不存在 |
| PermissionError | 权限不足 |
| RuntimeError | 运行时一般错误 |
| NotImplementedError | 功能未实现 |

### 5.2 选择正确的异常类型

```python
# 参数校验
def divide(a, b):
    if b == 0:
        raise ValueError("除数不能为零")
    return a / b

# 类型检查
def process(data: list):
    if not isinstance(data, list):
        raise TypeError(f"期望list类型，实际为{type(data).__name__}")

# 功能未实现
class AbstractProcessor:
    def process(self, data):
        raise NotImplementedError("子类必须实现process方法")
```

## 六、记录异常日志

### 6.1 使用logging模块

```python
import logging

logger = logging.getLogger(__name__)

try:
    risky_operation()
except ValueError as e:
    logger.warning(f"参数警告: {e}")
except Exception as e:
    # logger.exception 自动记录堆栈信息
    logger.exception("操作失败")
    raise
```

### 6.2 日志级别选择

- **DEBUG**: 详细调试信息，开发阶段使用
- **INFO**: 正常操作流程信息
- **WARNING**: 可恢复的异常，需要关注
- **ERROR**: 严重错误，影响功能但程序可继续
- **CRITICAL**: 致命错误，程序可能无法继续

```python
# 可恢复异常
try:
    config = load_config()
except FileNotFoundError:
    logger.warning("配置文件不存在，使用默认配置")
    config = get_default_config()

# 严重错误
try:
    result = critical_operation()
except Exception as e:
    logger.error(f"关键操作失败: {e}", exc_info=True)
    raise
```

## 七、资源清理

### 7.1 使用finally确保清理

```python
file = None
try:
    file = open('data.txt', 'r')
    content = file.read()
    process(content)
finally:
    if file:
        file.close()
```

### 7.2 使用上下文管理器（推荐）

```python
with open('data.txt', 'r') as file:
    content = file.read()
    process(content)
# 自动关闭文件，即使发生异常
```

### 7.3 多资源管理

```python
# 正确：使用多个with
with open('input.txt', 'r') as infile, open('output.txt', 'w') as outfile:
    for line in infile:
        outfile.write(process(line))


# 或使用ExitStack管理动态资源
from contextlib import ExitStack

with ExitStack() as stack:
    files = [stack.enter_context(open(f)) for f in file_list]
    process_files(files)
```

## 八、最佳实践检查清单

### ✅ 异常捕获

- [ ] 禁止使用裸except
- [ ] 捕获具体异常类型
- [ ] except块代码尽量简短
- [ ] 避免在except中隐藏错误

### ✅ 异常抛出

- [ ] 使用有意义的错误消息
- [ ] 使用raise from保留异常链
- [ ] 选择正确的异常类型
- [ ] 自定义异常继承Exception

### ✅ 资源管理

- [ ] 使用with语句管理资源
- [ ] finally块确保资源释放
- [ ] 使用contextlib简化上下文管理

### ✅ 日志记录

- [ ] 使用logging模块记录异常
- [ ] 选择合适的日志级别
- [ ] 包含足够的上下文信息
- [ ] 生产环境禁用print异常

### ✅ 代码质量

- [ ] 区分业务异常和系统异常
- [ ] 异常消息对用户友好
- [ ] 异常代码可追踪定位
- [ ] 单元测试覆盖异常分支

---

## 九、自动化工具推荐

### 静态检查工具

| 工具 | 用途 | 安装 |
|-----|------|------|
| **ruff** | 快速lint，检测裸except等异常反模式 | `pip install ruff` |
| **mypy** | 类型检查，发现类型相关异常隐患 | `pip install mypy` |
| **pylint** | 深度分析，检测broad-except和bare-except | `pip install pylint` |
| **bandit** | 安全扫描，检测异常中的信息泄露 | `pip install bandit` |
| **pytest** | 单元测试，覆盖异常分支 | `pip install pytest` |
| **pytest-raises** | 异常测试断言 | 内置于pytest |

### ruff异常相关规则

```toml
# pyproject.toml
[tool.ruff.lint]
select = [
    "E",    # pycodestyle
    "B",    # flake8-bugbear
    "BLE",  # flake8-blind-except (检测裸except)
    "SIM",  # flake8-simplify
    "TRY",  # tryceratops (异常处理最佳实践)
]
ignore = []

[tool.ruff.lint.flake8-blind-except]
# 禁止裸except
```

### pylint异常相关规则

```ini
# .pylintrc
[MESSAGES CONTROL]
enable =
    broad-except,      # 检测过宽的except
    bare-except,       # 检测裸except
    catching-non-exception,  # 检测捕获非异常类型
    misplaced-future,  # 检测future导入位置
    try-except-raise,  # 检测无意义的try-except-raise
```

### 异常测试示例

```python
import pytest

def test_divide_by_zero():
    with pytest.raises(ValueError, match="除数不能为零"):
        divide(10, 0)

def test_file_not_found():
    with pytest.raises(FileNotFoundError):
        read_config("nonexistent.json")

def test_custom_exception_chain():
    with pytest.raises(DatabaseError) as exc_info:
        connect_database("invalid://url")
    assert exc_info.value.__cause__ is not None
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
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy
```

### CI集成建议

```yaml
# GitHub Actions示例
name: Exception Quality
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install ruff mypy pylint pytest
      - run: ruff check --select BLE,TRY .
      - run: mypy --strict .
      - run: pytest -x --tb=short
```
