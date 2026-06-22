# Python可移植性规范详细参考

## 一、使用标准库优先

### 核心原则
标准库是Python可移植性的基石，经过全面测试且跨平台兼容。

### 推荐使用的标准库模块

| 功能领域 | 推荐模块 | 说明 |
|---------|---------|------|
| 路径操作 | pathlib, os.path | 跨平台路径处理 |
| 文件操作 | shutil, tempfile | 高级文件操作 |
| 进程管理 | subprocess | 跨平台进程调用 |
| 编码处理 | codecs, locale | 字符编码处理 |
| 配置管理 | configparser, argparse | 配置和参数解析 |
| 日志记录 | logging | 标准日志系统 |
| 并发处理 | threading, multiprocessing, concurrent.futures | 多线程/多进程 |
| 网络操作 | socket, ssl, urllib | 网络通信 |
| 日期时间 | datetime, zoneinfo | 时间处理（Python 3.9+） |

### 示例：使用标准库替代第三方依赖

```python
# 推荐：使用标准库
import json
import configparser
from pathlib import Path
from datetime import datetime, timezone

# 读取JSON配置
config_path = Path("config") / "settings.json"
config = json.loads(config_path.read_text(encoding="utf-8"))

# 不推荐：使用平台可能不支持的第三方库
# import some_platform_specific_lib
```

## 二、避免平台特定代码

### 识别平台特定代码

```python
import sys
import platform
import os

def detect_platform():
    """检测当前平台"""
    return {
        "system": platform.system(),        # Windows/Linux/Darwin
        "machine": platform.machine(),      # x86_64/AMD64/ARM64
        "python": sys.version_info,
    }
```

### 条件导入平台特定模块

```python
import sys

if sys.platform == "win32":
    import msvcrt
    def get_char():
        return msvcrt.getch().decode("utf-8")
elif sys.platform.startswith("linux"):
    import tty
    import termios
    def get_char():
        fd = sys.stdin.fileno()
        old_settings = termios.tcgetattr(fd)
        try:
            tty.setraw(sys.stdin.fileno())
            ch = sys.stdin.read(1)
        finally:
            termios.tcsetattr(fd, termios.TCSADRAIN, old_settings)
        return ch
else:
    def get_char():
        return input("Enter a character: ")[0]
```

### 避免硬编码平台常量

```python
# 错误：硬编码路径分隔符
config_path = "config" + "\\" + "settings.json"  # Windows only

# 正确：使用pathlib
from pathlib import Path
config_path = Path("config") / "settings.json"

# 错误：硬编码换行符
content = "line1\r\nline2\r\n"  # Windows换行

# 正确：使用os.linesep或文本模式
import os
content = f"line1{os.linesep}line2{os.linesep}"
```

## 三、路径处理（pathlib）

### pathlib最佳实践

```python
from pathlib import Path

# 构建路径
config_dir = Path.home() / ".config" / "myapp"
data_file = Path("data") / "users" / "profile.json"

# 读取文件
content = data_file.read_text(encoding="utf-8")

# 写入文件
data_file.write_text(json.dumps(data), encoding="utf-8")

# 检查路径
if config_dir.exists() and config_dir.is_dir():
    print(f"Config directory: {config_dir}")

# 遍历目录
for file in Path("src").rglob("*.py"):
    print(file.relative_to(Path.cwd()))

# 处理临时文件
import tempfile
with tempfile.TemporaryDirectory() as tmpdir:
    tmp_file = Path(tmpdir) / "cache.txt"
    tmp_file.write_text("temporary data", encoding="utf-8")
```

### 路径转换

```python
from pathlib import Path, PurePosixPath, PureWindowsPath

# 转换为字符串（跨平台安全）
path_str = str(Path("data/file.txt"))

# 获取POSIX格式路径（用于URL或配置）
posix_path = Path("data/file.txt").as_posix()

# 解析用户目录
home = Path.home()
documents = home / "Documents"

# 解析相对路径
relative = Path("src/../config/./settings.json").resolve()
```

## 四、编码处理

### 文件读写编码规范

