# Python序列化安全规范详细参考

## 1. pickle反序列化危险

### 1.1 安全风险

pickle模块可以序列化任意Python对象，反序列化时会执行`__reduce__`方法中定义的代码，导致任意代码执行漏洞。

```python
# 危险示例：恶意pickle数据可执行任意命令
import pickle

# 攻击者构造的恶意数据
malicious_data = b"cos\nsystem\n(S'rm -rf /'\ntR."

# 反序列化时会执行删除命令
pickle.loads(malicious_data)  # 危险！
```

### 1.2 安全规范

```python
# 错误：反序列化不可信数据
data = pickle.loads(untrusted_network_data)  # 禁止！

# 正确：使用安全的替代方案
import json
data = json.loads(untrusted_network_data)  # 推荐

# 如必须使用pickle，需验证数据来源
import hmac
import hashlib

def safe_unpickle(data: bytes, signature: bytes, secret_key: bytes):
    """带签名验证的安全反序列化"""
    expected_sig = hmac.new(secret_key, data, hashlib.sha256).digest()
    if not hmac.compare_digest(signature, expected_sig):
        raise ValueError("Invalid signature")
    return pickle.loads(data)  # 签名验证通过后才反序列化
```

## 2. JSON安全处理

### 2.1 基本安全原则

```python
import json

# JSON本身安全，但需注意：
# 1. 深度限制 - 防止栈溢出攻击
# 2. 数值精度 - 大整数可能丢失精度
# 3. 键值数量 - 防止内存耗尽

def safe_json_loads(data: str, max_depth: int = 20, max_size: int = 10 * 1024 * 1024):
    """安全的JSON解析"""
    if len(data) > max_size:
        raise ValueError(f"Data size exceeds limit: {max_size}")
    
    result = json.loads(data)
    
    # 检查嵌套深度
    def check_depth(obj, current_depth=0):
        if current_depth > max_depth:
            raise ValueError(f"JSON depth exceeds limit: {max_depth}")
        if isinstance(obj, dict):
            for v in obj.values():
                check_depth(v, current_depth + 1)
        elif isinstance(obj, list):
            for item in obj:
                check_depth(item, current_depth + 1)
    
    check_depth(result)
    return result
```

### 2.2 大整数精度问题

```python
# 问题：JSON对大整数支持有限
import json

large_num = {"id": 9007199254740993}  # 超过JavaScript安全整数范围
data = json.dumps(large_num)
parsed = json.loads(data)
print(parsed["id"])  # 可能变成 9007199254740992

# 解决方案：使用字符串表示
def safe_json_dumps(obj):
    """将大整数转为字符串"""
    def convert_bigints(o):
        if isinstance(o, dict):
            return {k: convert_bigints(v) for k, v in o.items()}
        elif isinstance(o, list):
            return [convert_bigints(i) for i in o]
        elif isinstance(o, int) and abs(o) > 2**53:
            return str(o)
        return o
    return json.dumps(convert_bigints(obj))
```

## 3. YAML安全加载

### 3.1 安全风险

```python
# 危险：yaml.load() 可执行任意Python代码
import yaml

malicious_yaml = """
!!python/object/apply:os.system
args: ['rm -rf /']
"""

data = yaml.load(malicious_yaml)  # 危险！会执行命令
```

### 3.2 安全规范

```python
import yaml

# 正确：始终使用 safe_load
data = yaml.safe_load(untrusted_yaml_data)  # 推荐

# 或明确指定 SafeLoader
data = yaml.load(untrusted_yaml_data, Loader=yaml.SafeLoader)

# 禁止使用的加载器
# yaml.load(data)                      # 禁止！
# yaml.load(data, Loader=yaml.Loader)  # 禁止！
# yaml.load(data, Loader=yaml.UnsafeLoader)  # 禁止！
# yaml.unsafe_load(data)               # 禁止！

# PyYAML 5.1+ 版本要求必须指定Loader
# 使用 full_load 加载标准YAML类型（不含Python对象）
data = yaml.load(data, Loader=yaml.FullLoader)  # 比SafeLoader稍宽松
```

## 4. msgpack安全

### 4.1 基本使用

```python
import msgpack

# msgpack 相对安全，不会自动执行代码
data = msgpack.unpackb(binary_data, raw=False)  # raw=False 防止bytes转str问题

# 安全配置
def safe_msgpack_unpack(data: bytes, max_depth: int = 5, max_len: int = 1000000):
    """安全的msgpack解析"""
    return msgpack.unpackb(
        data,
        raw=False,           # 将bytes解码为str
        max_depth=max_depth, # 限制嵌套深度
        max_len=max_len,     # 限制列表/字典长度
        strict_map_key=True  # 禁止非字符串键
    )
```

### 4.2 注意事项

