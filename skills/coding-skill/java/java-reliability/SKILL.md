---
name: java-reliability
description: Java编码规范 - 可靠性规范。适用于所有Java代码开发场景：(1) 代码编写时考虑可靠性；(2) 代码审查时检查可靠性；(3) 防御式编程、容错、自愈机制实现；(4) 资源管理、异常处理。触发关键词：Java、可靠、reliability、容错、自愈、防御式编程、资源泄露、故障恢复
---

# Java可靠性规范

基于Clean Code指导书第2章"可靠性"特征

## 核心原则

复杂系统即使经过严格测试，也很难保证没有问题，所以为了降低系统故障的概率，系统必须预备**预防错误、容错、故障恢复（自愈）**的能力。要在设计阶段识别系统潜在的故障场景，并根据故障模式的严重程度、出现的概率进行排序，挑选风险较高的故障模式分析预防、容错、故障恢复措施。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 操作防呆设计 |
| 1.2 | 系统过载保护（负载均衡、流控） |
| 2.1 | 消息重复发送（通信可靠性） |
| 2.2 | 状态定时同步 |
| 3.1 | 防御式编程 |
| 3.2 | 资源管理（获取→使用→释放） |
| 3.3 | 核心流程依赖最小化 |
| 3.4 | 流程异常降级处理 |
| 3.5 | 子系统崩溃处理（主备倒换） |

## 防御式编程

```java
// ✅ 参数校验
public void processData(String input) {
    if (input == null) {
        throw new IllegalArgumentException("输入不能为空");
    }
    input = input.trim();
    if (input.isEmpty()) {
        throw new IllegalArgumentException("输入不能为空字符串");
    }
    if (input.length() > MAX_LENGTH) {
        throw new IllegalArgumentException("输入长度超出限制");
    }
    doProcess(input);
}

// ✅ 资源管理使用try-with-resources
public String readFile(String path) throws IOException {
    try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
        return reader.lines().collect(Collectors.joining("\n"));
    }
}
```

## 降级处理

```java
// ✅ 多层降级
public PaymentResult pay(Order order) {
    try {
        return doPay(order);
    } catch (PaymentException e) {
        log.error("支付失败，尝试降级处理", e);
        return tryFallbackPayment(order);
    }
}

private PaymentResult tryFallbackPayment(Order order) {
    try {
        return fallbackGateway.pay(order);
    } catch (Exception e) {
        return PaymentResult.pending(order.getId(), "请稍后重试支付");
    }
}
```

## 详细规范

见 [references/reliability.md](references/reliability.md)

## 检查清单

- [ ] 关键操作是否有防呆设计？
- [ ] 是否有流控和负载均衡机制？
- [ ] 通信失败是否有重试机制？
- [ ] 状态是否需要同步？
- [ ] 资源是否正确管理（获取→使用→释放）？
- [ ] 核心流程依赖是否最小化？
- [ ] 是否有降级处理策略？
- [ ] 子系统故障是否可检测和恢复？