```python
from pathlib import Path
import json
import csv

# 文本文件读写
text_content = Path("data.txt").read_text(encoding="utf-8")
Path("output.txt").write_text(text_content, encoding="utf-8")

# CSV文件处理
with open("data.csv", "r", encoding="utf-8", newline="") as f:
    reader = csv.DictReader(f)
    rows = list(reader)

with open("output.csv", "w", encoding="utf-8", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["name", "value"])
    writer.writeheader()
    writer.writerows(data)

# JSON文件处理
with open("config.json", "r", encoding="utf-8") as f:
    config = json.load(f)

with open("output.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
```

### 字符串编码转换

```python
# 字符串转字节
text = "中文内容"
bytes_data = text.encode("utf-8")

# 字节转字符串
decoded = bytes_data.decode("utf-8")

# 处理未知编码
import chardet
with open("unknown.txt", "rb") as f:
    raw = f.read()
    detected = chardet.detect(raw)
    text = raw.decode(detected["encoding"])
```

## 五、环境变量处理

### 跨平台环境变量操作

```python
import os
from pathlib import Path

# 读取环境变量（提供默认值）
db_host = os.getenv("DB_HOST", "localhost")
db_port = int(os.getenv("DB_PORT", "5432"))
debug = os.getenv("DEBUG", "false").lower() in ("true", "1", "yes")

# 注意：Windows环境变量不区分大小写，Linux区分
# 使用统一的命名约定
api_key = os.getenv("API_KEY") or os.getenv("api_key")

# 设置环境变量（仅当前进程）
os.environ["APP_MODE"] = "production"

# 路径类环境变量处理
python_path = os.getenv("PYTHONPATH", "")
path_list = [Path(p) for p in python_path.split(os.pathsep) if p]

# 跨平台HOME目录
home = Path.home()
config_home = Path(os.getenv("XDG_CONFIG_HOME", home / ".config"))
```

### 使用python-dotenv管理配置

```python
# pip install python-dotenv
from dotenv import load_dotenv
from pathlib import Path
import os

# 加载.env文件
env_path = Path(".env")
load_dotenv(env_path)

# 读取配置
database_url = os.getenv("DATABASE_URL")
```

## 六、版本兼容（__future__）

### 使用__future__实现向后兼容

```python
# Python 3.7+ 兼容的异步语法
from __future__ import annotations  # Python 3.7+
from __future__ import generator_stop  # Python 3.5+

# 启用延迟注解评估（支持循环引用）
from __future__ import annotations
from typing import Optional

class Node:
    def __init__(self, value: int, next_node: Optional[Node] = None):
        self.value = value
        self.next_node = next_node
```

### 声明Python版本要求

```toml
# pyproject.toml
[project]
name = "myapp"
version = "1.0.0"
requires-python = ">=3.8,<4.0"

[project.optional-dependencies]
dev = ["pytest>=7.0", "black>=23.0"]
```

```python
# 代码中检查版本
import sys

if sys.version_info < (3, 8):
    raise RuntimeError("This application requires Python 3.8 or later")

# 使用hasattr检查API可用性
if hasattr(sys, "orig_argv"):
    original_args = sys.orig_argv
```

## 七、跨平台库推荐

### 文件与路径

| 功能 | 推荐库 | 说明 |
|-----|-------|------|
| 高级路径操作 | pathlib (标准库) | 面向对象路径 |
| 文件监控 | watchdog | 跨平台文件事件 |
| 配置管理 | dynaconf, python-dotenv | 多格式配置 |

### 网络与并发

| 功能 | 推荐库 | 说明 |
|-----|-------|------|
| HTTP客户端 | httpx, requests | 跨平台HTTP |
| 异步框架 | asyncio (标准库), trio | 异步编程 |
| 任务队列 | dramatiq, huey | 轻量级任务队列 |

### 系统交互

| 功能 | 推荐库 | 说明 |
|-----|-------|------|
| 进程管理 | subprocess (标准库), sh | Shell命令 |
| 系统信息 | psutil | 进程和系统监控 |
| 终端UI | rich, textual | 跨平台终端UI |

### 示例：使用psutil获取系统信息

