---
name: python-maintainability
description: Python编码规范 - 可维护性规范。适用于所有Python代码开发场景：(1) 代码设计时考虑可维护性；(2) 代码审查时检查可维护性；(3) 重构代码提升可维护性。触发关键词：Python、可维护、maintainability、抽象、接口、依赖、耦合、复用、SOLID
---

# Python可维护性规范

基于Clean Code指导书

## 核心原则

可维护是指软件可修改、可扩展和可复用的能力。修改/扩展代码要影响局部化，应对的方法是**抽象**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 使用ABC/Protocol定义抽象 |
| 1.2 | 依赖注入而非硬编码依赖 |
| 1.3 | 遵循SOLID原则 |
| 2.1 | 接口最小化 |
| 3.1 | 优先组合而非继承 |
| 4.1 | 使用dataclass简化数据类 |
| 5.1 | 提取公共代码（DRY） |

## 接口抽象

```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def pay(self, request: PaymentRequest) -> PaymentResult:
        ...

    @abstractmethod
    def refund(self, request: RefundRequest) -> RefundResult:
        ...

# 依赖抽象
class OrderService:
    def __init__(self, payment_gateway: PaymentGateway):
        self._payment_gateway = payment_gateway
```

## 详细规范

见 [references/maintainability.md](references/maintainability.md)

## 检查清单

- [ ] 是否通过抽象隔离了变化？
- [ ] 是否依赖抽象而非具体实现？
- [ ] 是否有重复代码需要提取？
- [ ] 继承关系是否符合is-a关系？
- [ ] 是否遵循单一职责原则？
