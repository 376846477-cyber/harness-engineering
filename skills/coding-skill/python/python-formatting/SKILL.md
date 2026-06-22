---
name: python-formatting
description: Python编码规范 - 排版格式规范。适用于所有Python代码开发场景：(1) 编写Python代码时的格式检查；(2) 代码审查时检查缩进、空行；(3) 格式化代码；(4) import排序。触发关键词：Python、格式、缩进、空行、import、格式化、format、PEP 8、Black
---

# Python排版格式规范

基于PEP 8排版格式规范

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 4空格缩进，不使用制表符 |
| 规则1.2 | 行宽不超过79字符（团队可放宽至99或120） |
| 规则1.3 | import每行一个，按标准库/第三方/本地分组 |
| 规则1.4 | 类之间2空行，方法之间1空行 |
| 规则1.5 | 运算符两侧加空格，逗号后加空格 |

## 缩进示例

```python
# 正确：4空格缩进
def calculate_total(items):
    total = 0
    for item in items:
        if item.is_valid():
            total += item.price
    return total
```

## import排序

```python
# 标准库
import os
import sys

# 第三方库
import requests
from django import forms

# 本地模块
from myapp.models import User
from myapp.utils import format_date
```

## 空行规范

```python
import os


class UserService:
    """用户服务类。"""

    def __init__(self, repository):
        self.repository = repository

    def get_user(self, user_id):
        return self.repository.find_by_id(user_id)

    def create_user(self, data):
        return self.repository.create(data)


def helper_function():
    pass
```

## 详细规范

见 [references/formatting.md](references/formatting.md)
