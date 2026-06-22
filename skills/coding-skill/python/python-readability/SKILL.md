---
name: python-readability
description: Python编码规范 - 可读性规范。适用于所有Python代码开发场景：(1) 代码编写时提升可读性；(2) 代码审查时检查可读性；(3) 重构代码使其更Pythonic。触发关键词：Python、可读、readability、Pythonic、易读、理解、阅读
---

# Python可读性规范

基于Clean Code指导书

## 核心原则

提升可读性就是要：**传递足够多的有用信息、减少信息失真、减少信息干扰、降低信息理解难度**。Pythonic代码天然具有高可读性。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 利用名称传递信息 |
| 1.2 | 利用格式传递信息 |
| 1.3 | 注释作为代码的补充 |
| 2.1 | 使用Pythonic惯用法 |
| 3.1 | 无废弃代码 |
| 4.1 | 功能单一 |
| 4.2 | 简化语句逻辑，使用卫语句 |
| 4.3 | 变量作用域尽量小 |

## Pythonic示例

```python
# 不Pythonic
result = []
for item in items:
    if item.is_valid():
        result.append(item.name)

# Pythonic
result = [item.name for item in items if item.is_valid()]
```

```python
# 不Pythonic
if len(my_list) > 0:
    process(my_list)

# Pythonic
if my_list:
    process(my_list)
```

## 简化嵌套

```python
# 卫语句
def get_user(user_id):
    if user_id is None:
        return None
    user = repository.find_by_id(user_id)
    if user is None:
        return None
    if not user.is_active:
        return None
    return user
```

## 详细规范

见 [references/readability.md](references/readability.md)

## 检查清单

- [ ] 名称是否传递了足够的信息？
- [ ] 代码是否Pythonic？
- [ ] 代码格式是否一致？
- [ ] 是否有废弃代码？
- [ ] 函数是否功能单一？
- [ ] 嵌套层级是否过深？
