---
name: cpp-readability
description: C++编码规范 - 可读性规范。适用于所有C++代码开发场景：(1) 代码编写时提升可读性；(2) 代码审查时检查可读性；(3) 重构代码提升可读性。触发关键词：C++、可读、readability、易读、理解、阅读
---

# C++可读性规范

基于Clean Code指导书

## 核心原则

提升可读性就是要：**传递足够多的有用信息、减少信息失真、减少信息干扰、降低信息理解难度**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 利用名称传递信息 |
| 1.2 | 利用格式传递信息 |
| 1.3 | 注释作为代码的补充 |
| 2.1 | 使用现代C++惯用法 |
| 3.1 | 无废弃代码 |
| 4.1 | 功能单一 |
| 4.2 | 简化语句逻辑，使用卫语句 |
| 4.3 | 变量作用域尽量小 |

## 现代C++惯用法

```cpp
// 范围for替代迭代器
for (const auto& user : users) {
    process(user);
}

// auto简化类型
auto it = map.find(key);

// 结构化绑定（C++17）
auto [key, value] = *map.begin();

// std::optional替代指针返回
std::optional<User> find(int id);

// if constexpr替代SFINAE
if constexpr (std::is_integral_v<T>) { ... }
```

## 详细规范

见 [references/readability.md](references/readability.md)

## 检查清单

- [ ] 名称是否传递了足够的信息？
- [ ] 代码是否使用现代C++惯用法？
- [ ] 代码格式是否一致？
- [ ] 是否有废弃代码？
- [ ] 函数是否功能单一？
- [ ] 嵌套层级是否过深？
