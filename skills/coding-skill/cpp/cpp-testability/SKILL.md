---
name: cpp-testability
description: C++编码规范 - 可测试性规范。适用于所有C++代码开发场景：(1) 代码编写时考虑可测试性；(2) 代码审查时检查可测试性；(3) GTest、GMock、依赖注入。触发关键词：C++、可测试、testability、GTest、GMock、mock、stub、隔离、单元测试
---

# C++可测试性规范

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

```cpp
class OrderService {
public:
    explicit OrderService(
        std::shared_ptr<IOrderRepository> repo,
        std::shared_ptr<IPaymentGateway> gateway
    ) : repo_(std::move(repo))
      , gateway_(std::move(gateway)) {}

    Result create(const OrderRequest& request);

private:
    std::shared_ptr<IOrderRepository> repo_;
    std::shared_ptr<IPaymentGateway> gateway_;
};

// 测试轻松注入mock
TEST(OrderServiceTest, CreateOrder) {
    auto mock_repo = std::make_shared<MockOrderRepository>();
    auto mock_gateway = std::make_shared<MockPaymentGateway>();
    OrderService service(mock_repo, mock_gateway);

    EXPECT_CALL(*mock_repo, save(_)).Times(1);
    service.create(request);
}
```

## 详细规范

见 [references/testability.md](references/testability.md)

## 检查清单

- [ ] 类是否单一职责？
- [ ] 是否使用依赖注入？
- [ ] 外部依赖是否可mock？
- [ ] 结果是否方便验证？
