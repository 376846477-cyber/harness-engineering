---
name: java-maintainability
description: Java编码规范 - 可维护性规范。适用于所有Java代码开发场景：(1) 代码设计时考虑可维护性；(2) 代码审查时检查可维护性；(3) 重构代码提升可维护性；(4) 抽象、依赖、耦合优化。触发关键词：Java、可维护、maintainability、抽象、接口、依赖、耦合、复用、开闭原则
---

# Java可维护性规范

基于Clean Code指导书第2章"可维护"特征

## 核心原则

可维护是指软件可修改、可扩展和可复用的能力。修改/扩展代码要影响局部化，即尽量减小影响的范围，应对的方法是**抽象**，通过抽象带来的相对稳定的间接层来隔离变化。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 类型抽象（泛型） |
| 1.2 | 实现抽象出接口（封装） |
| 1.3 | 抽象构建框架，实现延迟到子类（多态） |
| 2.1 | 接口最小化/参数最少化 |
| 3.1 | 依赖用户角度定义的接口 |
| 3.2 | 子类不破坏父类接口契约（里氏替换） |
| 4.1 | 适配器模式 |
| 4.2 | 生产消费者模式 |
| 4.3 | 观察者模式 |
| 5.1 | 继承 vs 组合（优先组合） |
| 5.2 | 提取公共代码 |

## 接口抽象

```java
// ✅ 定义稳定的接口
public interface PaymentGateway {
    PaymentResult pay(PaymentRequest request);
    RefundResult refund(RefundRequest request);
}

// 多种实现
public class AlipayGateway implements PaymentGateway { }
public class WechatPayGateway implements PaymentGateway { }

// 依赖抽象
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}

// ❌ 直接依赖具体实现
public class OrderService {
    private final AlipayGateway alipayGateway;  // 紧耦合
}
```

## 泛型抽象

```java
// ✅ 泛型抽象 - 算法复用
public interface Repository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    T save(T entity);
}

public class UserRepository implements Repository<User, Long> { }
public class OrderRepository implements Repository<Order, String> { }
```

## 详细规范

见 [references/maintainability.md](references/maintainability.md)

## 检查清单

- [ ] 是否通过接口抽象隔离了变化？
- [ ] 接口是否遵循单一职责（方法数量合理）？
- [ ] 是否依赖抽象而非具体实现？
- [ ] 子类是否能替换父类而不破坏契约？
- [ ] 是否有重复代码需要提取？
- [ ] 继承关系是否符合is-a关系（优先用组合）？
- [ ] 依赖方向是否稳定（依赖抽象而非实现）？