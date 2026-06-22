---
name: python-naming
description: Python编码规范 - 命名规范。适用于所有Python代码开发场景：(1) 编写任何Python模块、类、函数时的命名检查；(2) 代码审查时检查命名规范；(3) 重构时确保命名符合PEP 8规范。触发关键词：Python、命名、snake_case、PascalCase、常量、变量、函数、类、naming、PEP 8
---

# Python命名规范

基于PEP 8命名规范

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 模块名小写，可用下划线分隔 |
| 规则1.2 | 包名小写，简短 |
| 规则1.3 | 类名PascalCase（大驼峰） |
| 规则1.4 | 函数/方法名snake_case |
| 规则1.5 | 变量名snake_case |
| 规则1.6 | 常量UPPER_SNAKE_CASE |

## 命名风格对照

| 类型 | 风格 | 示例 |
|-----|------|-----|
| 模块/包 | 小写+下划线 | `user_service.py` |
| 类 | PascalCase | `UserService`, `HttpRequest` |
| 函数/方法 | snake_case | `get_user_by_id()` |
| 变量 | snake_case | `user_name`, `record_count` |
| 常量 | UPPER_SNAKE_CASE | `MAX_CONNECTIONS` |
| 私有属性 | 前导下划线 | `_internal_state` |
| 名称修饰 | 双前导下划线 | `__private_var` |
| 魔术方法 | 双下划线包围 | `__init__`, `__str__` |

## 常见错误

**错误示例**:
```python
def GetUser(userId):      # 函数应snake_case
    pass
UserName = "Alice"        # 变量应snake_case
maxconnections = 100       # 常量应UPPER_SNAKE_CASE
class user_service:        # 类应PascalCase
    pass
```

**正确示例**:
```python
def get_user(user_id):
    pass
user_name = "Alice"
MAX_CONNECTIONS = 100
class UserService:
    pass
```

## 详细规范

见 [references/naming.md](references/naming.md)
