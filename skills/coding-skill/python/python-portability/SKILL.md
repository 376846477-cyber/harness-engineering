---
name: python-portability
description: Python编码规范 - 可移植性规范。适用于所有Python代码开发场景：(1) 代码编写时考虑可移植性；(2) 代码审查时检查可移植性；(3) 跨平台实现。触发关键词：Python、可移植、portability、平台、跨平台、编码、pathlib
---

# Python可移植性规范

基于Clean Code指导书

## 核心原则

Python本身具有良好的跨平台性，但编码时仍需注意避免引入平台相关的依赖。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 使用pathlib处理路径 |
| 1.2 | 明确指定字符编码 |
| 1.3 | 使用os.path或pathlib替代硬编码分隔符 |
| 1.4 | 注意Python版本兼容性 |
| 2.1 | 使用跨平台依赖库 |
| 3.1 | 时区使用UTC或明确指定 |

## 路径处理

```python
from pathlib import Path

# 使用pathlib替代硬编码路径
config_path = Path("data") / "files" / "config.txt"
content = config_path.read_text(encoding="utf-8")

# 临时目录
import tempfile
with tempfile.NamedTemporaryFile(mode="w", encoding="utf-8", delete=False) as f:
    f.write(content)
    temp_path = f.name
```

## 字符编码

```python
# 明确指定编码
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()

# 写入时也指定编码
with open("output.txt", "w", encoding="utf-8") as f:
    f.write(content)
```

## 详细规范

见 [references/portability.md](references/portability.md)

## 检查清单

- [ ] 路径是否使用pathlib？
- [ ] 文件操作是否指定编码？
- [ ] 依赖库是否跨平台？
- [ ] 时间处理是否使用UTC？
- [ ] Python版本是否兼容？
