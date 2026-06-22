---
name: java-testability
description: Java编码规范 - 可测试性规范。适用于所有Java代码开发场景：(1) 代码编写时考虑可测试性；(2) 代码审查时检查可测试性；(3) 提升代码可测试性的重构；(4) 隔离、可控、可观测实现。触发关键词：Java、可测试、testability、单元测试、mock、stub、隔离、测试
---

# Java可测试性规范

基于Clean Code指导书第2章"可测试"特征

## 核心原则

复杂系统由于外部输入组合场景太多，所以全系统测试很难测试充分，而且全系统测试的成本也很高。采用分治法，把复杂系统的各个子系统、模块、子功能隔离测试，隔离测试完后再集成测试，可以大大降低测试复杂度。可测试性好的代码，要满足：**可隔离、可控制、可观测、可定位**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 高内聚低耦合（单一职责） |
| 2.1 | 减少系统状态和全局变量 |
| 2.2 | 支持依赖注入 |
| 2.3 | 减少输入数量（使用参数对象） |
| 3.1 | 结果可观测 |
| 3.2 | 过程可观测（保留过程数据） |
| 4.1 | 异常输出适当日志 |

## 依赖注入

```java
// ✅ 构造函数注入依赖
public class OrderService {
    private final OrderRepository repository;
    private final PaymentGateway paymentGateway;

    public OrderService(OrderRepository repository, PaymentGateway paymentGateway) {
        this.repository = repository;
        this.paymentGateway = paymentGateway;
    }
}

// 测试轻松注入mock
@Test
void testCreateOrder() {
    OrderRepository mockRepo = mock(OrderRepository.class);
    PaymentGateway mockPayment = mock(PaymentGateway.class);
    OrderService service = new OrderService(mockRepo, mockPayment);

    Order result = service.createOrder(request);

    assertNotNull(result);
    verify(mockRepo).save(any(Order.class));
}

// ❌ 静态全局依赖难以测试
public class OrderService {
    public Order createOrder(OrderRequest request) {
        Connection conn = DatabaseUtils.getConnection();  // 静态全局
    }
}
```

## 接口抽象便于Mock

```java
// ✅ 接口抽象
public interface Clock {
    long currentTimeMillis();
}

// 测试注入固定时间
@Test
void testGenerateToken() {
    Clock fixedClock = () -> 1000000L;
    TokenService tokenService = new TokenService(fixedClock);

    String token = tokenService.generateToken(user);

    assertEquals("1:1360000", token);
}
```

## 详细规范

见 [references/testability.md](references/testability.md)

## 检查清单

- [ ] 类是否单一职责（高内聚低耦合）？
- [ ] 是否使用依赖注入（避免静态全局依赖）？
- [ ] 外部依赖是否抽象成接口便于mock？
- [ ] 输入参数是否过多（考虑参数对象）？
- [ ] 结果是否方便验证？
- [ ] 关键过程数据是否可查询？
- [ ] 异常是否有适当的日志记录？