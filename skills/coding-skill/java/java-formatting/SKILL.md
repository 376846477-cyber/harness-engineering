---
name: java-formatting
description: Java编码规范 - 排版格式规范。适用于所有Java代码开发场景：(1) 编写Java代码时的格式检查；(2) 代码审查时检查缩进、大括号；(3) 格式化代码；(4) switch/枚举格式。触发关键词：Java、格式、缩进、大括号、格式化、format、indentation、brace
---

# Java排版格式规范

基于Java语言通用编程规范V3"排版格式"

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则3.1 | 左大括号不换行，右大括号独立一行 |
| 规则3.2 | 4空格缩进，制表符不用于缩进 |
| 规则3.3 | 推荐行宽不超过120字符 |
| 规则3.4 | 方法参数逗号后换行 |
| 规则3.5 | 关键字后、运算符前后加空格 |

## 格式示例

### 大括号
```java
public class UserService {
    public void doSomething() {
        if (condition) {
            // code
        }
    }
}
```

### 缩进
```java
// 4空格缩进
public void method() {
    if (condition) {
        doSomething();
    }
}
```

### 链式调用
```java
User user = userService.getUserById(id)
    .setStatus(UserStatus.ACTIVE)
    .save();
```

## 详细格式规范

见 [references/formatting.md](references/formatting.md)