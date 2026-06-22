---
name: python-testability
description: Python编码规范 - 可测试性规范。适用于所有Python代码开发场景：(1) 代码编写时考虑可测试性；(2) 代码审查时检查可测试性；(3) pytest、mock、隔离。触发关键词：Python、可测试、testability、pytest、mock、stub、隔离、单元测试
---

# Python可测试性规范

基于Clean Code指导书

## 核心原则

可测试性好的代码要满足：**可隔离、可控制、可观测、可定位**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 高内聚低耦合 |
| 2.1 | 减少全局状态 |
| 2.2 | 支持依赖注入 |
| 3.1 | 结果可观测 |
| 3.2 | 过程可观测 |
| 4.1 | 异常输出适当日志 |

## 依赖注入

```python
# 依赖注入
class OrderService:
    def __init__(self, repository: OrderRepository, gateway: PaymentGateway):
        self._repository = repository
        self._gateway = gateway

# 测试轻松注入mock
def test_create_order():
    mock_repo = MagicMock(spec=OrderRepository)
    mock_gateway = MagicMock(spec=PaymentGateway)
    service = OrderService(mock_repo, mock_gateway)

    service.create_order(request)

    mock_repo.save.assert_called_once()
```

## pytest示例

```python
import pytest

@pytest.fixture
def user_service():
    repo = InMemoryUserRepository()
    return UserService(repo)

def test_create_user(user_service):
    user = user_service.create("Alice", "alice@example.com")
    assert user.name == "Alice"
    assert user_service.find_by_name("Alice") is not None
```

## 详细规范

见 [references/testability.md](references/testability.md)

## 检查清单

- [ ] 类是否单一职责？
- [ ] 是否使用依赖注入？
- [ ] 外部依赖是否可mock？
- [ ] 结果是否方便验证？
- [ ] 关键过程数据是否可查询？
