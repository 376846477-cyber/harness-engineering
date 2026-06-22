# Python加密安全规范详细参考

## 一、密码哈希与存储

### 1.1 推荐算法

| 算法 | 适用场景 | 安全等级 |
|------|----------|----------|
| bcrypt | 通用密码存储 | 高 |
| argon2 | 高安全要求场景 | 极高 |
| pbkdf2_hmac | 兼容性要求场景 | 中高 |

### 1.2 bcrypt 使用示例

```python
import bcrypt

def hash_password(password: str) -> str:
    salt = bcrypt.gensalt(rounds=12)
    hashed = bcrypt.hashpw(password.encode('utf-8'), salt)
    return hashed.decode('utf-8')

def verify_password(password: str, hashed: str) -> bool:
    return bcrypt.checkpw(password.encode('utf-8'), hashed.encode('utf-8'))

# 使用示例
hashed = hash_password("user_password_123")
is_valid = verify_password("user_password_123", hashed)
```

### 1.3 argon2 使用示例

```python
from argon2 import PasswordHasher

ph = PasswordHasher(
    time_cost=3,
    memory_cost=65536,
    parallelism=4,
    hash_len=32,
    salt_len=16
)

def hash_password_argon2(password: str) -> str:
    return ph.hash(password)

def verify_password_argon2(hash: str, password: str) -> bool:
    try:
        ph.verify(hash, password)
        return True
    except:
        return False
```

### 1.4 passlib 统一接口

```python
from passlib.hash import bcrypt, argon2

def hash_with_bcrypt(password: str) -> str:
    return bcrypt.using(rounds=12).hash(password)

def hash_with_argon2(password: str) -> str:
    return argon2.using(
        memory_cost=65536,
        time_cost=3,
        parallelism=4
    ).hash(password)
```

### 1.5 禁止使用的算法

- MD5（用于密码存储）- 已被破解
- SHA1（用于密码存储）- 碰撞攻击可行
- 简单哈希（无盐值）- 易受彩虹表攻击

---

## 二、随机数生成

### 2.1 secrets 模块（推荐）

```python
import secrets

# 生成安全令牌
token_hex = secrets.token_hex(32)
token_url = secrets.token_urlsafe(32)
token_bytes = secrets.token_bytes(32)

# 生成安全随机整数
random_int = secrets.randbelow(1000)
random_bits = secrets.randbits(256)

# 安全选择
items = ['a', 'b', 'c', 'd']
choice = secrets.choice(items)

# API密钥生成
def generate_api_key() -> str:
    return f"sk_{secrets.token_urlsafe(32)}"
```

### 2.2 禁止事项

```python
import random
import os

# 禁止：使用random模块生成安全随机数
random.random()  # 不安全！
random.randint(0, 100)  # 不安全！
random.choice(items)  # 不安全！

# 禁止：使用时间戳作为随机种子
seed = int(time.time())  # 不安全！

# 允许：使用os.urandom作为备选
random_bytes = os.urandom(32)
```

---

## 三、对称加密

### 3.1 Fernet 对称加密（cryptography库）

```python
from cryptography.fernet import Fernet

# 生成密钥
key = Fernet.generate_key()
cipher = Fernet(key)

# 加密
def encrypt_data(data: str) -> bytes:
    return cipher.encrypt(data.encode('utf-8'))

# 解密
def decrypt_data(encrypted: bytes) -> str:
    return cipher.decrypt(encrypted).decode('utf-8')

# 使用示例
encrypted = encrypt_data("敏感数据")
decrypted = decrypt_data(encrypted)
```

### 3.2 AES-256-GCM 加密

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

def encrypt_aes_gcm(plaintext: bytes, key: bytes) -> tuple:
    aesgcm = AESGCM(key)
    nonce = os.urandom(12)
    ciphertext = aesgcm.encrypt(nonce, plaintext, None)
    return (nonce, ciphertext)

def decrypt_aes_gcm(nonce: bytes, ciphertext: bytes, key: bytes) -> bytes:
    aesgcm = AESGCM(key)
    return aesgcm.decrypt(nonce, ciphertext, None)

# 密钥生成（32字节 = AES-256）
key = os.urandom(32)
nonce, encrypted = encrypt_aes_gcm(b"敏感数据", key)
decrypted = decrypt_aes_gcm(nonce, encrypted, key)
```

### 3.3 禁止模式

- ECB模式 - 相同明文产生相同密文
- 无认证加密 - 易受篡改攻击

---

## 四、非对称加密（RSA）

### 4.1 RSA 密钥生成与加解密

```python
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.backends import default_backend