```python
import psutil

# 获取CPU使用率
cpu_percent = psutil.cpu_percent(interval=1)

# 获取内存信息
memory = psutil.virtual_memory()

# 获取磁盘使用
disk = psutil.disk_usage("/")

# 获取网络接口
net_if_addrs = psutil.net_if_addrs()
```

## 八、虚拟环境

### 使用venv管理环境

```bash
# 创建虚拟环境
python -m venv .venv

# Windows激活
.venv\Scripts\activate

# Linux/macOS激活
source .venv/bin/activate

# 安装依赖
pip install -r requirements.txt

# 导出依赖
pip freeze > requirements.txt
```

### 使用poetry管理项目

```bash
# 安装poetry
pip install poetry

# 初始化项目
poetry new myproject
cd myproject

# 添加依赖
poetry add requests
poetry add --group dev pytest

# 安装项目依赖
poetry install

# 运行命令
poetry run python main.py
```

### pyproject.toml完整示例

```toml
[tool.poetry]
name = "myapp"
version = "1.0.0"
description = "Cross-platform Python application"
authors = ["Developer <dev@example.com>"]
requires-python = ">=3.8,<4.0"

[tool.poetry.dependencies]
python = "^3.8"
requests = "^2.28"
python-dotenv = "^1.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0"
black = "^23.0"
mypy = "^1.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

## 九、可移植性检查清单

### 代码层面

- [ ] 所有文件读写都指定`encoding="utf-8"`
- [ ] 使用`pathlib.Path`处理所有路径
- [ ] 避免硬编码路径分隔符（`\`或`/`）
- [ ] 使用`os.linesep`或文本模式处理换行
- [ ] 环境变量读取使用`os.getenv()`并提供默认值
- [ ] 条件导入平台特定模块
- [ ] 使用`sys.platform`而非`os.name`判断平台
- [ ] 明确声明`requires-python`版本

### 测试层面

- [ ] 在Windows和Linux上运行测试
- [ ] 测试路径包含空格和Unicode字符
- [ ] 测试不同locale环境下的编码
- [ ] 验证临时文件的正确清理
- [ ] 测试信号处理的跨平台兼容性

### 部署层面

- [ ] 使用虚拟环境隔离依赖
- [ ] 锁定依赖版本（requirements.txt或poetry.lock）
- [ ] 避免依赖系统级Python包
- [ ] 使用Docker实现环境一致性
- [ ] 编写跨平台的部署脚本

## 十、常见陷阱与解决方案

### 换行符问题

```python
# 问题：不同系统换行符不同
# Windows: \r\n, Linux: \n, macOS(旧): \r

# 解决：使用文本模式或universal newlines
with open("data.txt", "r", encoding="utf-8", newline="") as f:
    content = f.read()

# 写入时让Python处理换行
with open("output.txt", "w", encoding="utf-8", newline="") as f:
    f.write("line1\nline2\n")  # Python自动转换
```

### 文件权限问题

```python
import os
from pathlib import Path

# Windows不支持完整的Unix权限
# 但可以使用os.chmod设置基本权限

file_path = Path("script.sh")

# 设置可执行权限（Unix）
if sys.platform != "win32":
    file_path.chmod(0o755)

# Windows需要检查扩展名
if sys.platform == "win32":
    # Windows通过文件扩展名判断可执行
    print(file_path.suffix in (".exe", ".bat", ".cmd"))
```

### 子进程调用

```python
import subprocess
import sys

# 跨平台安全方式
result = subprocess.run(
    ["python", "-c", "print('Hello')"],
    capture_output=True,
    text=True,
    check=True,
)

# 避免使用shell=True（安全风险）
# 错误：
# subprocess.run(f"echo {user_input}", shell=True)  # 命令注入风险

# 正确：
subprocess.run(["echo", user_input], text=True)
```

### 信号处理差异

```python
import signal
import sys

def signal_handler(signum, frame):
    print("Received signal:", signum)

# SIGINT (Ctrl+C) - 跨平台
signal.signal(signal.SIGINT, signal_handler)

# SIGTERM - Windows支持有限
if sys.platform != "win32":
    signal.signal(signal.SIGTERM, signal_handler)

# Windows特有信号
if sys.platform == "win32":
    signal.signal(signal.SIGBREAK, signal_handler)
```