```python
# 注意：msgpack-ext可能导致问题
# 禁止使用 use_bin_type=False (旧版本兼容模式)
# 禁止反序列化未知的extension类型

def safe_msgpack_unpack_strict(data: bytes):
    result = msgpack.unpackb(data, raw=False)
    
    # 检查可疑内容
    def validate(obj):
        if isinstance(obj, dict):
            for k, v in obj.items():
                if not isinstance(k, str):
                    raise ValueError("Non-string key detected")
                validate(v)
        elif isinstance(obj, list):
            for item in obj:
                validate(item)
        elif isinstance(obj, bytes):
            # 检查是否包含可疑二进制数据
            if len(obj) > 10 * 1024 * 1024:
                raise ValueError("Binary data too large")
    
    validate(result)
    return result
```

## 5. 数据校验

### 5.1 Schema验证

```python
from typing import Dict, Any, List
import jsonschema

# 定义数据模式
USER_SCHEMA = {
    "type": "object",
    "properties": {
        "name": {"type": "string", "minLength": 1, "maxLength": 100},
        "age": {"type": "integer", "minimum": 0, "maximum": 150},
        "email": {"type": "string", "format": "email"},
        "tags": {
            "type": "array",
            "items": {"type": "string"},
            "maxItems": 20
        }
    },
    "required": ["name", "email"],
    "additionalProperties": False
}

def validate_and_parse(data: str, schema: dict) -> Dict[str, Any]:
    """验证并解析数据"""
    try:
        obj = json.loads(data)
        jsonschema.validate(obj, schema)
        return obj
    except json.JSONDecodeError as e:
        raise ValueError(f"Invalid JSON: {e}")
    except jsonschema.ValidationError as e:
        raise ValueError(f"Schema validation failed: {e.message}")
```

### 5.2 类型安全解析

```python
from dataclasses import dataclass
from typing import Optional
import json

@dataclass
class User:
    name: str
    age: int
    email: str
    tags: Optional[List[str]] = None
    
    @classmethod
    def from_json(cls, data: str) -> 'User':
        """从JSON安全创建对象"""
        obj = json.loads(data)
        
        if not isinstance(obj, dict):
            raise ValueError("Expected JSON object")
        
        name = obj.get("name")
        if not isinstance(name, str) or not name:
            raise ValueError("Invalid or missing 'name'")
        
        age = obj.get("age", 0)
        if not isinstance(age, int) or age < 0:
            raise ValueError("Invalid 'age'")
        
        email = obj.get("email")
        if not isinstance(email, str) or "@" not in email:
            raise ValueError("Invalid 'email'")
        
        tags = obj.get("tags", [])
        if not isinstance(tags, list):
            raise ValueError("Invalid 'tags'")
        
        return cls(name=name, age=age, email=email, tags=tags)
```

## 6. 版本兼容

### 6.1 版本化数据格式

```python
import json
from typing import Any, Dict

VERSION = 2  # 当前数据格式版本

def serialize_with_version(data: Dict[str, Any]) -> str:
    """带版本信息的序列化"""
    envelope = {
        "version": VERSION,
        "data": data,
        "timestamp": time.time()
    }
    return json.dumps(envelope)

def deserialize_with_version(data: str) -> Dict[str, Any]:
    """带版本兼容的反序列化"""
    envelope = json.loads(data)
    version = envelope.get("version", 1)
    payload = envelope.get("data", envelope)  # 兼容旧格式
    
    # 版本迁移
    if version == 1:
        payload = migrate_v1_to_v2(payload)
    elif version > VERSION:
        raise ValueError(f"Unsupported version: {version}")
    
    return payload

def migrate_v1_to_v2(data: Dict) -> Dict:
    """V1到V2的迁移逻辑"""
    if "fullName" in data:
        data["name"] = data.pop("fullName")
    return data
```

### 6.2 向后兼容检查

```python
def check_compatibility(serialized_data: bytes) -> bool:
    """检查数据兼容性"""
    try:
        obj = json.loads(serialized_data)
        
        # 检查必需字段
        required_fields = ["id", "type"]
        for field in required_fields:
            if field not in obj:
                return False
        
        # 检查版本范围
        version = obj.get("version", 1)
        if version < 1 or version > VERSION:
            return False
        
        return True
    except (json.JSONDecodeError, TypeError):
        return False
```

## 7. 敏感数据处理

### 7.1 序列化前脱敏

```python
import json
import re
from typing import Any, Dict, Set

SENSITIVE_FIELDS = {
    "password", "passwd", "pwd", "secret", "token", 
    "api_key", "apikey", "private_key", "credit_card",
    "ssn", "phone", "email"
}

def mask_sensitive(value: str) -> str:
    """脱敏处理"""
    if len(value) <= 4:
        return "****"
    return value[:2] + "*" * (len(value) - 4) + value[-2:]

def sanitize_for_serialization(obj: Any, sensitive_fields: Set[str] = None) -> Any:
    """序列化前清理敏感数据"""
    fields = sensitive_fields or SENSITIVE_FIELDS
    
    if isinstance(obj, dict):
        result = {}
        for key, value in obj.items():
            key_lower = key.lower()
            if any(s in key_lower for s in fields):
                if isinstance(value, str):
                    result[key] = mask_sensitive(value)
                else:
                    result[key] = "[REDACTED]"
            else:
                result[key] = sanitize_for_serialization(value, fields)
        return result
    elif isinstance(obj, list):
        return [sanitize_for_serialization(item, fields) for item in obj]
    else:
        return obj

# 使用示例
user_data = {
    "name": "Alice",
    "email": "alice@example.com",
    "password": "secret123",
    "api_key": "sk-1234567890abcdef"
}

safe_data = sanitize_for_serialization(user_data)
print(json.dumps(safe_data))
# {"name": "Alice", "email": "al****@example.com", "password": "se****23", "api_key": "sk****ef"}
```

