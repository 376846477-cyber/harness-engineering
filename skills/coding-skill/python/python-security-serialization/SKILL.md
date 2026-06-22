---
name: python-security-serialization
description: Python编码规范 - 序列化安全规范。适用于所有Python代码开发场景：(1) pickle序列化/反序列化安全；(2) YAML安全加载；(3) JSON安全处理；(4) 敏感数据保护。触发关键词：Python、序列化、pickle、YAML、json、deserialize、marshal
---

# Python序列化安全规范

基于安全编程规范

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 禁止对不可信数据使用pickle.loads |
| 规则1.2 | YAML加载必须使用safe_load |
| 规则1.3 | 禁止序列化未加密的敏感数据 |
| 规则1.4 | 禁止使用marshal处理不可信数据 |

## pickle安全

**错误示例**:
```python
import pickle
data = pickle.loads(untrusted_data)  # 危险！可执行任意代码
```

**正确示例**:
```python
import json
data = json.loads(untrusted_data)    # 安全，仅解析数据
```

## YAML安全

**错误示例**:
```python
import yaml
data = yaml.load(untrusted_data)     # 危险！默认允许任意对象
```

**正确示例**:
```python
import yaml
data = yaml.safe_load(untrusted_data)  # 安全，仅解析基本类型
```

## 详细规范

见 [references/security-serialization.md](references/security-serialization.md)
