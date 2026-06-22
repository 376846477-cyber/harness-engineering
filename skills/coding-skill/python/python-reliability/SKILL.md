---
name: python-reliability
description: Python编码规范 - 可靠性规范。适用于所有Python代码开发场景：(1) 代码编写时考虑可靠性；(2) 代码审查时检查可靠性；(3) 防御式编程、容错、自愈机制实现。触发关键词：Python、可靠、reliability、容错、自愈、防御式编程、资源泄露、故障恢复
---

# Python可靠性规范

基于Clean Code指导书

## 核心原则

系统必须预备**预防错误、容错、故障恢复（自愈）**的能力。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 操作防呆设计 |
| 1.2 | 系统过载保护 |
| 2.1 | 防御式编程 |
| 2.2 | 资源管理（with语句） |
| 2.3 | 核心流程依赖最小化 |
| 2.4 | 降级处理策略 |

## 防御式编程

**错误示例**:
```python
def process_data(data):
    return data["user_id"].upper()
```

**正确示例**:
```python
def process_data(data: dict) -> str:
    if data is None:
        raise ValueError("数据不能为空")

    user_id = data.get("user_id")
    if not user_id:
        raise ValueError("缺少user_id")

    return do_process(user_id)
```

## 资源管理

**错误示例**:
```python
f = open("data.txt", "r", encoding="utf-8")
content = f.read()
f.close()
```

**正确示例**:
```python
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

**错误示例** - 数据库连接未管理:
```python
conn = create_connection(url)
cursor = conn.cursor()
cursor.execute(sql)
conn.close()
```

**正确示例** - 使用contextlib:
```python
from contextlib import contextmanager

@contextmanager
def database_connection(url):
    conn = create_connection(url)
    try:
        yield conn
    finally:
        conn.close()

with database_connection(url) as conn:
    cursor = conn.cursor()
    cursor.execute(sql)
```

## 详细规范

见 [references/reliability.md](references/reliability.md)

## 检查清单

- [ ] 关键操作是否有防呆设计？
- [ ] 资源是否使用with管理？
- [ ] 是否有降级处理策略？
- [ ] 异常是否正确处理？
- [ ] 是否有重试机制处理临时故障？
- [ ] 是否有熔断器防止级联故障？
- [ ] 核心流程是否依赖最小化？
- [ ] 输入参数是否经过校验？
- [ ] 是否有超时设置防止阻塞？
- [ ] 日志是否包含足够上下文信息？
