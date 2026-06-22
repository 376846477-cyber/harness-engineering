# Python排版格式规范详细参考

本文档基于PEP 8规范，提供Python代码格式化的完整指南。

## 1. 缩进规范

### 基本规则
- **使用4个空格**作为缩进单位，禁止使用制表符（Tab）
- Python 3禁止混用空格和制表符

### 续行缩进
```python
# 推荐：括号对齐
def long_function(var_one, var_two,
                  var_three, var_four):
    print(var_one)

# 推荐：悬挂缩进（增加一级缩进）
def long_function(
        var_one, var_two,
        var_three, var_four):
    print(var_one)

# 条件语句续行
if (this_is_one_thing
        and that_is_another_thing):
    do_something()

# 列表、字典等可自然换行
my_list = [
    1, 2, 3,
    4, 5, 6,
]
```

### 缩进检查清单
- [ ] 所有缩进使用4空格
- [ ] 无Tab字符混用
- [ ] 续行缩进与括号对齐或有明显区分
- [ ] 嵌套层级清晰可辨

## 2. 行宽规范

### 推荐限制
- **代码行**：PEP 8建议79字符，团队可约定放宽至99或120字符
- **文档字符串和注释**：建议72字符，便于阅读
- **URL或路径**：可适当超长

### 长行处理
```python
# 推荐：括号隐式续行
result = some_function_with_long_name(
    'arg1', 'arg2', 'arg3')

# 不推荐：反斜杠续行（除非必要）
result = some_function_with_long_name('arg1', 'arg2', \
                                      'arg3')

# 字符串拼接
long_string = (
    '这是一个很长的字符串，'
    '使用括号进行隐式拼接'
)
```

### 行宽检查清单
- [ ] 代码行不超过约定限制（79/99/120）
- [ ] 注释和docstring不超过72字符
- [ ] 优先使用括号续行而非反斜杠

## 3. import排序规范

### 分组规则
1. **标准库模块**
2. **第三方模块**
3. **本地/项目模块**

各组之间空一行分隔。

### 示例
```python
# 标准库
import os
import sys
from collections import defaultdict
from typing import Dict, List, Optional

# 第三方库
import numpy as np
import pandas as pd
from flask import Flask, request

# 本地模块
from myproject import utils
from myproject.models import User
from . import local_module
```

### import风格
```python
# 推荐：每个模块单独一行
import os
import sys

# 不推荐
import os, sys

# 推荐：from导入多个名称可合并
from subprocess import Popen, PIPE

# 推荐：绝对导入优先
from mypackage import module

# 相对导入（仅用于包内部）
from . import sibling
from ..parent import something
```

### import检查清单
- [ ] import按标准库→第三方→本地分组
- [ ] 组间有空行分隔
- [ ] 每个模块单独一行
- [ ] 使用isort自动格式化

## 4. 空行规范

### 规则
| 位置 | 空行数 |
|------|--------|
| 顶层函数/类定义之间 | 2行 |
| 类中方法之间 | 1行 |
| 函数内逻辑段落之间 | 1行 |
| 文件末尾 | 1行 |
| import分组之间 | 1行 |

### 示例
```python
"""模块文档字符串。"""

import os
import sys


class MyClass:
    """类文档字符串。"""
    
    def method_one(self):
        pass
    
    def method_two(self):
        pass


def top_level_function():
    """函数文档字符串。"""
    
    # 第一逻辑块
    x = 1
    y = 2
    
    # 第二逻辑块
    result = x + y
    return result
```

### 空行检查清单
- [ ] 顶层定义间有2空行
- [ ] 类方法间有1空行
- [ ] 文件末尾有1空行
- [ ] 函数内无多余空行

## 5. 空格规范

### 需要空格的位置
```python
# 逗号、冒号、分号后
my_list = [1, 2, 3]
my_dict = {'key': 'value'}

# 二元运算符两侧
x = y + z
result = (a + b) * (c - d)

# 比较运算符两侧
if x == y:
    pass

# 赋值运算符两侧
x = 5
```

