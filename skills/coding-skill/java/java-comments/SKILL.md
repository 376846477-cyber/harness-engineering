---
name: java-comments
description: Java编码规范 - 注释规范。适用于所有Java代码开发场景：(1) 编写任何Java类的Javadoc；(2) 代码审查时检查注释规范；(3) 添加方法注释、字段注释；(4) 文件头版权声明。触发关键词：Java、注释、Javadoc、comment、doc、文档、注释语言
---

# Java注释规范

基于Java语言通用编程规范V2"注释"

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则2.1 | 公开API使用Javadoc |
| 规则2.2 | 文件头包含版权、功能、作者 |
| 规则2.3 | 方法头包含功能、参数、返回值、异常 |
| 规则2.4 | 注释语言中文 |

## Javadoc要求

### 文件头注释
```java
/**
 * Copyright (C) 公司版权所有
 *
 * 功能：提供用户管理的核心服务
 *
 * 作者：张三 2024-01-01
 */
public class UserService {}
```

### 方法注释
```java
/**
 * 根据用户ID获取用户信息
 *
 * @param userId 用户ID
 * @return 用户信息，不存在返回null
 * @throws UserNotFoundException 用户不存在时抛出
 */
public User getUserById(Long userId) {}
```

## 注释规范

见 [references/comments.md](references/comments.md)