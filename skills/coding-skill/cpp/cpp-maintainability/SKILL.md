---
name: cpp-maintainability
description: C++编码规范 - 可维护性规范。适用于所有C++代码开发场景：(1) 代码设计时考虑可维护性；(2) 代码审查时检查可维护性；(3) 重构代码提升可维护性。触发关键词：C++、可维护、maintainability、抽象、接口、依赖、耦合、复用、SOLID
---

# C++可维护性规范

基于Clean Code指导书

## 核心原则

可维护是指软件可修改、可扩展和可复用的能力。修改/扩展代码要影响局部化，应对的方法是**抽象**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 使用纯虚函数/概念定义抽象 |
| 1.2 | 依赖注入而非硬编码依赖 |
| 1.3 | 遵循SOLID原则 |
| 2.1 | 接口最小化 |
| 3.1 | 优先组合而非继承 |
| 4.1 | 提取公共代码（DRY） |
| 5.1 | 优先使用C++20 Concepts替代SFINAE |

## 接口抽象

```cpp
class IPaymentGateway {
public:
    virtual ~IPaymentGateway() = default;
    virtual PaymentResult pay(const PaymentRequest& request) = 0;
    virtual RefundResult refund(const RefundRequest& request) = 0;
};

class OrderService {
public:
    explicit OrderService(std::shared_ptr<IPaymentGateway> gateway)
        : gateway_(std::move(gateway)) {}

private:
    std::shared_ptr<IPaymentGateway> gateway_;
};
```

## 模板元编程

```cpp
#include <concepts>

// C++20 Concepts替代SFINAE
template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

template<Numeric T>
T add(T a, T b) { return a + b; }

// 类型萃取检测成员
template<typename T, typename = void>
struct has_size : std::false_type {};

template<typename T>
struct has_size<T, std::void_t<decltype(std::declval<T>().size())>> 
    : std::true_type {};

// constexpr替代编译期计算
constexpr int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i) result *= i;
    return result;
}
static_assert(factorial(5) == 120);
```

## 详细规范

见 [references/maintainability.md](references/maintainability.md)

## 检查清单

- [ ] 是否通过抽象隔离了变化？
- [ ] 是否依赖抽象而非具体实现？
- [ ] 是否有重复代码需要提取？
- [ ] 继承关系是否符合is-a关系？
- [ ] 是否遵循单一职责原则？
- [ ] 模板元编程是否过度使用？
- [ ] 是否优先使用C++20 Concepts？