# 生成密钥对
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048,
    backend=default_backend()
)
public_key = private_key.public_key()

# 加密
def rsa_encrypt(plaintext: bytes, pub_key) -> bytes:
    ciphertext = pub_key.encrypt(
        plaintext,
        padding.OAEP(
            mgf=padding.MGF1(algorithm=hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )
    return ciphertext

# 解密
def rsa_decrypt(ciphertext: bytes, priv_key) -> bytes:
    plaintext = priv_key.decrypt(
        ciphertext,
        padding.OAEP(
            mgf=padding.MGF1(algorithm=hashes.SHA256()),
            algorithm=hashes.SHA256(),
            label=None
        )
    )
    return plaintext

# 密钥序列化
def export_private_key_pem(priv_key) -> bytes:
    return priv_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    )

def export_public_key_pem(pub_key) -> bytes:
    return pub_key.public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    )
```

### 4.2 RSA 签名与验证

```python
def sign_data(data: bytes, priv_key) -> bytes:
    signature = priv_key.sign(
        data,
        padding.PSS(
            mgf=padding.MGF1(hashes.SHA256()),
            salt_length=padding.PSS.MAX_LENGTH
        ),
        hashes.SHA256()
    )
    return signature

def verify_signature(data: bytes, signature: bytes, pub_key) -> bool:
    try:
        pub_key.verify(
            signature,
            data,
            padding.PSS(
                mgf=padding.MGF1(hashes.SHA256()),
                salt_length=padding.PSS.MAX_LENGTH
            ),
            hashes.SHA256()
        )
        return True
    except:
        return False
```

---

## 五、SSL/TLS 安全

### 5.1 requests 安全配置

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.ssl_ import create_urllib3_context

class TLSAdapter(HTTPAdapter):
    def init_poolmanager(self, *args, **kwargs):
        context = create_urllib3_context()
        context.minimum_version = ssl.TLSVersion.TLSv1_2
        context.set_ciphers('ECDHE+AESGCM:DHE+AESGCM')
        kwargs['ssl_context'] = context
        return super().init_poolmanager(*args, **kwargs)

# 使用示例
session = requests.Session()
session.mount('https://', TLSAdapter())
response = session.get('https://api.example.com', verify=True)

# 禁止：禁用证书验证
requests.get(url, verify=False)  # 禁止！
```

### 5.2 urllib3 安全配置

```python
import urllib3
import ssl

# 创建安全连接池
http = urllib3.PoolManager(
    cert_reqs='CERT_REQUIRED',
    ca_certs='/path/to/ca-bundle.crt'
)

# 禁用不安全警告（不推荐）
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)  # 仅调试用
```

### 5.3 ssl 模块配置

```python
import ssl

def create_secure_ssl_context() -> ssl.SSLContext:
    context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    context.minimum_version = ssl.TLSVersion.TLSv1_2
    context.verify_mode = ssl.CERT_REQUIRED
    context.check_hostname = True
    context.set_ciphers('ECDHE+AESGCM:DHE+AESGCM')
    context.load_default_certs()
    return context

# 禁用协议
# 禁止：TLS 1.0, TLS 1.1
# 推荐：TLS 1.2, TLS 1.3
```

---

## 六、密钥管理

### 6.1 环境变量存储

```python
import os
from dotenv import load_dotenv

load_dotenv()

# 读取密钥
API_KEY = os.getenv('API_KEY')
DATABASE_PASSWORD = os.getenv('DATABASE_PASSWORD')
SECRET_KEY = os.getenv('SECRET_KEY')

# 带默认值（非敏感配置）
DEBUG = os.getenv('DEBUG', 'false').lower() == 'true'

# 验证必需的密钥
def validate_required_secrets():
    required = ['API_KEY', 'DATABASE_PASSWORD', 'SECRET_KEY']
    missing = [k for k in required if not os.getenv(k)]
    if missing:
        raise ValueError(f"缺少必需的环境变量: {missing}")
```

### 6.2 keyring 安全存储

```python
import keyring

# 存储密钥
keyring.set_password("myapp", "database_user", "db_password")
keyring.set_password("myapp", "api_key", "sk_xxxx")

# 读取密钥
db_password = keyring.get_password("myapp", "database_user")
api_key = keyring.get_password("myapp", "api_key")

# 删除密钥
keyring.delete_password("myapp", "database_user")
```

### 6.3 禁止硬编码密钥

