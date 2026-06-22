---
name: python-exceptions
description: Python编码规范 - 异常处理规范。适用于所有Python代码开发场景：(1) 编写try/except代码；(2) 代码审查时检查异常处理；(3) 自定义异常类；(4) 上下文管理器。触发关键词：Python、异常、try、except、raise、Exception、上下文、contextmanager
---

# Python异常处理规范

基于Clean Code指导书

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 不捕获裸except或BaseException |
| 规则1.2 | 异常处理要具体精确 |
| 规则1.3 | 不要忽略异常，至少记录日志 |
| 规则1.4 | 使用with语句管理资源 |
| 规则1.5 | 异常信息不含敏感数据 |

## 异常处理示例

**错误示例**:
```python
try:
    result = do_something()
except:                    # 裸except，捕获所有异常
    pass                   # 忽略异常
```

**正确示例**:
```python
try:
    result = do_something()
except ValueError as e:
    logger.error(f"值错误: {e}")
    raise InvalidInputError("输入值无效") from e
except IOError as e:
    logger.error(f"IO错误: {e}")
    raise DataAccessException("数据访问失败") from e
```

## 自定义异常

```python
class AppError(Exception):
    """应用基础异常。"""
    pass

class UserNotFoundError(AppError):
    """用户不存在异常。"""
    pass

class InvalidInputError(AppError):
    """无效输入异常。"""
    pass
```

## 上下文管理器

```python
# 使用with管理资源
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

## 详细规范

见 [references/exceptions.md](references/exceptions.md)