### 7.2 使用dataclass标记敏感字段

```python
from dataclasses import dataclass, field
from typing import Optional
import json

def transient(value: Any) -> Any:
    """标记为瞬态字段，不参与序列化"""
    return field(default=value, repr=False, compare=False)

@dataclass
class UserCredentials:
    username: str
    password: str = transient("")  # 不序列化
    token: Optional[str] = transient(None)  # 不序列化
    
    def to_dict(self) -> dict:
        """安全导出，排除敏感字段"""
        return {
            "username": self.username
            # password和token被排除
        }
    
    def to_json(self) -> str:
        return json.dumps(self.to_dict())
```

### 7.3 加密后序列化

```python
from cryptography.fernet import Fernet
import json
import base64

class SecureSerializer:
    """加密序列化器"""
    
    def __init__(self, key: bytes):
        self.cipher = Fernet(key)
    
    def serialize(self, obj: Any) -> bytes:
        """加密后序列化"""
        json_data = json.dumps(obj)
        return self.cipher.encrypt(json_data.encode())
    
    def deserialize(self, data: bytes) -> Any:
        """解密后反序列化"""
        json_data = self.cipher.decrypt(data).decode()
        return json.loads(json_data)

# 使用示例
key = Fernet.generate_key()
serializer = SecureSerializer(key)

sensitive_data = {"password": "secret", "api_key": "key123"}
encrypted = serializer.serialize(sensitive_data)
decrypted = serializer.deserialize(encrypted)
```

## 8. 安全检查清单

### 8.1 pickle使用检查

- [ ] 是否对不可信数据使用了pickle.loads()
- [ ] 是否需要签名验证才能使用pickle
- [ ] 是否考虑了JSON/msgpack等安全替代方案
- [ ] 是否在文档中标注了pickle数据的可信来源

### 8.2 YAML使用检查

- [ ] 是否使用yaml.safe_load()而非yaml.load()
- [ ] 是否显式指定了SafeLoader
- [ ] 是否避免了yaml.UnsafeLoader
- [ ] PyYAML版本是否>=5.1

### 8.3 JSON使用检查

- [ ] 是否设置了JSON深度限制
- [ ] 是否处理了大整数精度问题
- [ ] 是否限制了最大JSON大小
- [ ] 是否验证了JSON schema

### 8.4 敏感数据检查

- [ ] 是否在序列化前脱敏处理
- [ ] 是否标记了敏感字段为transient
- [ ] 是否在日志中过滤了敏感数据
- [ ] 敏感数据是否需要加密存储

### 8.5 版本兼容检查

- [ ] 序列化数据是否包含版本信息
- [ ] 是否实现了版本迁移逻辑
- [ ] 是否处理了旧版本数据的兼容性
- [ ] 是否有数据格式变更的测试

## 9. 其他序列化格式

| 格式 | 安全性 | 推荐场景 | 注意事项 |
|------|--------|----------|----------|
| JSON | 安全 | Web API、配置文件 | 大整数精度、深度限制 |
| YAML | 需配置 | 配置文件 | 必须用safe_load |
| msgpack | 安全 | 高性能二进制传输 | 限制深度和长度 |
| pickle | 危险 | Python内部缓存 | 仅限可信数据 |
| marshal | 危险 | Python字节码 | 禁止用于不可信数据 |
| shelve | 危险 | 本地持久化 | 基于pickle，同样危险 |
| protobuf | 安全 | 跨语言RPC | 需定义schema |
| toml | 安全 | Python配置 | 使用tomllib(tomli) |

## 10. 快速参考

```python
# 安全序列化决策树
def safe_serialize(data: Any, untrusted: bool = True) -> str:
    if untrusted:
        # 对于不可信数据，只能用JSON/YAML safe_load/msgpack
        return json.dumps(data)
    else:
        # 可信数据可以用pickle，但仍推荐JSON
        return json.dumps(data)

def safe_deserialize(data: bytes, format: str, untrusted: bool = True) -> Any:
    if format == "json":
        return safe_json_loads(data.decode())
    elif format == "yaml":
        return yaml.safe_load(data.decode())
    elif format == "msgpack":
        return safe_msgpack_unpack(data)
    elif format == "pickle" and not untrusted:
        return pickle.loads(data)  # 仅限可信数据
    else:
        raise ValueError(f"Unsupported format: {format}")
```

---
**版本**: 2.0  
**更新日期**: 2025-05-28  
**适用Python版本**: 3.8+
