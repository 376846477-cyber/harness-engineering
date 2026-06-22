# Python可测试性规范详细参考

## 1. 测试框架选择

### 1.1 pytest框架（推荐）

pytest是Python最流行的测试框架，具有简洁语法和强大功能。

```python
# test_example.py

def add(a, b):
    return a + b

def test_add_positive_numbers():
    assert add(2, 3) == 5

def test_add_negative_numbers():
    assert add(-1, -1) == -2

def test_add_zero():
    assert add(5, 0) == 5
```

**优势：**
- 简洁的断言语法（使用assert）
- 自动发现测试
- 丰富的插件生态
- 详细的失败信息

### 1.2 unittest框架（标准库）

```python
import unittest

class TestCalculator(unittest.TestCase):
    
    def setUp(self):
        self.calc = Calculator()
    
    def tearDown(self):
        pass
    
    def test_add(self):
        self.assertEqual(self.calc.add(2, 3), 5)
    
    def test_divide_by_zero(self):
        with self.assertRaises(ValueError):
            self.calc.divide(10, 0)

if __name__ == '__main__':
    unittest.main()
```

### 1.3 测试目录结构

```
project/
├── src/
│   └── mymodule/
│       ├── __init__.py
│       └── calculator.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # 共享fixture
│   ├── test_calculator.py
│   └── test_integration/
│       └── test_api.py
├── pytest.ini
└── requirements-test.txt
```

## 2. Mock使用

### 2.1 unittest.mock基础

```python
from unittest.mock import Mock, MagicMock, patch

# Mock对象
mock_db = Mock()
mock_db.query.return_value = [{'id': 1, 'name': 'test'}]
result = mock_db.query('SELECT * FROM users')

# 验证调用
mock_db.query.assert_called_once_with('SELECT * FROM users')

# MagicMock（支持魔术方法）
mock_list = MagicMock()
mock_list.__len__.return_value = 5
assert len(mock_list) == 5
```

### 2.2 patch装饰器

```python
from unittest.mock import patch

class TestExternalAPI:
    
    @patch('mymodule.requests.get')
    def test_fetch_data(self, mock_get):
        mock_get.return_value.json.return_value = {'data': 'test'}
        
        result = fetch_data('http://api.example.com')
        
        assert result == {'data': 'test'}
        mock_get.assert_called_once_with('http://api.example.com')
    
    # 上下文管理器方式
    def test_with_context(self):
        with patch('mymodule.external_call') as mock_call:
            mock_call.return_value = 'mocked'
            result = process_data()
            assert result == 'processed: mocked'
```

### 2.3 pytest-mock插件

```python
# 使用pytest-mock的mocker fixture
def test_database_query(mocker):
    mock_cursor = mocker.Mock()
    mock_cursor.execute.return_value = None
    mock_cursor.fetchall.return_value = [(1, 'Alice')]
    
    mocker.patch('myapp.get_cursor', return_value=mock_cursor)
    
    result = query_users()
    assert result == [(1, 'Alice')]
```

## 3. 依赖注入

### 3.1 构造函数注入

```python
# 不推荐：硬编码依赖
class UserService:
    def __init__(self):
        self.db = Database()  # 硬编码依赖
    
    def get_user(self, user_id):
        return self.db.query(f"SELECT * FROM users WHERE id={user_id}")

# 推荐：依赖注入
class UserService:
    def __init__(self, db):
        self.db = db  # 通过构造函数注入
    
    def get_user(self, user_id):
        return self.db.query(f"SELECT * FROM users WHERE id={user_id}")

# 测试
def test_get_user():
    mock_db = Mock()
    mock_db.query.return_value = {'id': 1, 'name': 'Alice'}
    
    service = UserService(mock_db)
    result = service.get_user(1)
    
    assert result == {'id': 1, 'name': 'Alice'}
```

### 3.2 工厂模式注入

```python
class DataProcessor:
    def __init__(self, db_factory=None):
        self.db_factory = db_factory or self._default_db_factory
    
    def _default_db_factory(self):
        return Database()
    
    def process(self, data):
        db = self.db_factory()
        return db.save(data)

# 测试
def test_processor():
    mock_db = Mock()
    factory = lambda: mock_db
    
    processor = DataProcessor(db_factory=factory)
    processor.process({'key': 'value'})
    
    mock_db.save.assert_called_once_with({'key': 'value'})
```

## 4. Fixture使用

### 4.1 基础fixture

```python
# conftest.py
import pytest

@pytest.fixture
def sample_data():
    return {'id': 1, 'name': 'test'}

@pytest.fixture
def mock_database():
    db = Mock()
    db.connect.return_value = True
    return db

# test_example.py
def test_with_fixture(sample_data):
    assert sample_data['id'] == 1

def test_with_multiple_fixtures(sample_data, mock_database):
    assert mock_database.connect() is True
    assert sample_data['name'] == 'test'
```

### 4.2 fixture作用域

