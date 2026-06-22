---
name: python-security-crypto
description: Python编码规范 - 加密安全规范。适用于所有Python代码开发场景：(1) 密码哈希和存储；(2) 加密算法选择；(3) 敏感信息硬编码检测；(4) SSL/TLS安全。触发关键词：Python、加密、encrypt、密码、password、hashlib、cryptography、SSL、TLS、密钥、key
---

# Python加密安全规范

基于安全编程规范

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 禁止在日志中保存口令、密钥 |
| 规则1.2 | 禁止使用弱加密算法（MD5/SHA1存密码） |
| 规则1.3 | 密码存储必须加盐 |
| 规则1.4 | 禁止将敏感信息硬编码 |
| 规则1.5 | 安全场景必须使用secrets模块 |

## 密码存储

**正确示例**:
```python
import hashlib
import os

def hash_password(password: str) -> str:
    salt = os.urandom(32)
    key = hashlib.pbkdf2_hmac("sha256", password.encode(), salt, 100000)
    return salt.hex() + key.hex()

# 或使用bcrypt
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt())
```

## 敏感信息

**错误示例**:
```python
API_KEY = "sk-xxxxxx"        # 硬编码
DB_PASSWORD = "admin123"      # 硬编码
```

**正确示例**:
```python
import os
API_KEY = os.environ["API_KEY"]
DB_PASSWORD = os.environ["DB_PASSWORD"]
```

## 详细规范

见 [references/security-crypto.md](references/security-crypto.md)
