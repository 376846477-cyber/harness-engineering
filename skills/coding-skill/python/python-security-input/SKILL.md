---
name: python-security-input
description: Python编码规范 - 输入校验与安全规范。适用于所有Python代码开发场景：(1) SQL注入、命令注入防护；(2) 文件路径安全；(3) 模板注入防护；(4) 任何外部输入处理。触发关键词：Python、注入、SQL、injection、exec、命令、安全、校验、validation、输入、input
---

# Python输入校验与安全规范

基于安全编程规范

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 禁止直接拼接SQL |
| 规则1.2 | 禁止使用eval/exec处理不可信输入 |
| 规则1.3 | 禁止直接使用subprocess.shell=True |
| 规则1.4 | 文件路径校验前必须先标准化 |
| 规则1.5 | 禁止直接渲染不可信模板 |

## SQL注入防护

**错误示例**:
```python
query = f"SELECT * FROM users WHERE name = '{username}'"
cursor.execute(query)
```

**正确示例**:
```python
cursor.execute("SELECT * FROM users WHERE name = %s", (username,))
```

## 命令注入防护

**错误示例**:
```python
os.system(f"ls {user_input}")
```

**正确示例**:
```python
import shlex
result = subprocess.run(["ls", user_input], capture_output=True, text=True)
```

## 文件路径安全

```python
from pathlib import Path

base_dir = Path("/safe/directory")
file_path = (base_dir / user_input).resolve()
if not str(file_path).startswith(str(base_dir.resolve())):
    raise SecurityError("路径越界")
```

## 详细规范

见 [references/security-input.md](references/security-input.md)
