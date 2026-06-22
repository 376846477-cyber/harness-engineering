---
name: java-naming
description: Java编码规范 - 命名规范。适用于所有Java代码开发场景：(1) 编写任何Java类、方法、变量时的命名检查；(2) 代码审查时检查命名规范；(3) 重构时确保命名符合规范；(4) 解释命名规则。触发关键词：Java、class、method、variable、命名、驼峰、常量、identifier、naming、camelCase
---

# Java命名规范

基于Java语言通用编程规范V4.7第1章"命名"

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 源文件编码UTF-8，空格仅0x20 |
| 规则1.2 | 标识符仅ASCII字母数字下划线，长度2-64 |
| 规则1.3 | 包名小写，com开头 |
| 规则1.4 | 类/接口大驼峰，测试类加Test |
| 规则1.5 | 方法小驼峰 |

## 命名风格对照

| 类型 | 风格 | 示例 |
|-----|------|-----|
| 类/接口/枚举 | 大驼峰 | `UserService`, `XmlParser` |
| 方法/变量/参数 | 小驼峰 | `getUserById()`, `userName` |
| 常量 | 全大写下划线 | `MAX_CONNECTIONS` |
| 枚举值 | 全大写下划线 | `STATUS_SUCCESS` |
| 泛型类型 | 单大写字母 | `E`, `T`, `K`, `V` |

## 常见错误

**❌ 错误示例**:
```java
String name_;           // 不应有下划线后缀
int mCount;             // 不应有m前缀
class data {}           // 类名应大驼峰
void TYPE() {}          // 方法应小驼峰
static final int COUNT = 10;  // 常量应全大写
```

**✅ 正确示例**:
```java
String userName;
int recordCount;
class UserService {}
void getUserInfo() {}
static final int MAX_RECORDS = 10;
```

## 详细规范

见 [references/naming.md](references/naming.md)