```python
@pytest.fixture(scope='function')  # 默认，每个测试函数执行一次
def fresh_db():
    return create_test_db()

@pytest.fixture(scope='class')  # 每个测试类执行一次
def class_db():
    db = create_test_db()
    yield db
    db.close()

@pytest.fixture(scope='module')  # 每个模块执行一次
def module_config():
    return load_config()

@pytest.fixture(scope='session')  # 整个测试会话执行一次
def global_client():
    client = TestClient()
    yield client
    client.close()
```

### 4.3 fixture清理

```python
@pytest.fixture
def temp_file():
    import tempfile
    import os
    
    fd, path = tempfile.mkstemp()
    os.close(fd)
    
    yield path  # 测试使用path
    
    # 清理
    if os.path.exists(path):
        os.remove(path)
```

## 5. 测试替身

### 5.1 Dummy（哑对象）

```python
# 仅用于填充参数，不被使用
class DummyLogger:
    def info(self, msg): pass
    def error(self, msg): pass

def test_process():
    processor = Processor(logger=DummyLogger())
    result = processor.run()
    assert result is not None
```

### 5.2 Stub（桩对象）

```python
# 返回预设值
class StubDatabase:
    def query(self, sql):
        return [{'id': 1, 'name': 'stubbed'}]

def test_get_users():
    repo = UserRepository(db=StubDatabase())
    users = repo.get_all()
    assert len(users) == 1
```

### 5.3 Spy（间谍对象）

```python
class SpyEmailService:
    def __init__(self):
        self.sent_emails = []
    
    def send(self, to, subject, body):
        self.sent_emails.append({'to': to, 'subject': subject})

def test_send_notification():
    spy = SpyEmailService()
    notifier = Notifier(email_service=spy)
    
    notifier.notify('user@example.com', 'Hello')
    
    assert len(spy.sent_emails) == 1
    assert spy.sent_emails[0]['to'] == 'user@example.com'
```

### 5.4 Mock（模拟对象）

```python
# 验证行为
from unittest.mock import Mock

def test_service_call():
    mock_api = Mock()
    service = MyService(api=mock_api)
    
    service.process('data')
    
    mock_api.send.assert_called_once()
    mock_api.send.assert_called_with('data')
```

### 5.5 Fake（假对象）

```python
# 使用简化实现替代真实组件
class FakeDatabase:
    def __init__(self):
        self.data = {}
    
    def save(self, key, value):
        self.data[key] = value
    
    def get(self, key):
        return self.data.get(key)

def test_repository():
    repo = Repository(db=FakeDatabase())
    repo.save('key1', 'value1')
    
    assert repo.get('key1') == 'value1'
```

## 6. 参数化测试

### 6.1 pytest.mark.parametrize

```python
@pytest.mark.parametrize('input,expected', [
    (2, 4),
    (3, 9),
    (4, 16),
    (5, 25),
])
def test_square(input, expected):
    assert square(input) == expected

# 多参数组合
@pytest.mark.parametrize('x', [1, 2, 3])
@pytest.mark.parametrize('y', [10, 20])
def test_multiply(x, y):
    assert multiply(x, y) == x * y
```

### 6.2 参数化测试类

```python
@pytest.mark.parametrize('value,expected', [
    ('valid@email.com', True),
    ('invalid-email', False),
    ('@missing-local.com', False),
    ('missing-at.com', False),
    ('', False),
])
class TestEmailValidator:
    def test_validate(self, value, expected):
        validator = EmailValidator()
        assert validator.is_valid(value) == expected
```

### 6.3 使用pytest.param添加标记

```python
@pytest.mark.parametrize('input,expected', [
    pytest.param(1, 2, id='positive'),
    pytest.param(-1, -2, id='negative'),
    pytest.param(0, 0, id='zero'),
    pytest.param('a', 'aa', 
                 marks=pytest.mark.xfail(reason='字符串不支持')),
])
def test_double(input, expected):
    assert double(input) == expected
```

## 7. 覆盖率工具（pytest-cov）

### 7.1 安装和运行

```bash
# 安装
pip install pytest-cov

# 运行测试并生成覆盖率报告
pytest --cov=myapp tests/

# 生成HTML报告
pytest --cov=myapp --cov-report=html tests/

# 生成XML报告（CI集成）
pytest --cov=myapp --cov-report=xml tests/
```

### 7.2 配置文件

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*