```python
# 禁止：硬编码密钥
API_KEY = "sk_live_xxxxxxxxxxxx"  # 禁止！
DB_PASSWORD = "password123"  # 禁止！
SECRET_KEY = "my-secret-key"  # 禁止！

# 正确：从环境变量读取
API_KEY = os.environ.get('API_KEY')
DB_PASSWORD = os.environ.get('DB_PASSWORD')
SECRET_KEY = os.environ.get('SECRET_KEY')
```

---

## 七、敏感数据清理

### 7.1 内存清理

```python
import ctypes
import secrets

def secure_erase_string(s: str):
    """安全擦除字符串内存"""
    if not isinstance(s, str):
        return
    # Python字符串不可变，但可以覆盖引用
    # 对于密码，建议使用bytearray
    pass

def secure_erase_bytes(data: bytearray):
    """安全擦除字节数组"""
    for i in range(len(data)):
        data[i] = 0
    # 防止编译器优化
    ctypes.memset(id(data), 0, len(data))

# 使用bytearray存储敏感数据
password = bytearray(b"my_secure_password")
try:
    process_password(password)
finally:
    secure_erase_bytes(password)
```

### 7.2 日志脱敏

```python
import re

def mask_sensitive_data(text: str, patterns: dict) -> str:
    """脱敏敏感数据"""
    for name, pattern in patterns.items():
        if name == 'credit_card':
            text = re.sub(pattern, r'\1****\2', text)
        elif name == 'email':
            text = re.sub(pattern, r'\1***@\2', text)
        elif name == 'phone':
            text = re.sub(pattern, r'\1****\2', text)
    return text

# 脱敏模式
PATTERNS = {
    'credit_card': r'(\d{4})\d{8,11}(\d{4})',
    'email': r'(\w{1,3})\w*@(\w+\.\w+)',
    'phone': r'(\d{3})\d{4}(\d{4})'
}

# 日志输出示例
log_text = "用户 credit_card=1234567890123456 登录"
safe_log = mask_sensitive_data(log_text, PATTERNS)
# 输出: 用户 credit_card=1234****3456 登录
```

---

## 八、安全检查清单

### 8.1 密码存储检查

- [ ] 使用bcrypt/argon2/pbkdf2进行密码哈希
- [ ] 哈希使用随机盐值（自动生成）
- [ ] 设置足够的工作因子（bcrypt rounds >= 12）
- [ ] 禁止明文存储密码
- [ ] 禁止使用MD5/SHA1存储密码

### 8.2 随机数检查

- [ ] 安全场景使用secrets模块
- [ ] 禁止使用random模块生成安全随机数
- [ ] 令牌长度 >= 32字节
- [ ] 使用secrets.compare_digest进行常量时间比较

### 8.3 加密检查

- [ ] 使用AES-256-GCM或Fernet
- [ ] 禁止ECB模式
- [ ] RSA密钥长度 >= 2048位
- [ ] 使用OAEP填充而非PKCS1v15
- [ ] 每次加密使用新的nonce/IV

### 8.4 SSL/TLS检查

- [ ] 最小版本 >= TLS 1.2
- [ ] 启用证书验证（verify=True）
- [ ] 检查主机名（check_hostname=True）
- [ ] 禁用不安全密码套件
- [ ] 禁止verify=False

### 8.5 密钥管理检查

- [ ] 密钥存储在环境变量或密钥管理服务
- [ ] 禁止硬编码密钥、密码
- [ ] .env文件添加到.gitignore
- [ ] 实施密钥轮换策略
- [ ] 生产环境不使用测试密钥

### 8.6 代码审查要点

```python
# 检测硬编码密钥的正则模式
HARDCODED_PATTERNS = [
    r'(?i)(password|passwd|pwd)\s*=\s*["\']',
    r'(?i)(api_key|apikey|secret)\s*=\s*["\']',
    r'(?i)(token|auth)\s*=\s*["\'][a-zA-Z0-9]{20,}',
    r'-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----',
]
```

---

## 九、常用库版本要求

| 库 | 推荐版本 | 说明 |
|----|----------|------|
| cryptography | >= 41.0.0 | 主流加密库 |
| bcrypt | >= 4.0.0 | 密码哈希 |
| argon2-cffi | >= 21.0.0 | 最安全密码哈希 |
| passlib | >= 1.7.4 | 统一密码哈希接口 |
| pyjwt | >= 2.8.0 | JWT令牌 |
| secrets | 内置 | 安全随机数 |
| keyring | >= 24.0.0 | 系统密钥存储 |

---

## 十、参考文献

- OWASP Password Storage Cheat Sheet
- NIST SP 800-63B Digital Identity Guidelines
- Python cryptography documentation
- OWASP Cryptographic Storage Cheat Sheet