### 不需要空格的位置
```python
# 括号内部两侧
my_list = [1, 2, 3]  # 正确
my_list = [ 1, 2, 3 ]  # 错误

# 函数调用
func(arg1, arg2)  # 正确
func( arg1, arg2 )  # 错误

# 函数参数默认值
def func(arg=default):  # 正确
    pass

def func(arg = default):  # 错误
    pass

# 切片操作
my_list[1:3]  # 正确
my_list[1 : 3]  # 错误
```

### 空格检查清单
- [ ] 运算符两侧有空格
- [ ] 括号内无多余空格
- [ ] 参数默认值等号无空格
- [ ] 切片操作无多余空格

## 6. 换行规则

### 运算符换行
```python
# 推荐：运算符在新行开头
income = (gross_wages
          + taxable_interest
          + (dividends - qualified_dividends)
          - ira_deduction
          - student_loan_interest)

# 不推荐：运算符在行尾
income = (gross_wages +
          taxable_interest +
          (dividends - qualified_dividends) -
          ira_deduction -
          student_loan_interest)
```

### 函数调用换行
```python
# 括号对齐
result = function(
    arg1, arg2,
    arg3, arg4
)

# 或悬挂缩进
result = function(
        arg1, arg2,
        arg3, arg4
)
```

## 7. 注释格式

### 行内注释
```python
x = x + 1  # 补偿边界条件
```
- 与代码至少隔2个空格
- 以#和1个空格开头

### 块注释
```python
# 这是块注释的第一行
# 这是块注释的第二行
# 每行以#和空格开头
```

### 文档字符串（Docstring）
```python
def function(arg1, arg2):
    """函数的简短描述。
    
    更详细的描述信息。
    
    Args:
        arg1: 参数1说明
        arg2: 参数2说明
    
    Returns:
        返回值说明
    """
    pass
```

## 8. 编码声明

Python 3默认UTF-8，无需声明。如需声明：

```python
# -*- coding: utf-8 -*-
```

或

```python
#!/usr/bin/env python3
# coding: utf-8
```

## 9. 格式化工具

### Black（推荐）
```bash
# 安装
pip install black

# 格式化单个文件
black myfile.py

# 格式化整个项目
black .

# 检查模式（不修改）
black --check .
```

特点：
- 不妥协的格式化风格
- 几乎零配置
- 团队统一风格

### autopep8
```bash
# 安装
pip install autopep8

# 格式化
autopep8 --in-place --aggressive myfile.py
```

### isort（import排序）
```bash
# 安装
pip install isort

# 排序import
isort myfile.py

# 检查模式
isort --check-only .
```

### flake8（检查工具）
```bash
# 安装
pip install flake8

# 检查
flake8 myfile.py

# 忽略特定规则
flake8 --ignore=E501,W503 myfile.py
```

## 10. 完整检查清单

### 格式检查
- [ ] 缩进：统一使用4空格
- [ ] 行宽：不超过约定限制
- [ ] 空行：符合规范要求
- [ ] 空格：正确使用位置

### import检查
- [ ] 分组：标准库→第三方→本地
- [ ] 排序：字母顺序
- [ ] 风格：每模块一行

### 代码质量
- [ ] 注释：清晰、有意义
- [ ] 命名：符合PEP 8规范
- [ ] 文档：关键函数有docstring

### 工具配置
- [ ] Black配置：pyproject.toml
- [ ] isort配置：兼容Black
- [ ] flake8配置：.flake8或setup.cfg

## 11. 工具配置示例

### pyproject.toml（Black + isort）
```toml
[tool.black]
line-length = 99
target-version = ['py38', 'py39', 'py310']
include = '\.pyi?$'

[tool.isort]
profile = "black"
line_length = 99
```

### .flake8
```ini
[flake8]
max-line-length = 99
extend-ignore = E203, W503
exclude = .git, __pycache__, build, dist
```

---
**参考文档**：
- [PEP 8 -- Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [Black文档](https://black.readthedocs.io/)
- [isort文档](https://pycqa.github.io/isort/)
