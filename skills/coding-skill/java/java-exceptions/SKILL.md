---
name: java-exceptions
description: Java编码规范 - 异常处理规范。适用于所有Java代码开发场景：(1) 编写try-catch代码；(2) 代码审查时检查异常处理；(3) 异常抛出和日志记录；(4) 自定义异常类。触发关键词：Java、异常、try、catch、throw、exception、日志、log、error
---

# Java异常处理规范

基于Java语言通用编程规范V7"异常和日志"

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则7.1 | 不捕获通用Exception/RuntimeException |
| 规则7.2 | 异常处理要具体精确 |
| 规则7.3 | 不要忽略异常，至少记录日志 |
| 规则7.4 | 使用日志框架非System.out/err |
| 规则7.5 | 异常信息不含敏感数据 |

## 异常处理示例

**❌ 错误示例**:
```java
try {
    // operation
} catch (Exception e) {  // 捕获太宽泛
    // 空的异常处理
}

throw new Exception("错误");  // 异常类型不具体
```

**✅ 正确示例**:
```java
try {
    // operation
} catch (IOException e) {
    log.error("读取文件失败", e);
    throw new FileReadException("文件读取失败", e);
} catch (SQLException e) {
    log.error("数据库操作失败", e);
    throw new DataAccessException("数据访问失败", e);
}
```

## 日志规范

- 使用logback/log4j等日志框架
- ERROR级别记录业务异常
- WARN级别记录可恢复异常
- DEBUG级别记录调试信息

## 详细异常规范

见 [references/exceptions.md](references/exceptions.md)