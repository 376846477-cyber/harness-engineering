---
name: java-readability
description: Java编码规范 - 可读性规范。适用于所有Java代码开发场景：(1) 代码编写时提升可读性；(2) 代码审查时检查可读性；(3) 重构代码使其更易读；(4) 命名、格式、注释优化。触发关键词：Java、可读、readability、易读、理解、阅读、理解难度
---

# Java可读性规范

基于Clean Code指导书第2章"可读"特征

## 核心原则

可读性本质上类似通信里面的信息传递，即作者要将想到的关键信息准确完整的传递给读者，并且降低读者理解的难度。提升可读性就是要：**传递足够多的有用信息、减少信息失真（误解）、减少信息干扰、降低信息理解难度**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 利用名称传递信息（好名字自注释） |
| 1.2 | 利用格式传递信息（缩进、空行、风格一致） |
| 1.3 | 注释作为代码的补充，优先重构而非注释 |
| 2.1 | 业务术语统一，避免不一致导致误解 |
| 2.2 | 命名准确具体，布尔值用is/has/can/should前缀 |
| 3.1 | 无废弃代码和无效注释 |
| 3.2 | 风格一致，比"正确"更重要 |
| 4.1 | 功能单一，一段代码不同时处理多件事 |
| 4.2 | 简化语句逻辑，使用保护语句减少嵌套 |
| 4.3 | 变量作用域尽量小 |
| 4.4 | 名字不要过长（过长需拆分） |

## 命名规范

```java
// ✅ 函数名体现行为和意图
public User findUserById(Long userId) { }
public boolean validatePassword(String raw, String encrypted) { }

// ✅ 变量名带重要细节后缀
long timeoutMs = 5000;
BigDecimal amountInCents = new BigDecimal("9999");

// ❌ 名字无法传递信息
public Object get(Object o) { }
int x;  // 作用域大但名字太短
```

## 简化嵌套

```java
// ✅ 使用保护语句减少嵌套
public User getUser(Long userId) {
    if (userId == null) return null;
    User user = userRepository.findById(userId);
    if (user == null) return null;
    if (!user.isActive()) return null;
    return user;
}

// ❌ 嵌套过深
public User getUser(Long userId) {
    if (userId != null) {
        User user = userRepository.findById(userId);
        if (user != null) {
            if (user.isActive()) { return user; }
        }
    }
    return null;
}
```

## 详细规范

见 [references/readability.md](references/readability.md)

## 检查清单

- [ ] 名称是否传递了足够的信息？
- [ ] 名称是否有二义性？
- [ ] 代码格式是否一致？
- [ ] 是否有废弃代码和无效注释？
- [ ] 函数是否功能单一？
- [ ] 嵌套层级是否过深（>3层）？
- [ ] 变量作用域是否尽可能小？
- [ ] 名字长度是否合适？