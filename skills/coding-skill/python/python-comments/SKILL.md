---
name: python-comments
description: Python编码规范 - 注释规范。适用于所有Python代码开发场景：(1) 编写任何Python模块的Docstring；(2) 代码审查时检查注释规范；(3) 添加函数注释、类注释。触发关键词：Python、注释、Docstring、comment、doc、文档、PEP 257
---

# Python注释规范

基于PEP 257 Docstring规范

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 公开模块、类、函数使用Docstring |
| 规则1.2 | 使用三重双引号 |
| 规则1.3 | 单行Docstring写在一行 |
| 规则1.4 | 多行Docstring包含摘要、空行、详细说明 |

## Docstring格式

### 模块Docstring
```python
"""用户服务模块。

提供用户管理的核心功能，包括创建、查询、更新和删除用户。
"""
```

### 函数Docstring
```python
def get_user_by_id(user_id: int) -> User:
    """根据用户ID获取用户信息。

    Args:
        user_id: 用户唯一标识符

    Returns:
        User: 用户信息对象

    Raises:
        UserNotFoundError: 用户不存在时抛出
    """
```

### 类Docstring
```python
class UserService:
    """用户管理服务。

    提供用户CRUD操作和业务逻辑处理。

    Attributes:
        repository: 用户数据仓库实例
    """
```

## 详细规范

见 [references/comments.md](references/comments.md)