# 覆盖率配置
addopts = --cov=myapp --cov-report=term-missing --cov-fail-under=80
```

### 7.3 .coveragerc配置

```ini
[run]
source = myapp
omit = 
    myapp/tests/*
    myapp/__init__.py
    */migrations/*

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise NotImplementedError
    if __name__ == .__main__.:
    @abstractmethod
fail_under = 80
```

### 7.4 覆盖率目标

| 类型 | 目标覆盖率 | 说明 |
|------|-----------|------|
| 关键业务逻辑 | 100% | 核心功能必须完全覆盖 |
| 工具函数 | 90%+ | 辅助功能高覆盖 |
| API接口 | 95%+ | 所有端点需测试 |
| 异常处理 | 80%+ | 错误路径需覆盖 |
| 整体项目 | 80%+ | 最低标准 |

## 8. 可测试性检查清单

### 8.1 设计阶段

- [ ] 使用依赖注入而非硬编码依赖
- [ ] 避免全局状态和单例模式
- [ ] 函数保持纯函数特性（无副作用）
- [ ] 模块职责单一，易于隔离测试
- [ ] 接口抽象清晰，便于mock

### 8.2 编码阶段

- [ ] 函数参数可mock
- [ ] 外部依赖通过参数或配置注入
- [ ] 避免在函数内部创建难以测试的对象
- [ ] 使用工厂模式创建复杂对象
- [ ] 时间/随机数等不可控因素可注入

### 8.3 测试阶段

- [ ] 每个测试独立运行
- [ ] 测试不依赖执行顺序
- [ ] 使用fixture管理测试数据
- [ ] 清理测试产生的副作用
- [ ] 测试命名清晰描述测试场景

### 8.4 覆盖率检查

- [ ] 行覆盖率达标（80%+）
- [ ] 分支覆盖率达标
- [ ] 关键路径100%覆盖
- [ ] 异常处理路径有测试
- [ ] 边界条件有测试

### 8.5 Mock使用检查

- [ ] 不过度使用mock
- [ ] mock行为与真实实现一致
- [ ] 验证mock调用次数和参数
- [ ] 测试完成后重置mock状态
- [ ] 使用适当类型的测试替身

## 9. 最佳实践示例

### 9.1 可测试的服务类

```python
# myapp/user_service.py
from typing import Protocol

class DatabaseProtocol(Protocol):
    def query(self, sql: str) -> list: ...
    def execute(self, sql: str) -> None: ...

class UserService:
    def __init__(self, db: DatabaseProtocol, logger=None):
        self.db = db
        self.logger = logger or NullLogger()
    
    def get_user(self, user_id: int) -> dict:
        self.logger.info(f"Getting user {user_id}")
        result = self.db.query(f"SELECT * FROM users WHERE id={user_id}")
        if not result:
            raise ValueError(f"User {user_id} not found")
        return result[0]

class NullLogger:
    def info(self, msg): pass
    def error(self, msg): pass
```

### 9.2 完整的测试示例

```python
# tests/test_user_service.py
import pytest
from unittest.mock import Mock
from myapp.user_service import UserService

class TestUserService:
    
    @pytest.fixture
    def mock_db(self):
        return Mock()
    
    @pytest.fixture
    def mock_logger(self):
        return Mock()
    
    @pytest.fixture
    def service(self, mock_db, mock_logger):
        return UserService(db=mock_db, logger=mock_logger)
    
    def test_get_user_success(self, service, mock_db, mock_logger):
        mock_db.query.return_value = [{'id': 1, 'name': 'Alice'}]
        
        result = service.get_user(1)
        
        assert result == {'id': 1, 'name': 'Alice'}
        mock_db.query.assert_called_once()
        mock_logger.info.assert_called_once()
    
    def test_get_user_not_found(self, service, mock_db):
        mock_db.query.return_value = []
        
        with pytest.raises(ValueError, match="User 1 not found"):
            service.get_user(1)
    
    @pytest.mark.parametrize('user_id,expected_name', [
        (1, 'Alice'),
        (2, 'Bob'),
        (3, 'Charlie'),
    ])
    def test_get_multiple_users(self, service, mock_db, user_id, expected_name):
        mock_db.query.return_value = [{'id': user_id, 'name': expected_name}]
        
        result = service.get_user(user_id)
        
        assert result['name'] == expected_name
```

## 10. 常见反模式

### 10.1 避免过度Mock

```python
# 反模式：过度mock
def test_process_data(mocker):
    mock_a = mocker.Mock()
    mock_b = mocker.Mock()
    mock_c = mocker.Mock()
    # ...过多mock导致测试脆弱

# 推荐：使用真实对象或Fake
def test_process_data():
    fake_db = FakeDatabase()
    processor = DataProcessor(db=fake_db)
    assert processor.run() is not None
```

### 10.2 避免测试私有方法

```python
# 反模式：测试私有方法
def test_private_method():
    obj = MyClass()
    result = obj._internal_calc()  # 不要测试私有方法

# 推荐：测试公开接口
def test_public_method():
    obj = MyClass()
    result = obj.calculate()  # 通过公开方法测试
    assert result > 0
```

### 10.3 避免测试依赖顺序

```python
# 反模式：依赖测试顺序
class TestBad:
    def test_first(self):
        self.data = create_data()
    
    def test_second(self):  # 依赖test_first
        assert self.data is not None

# 推荐：每个测试独立
class TestGood:
    @pytest.fixture
    def data(self):
        return create_data()
    
    def test_first(self, data):
        assert data is not None
    
    def test_second(self, data):
        assert data is not None
```
