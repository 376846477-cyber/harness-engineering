# 华为Java异常处理规范详细参考

## 规则7.1 异常捕获原则

- 不捕获通用Exception、RuntimeException
- 捕获具体异常类型
- 异常层次：Error > RuntimeException > Checked Exception

**✅ 正确示例**:
```java
try {
    // 操作
} catch (IOException e) {
    // 处理IO异常
} catch (SQLException e) {
    // 处理SQL异常
}
```

**❌ 错误示例**:
```java
try {
    // 操作
} catch (Exception e) {
    // 太宽泛
}
```

## 规则7.2 异常处理要精确

- 不要捕获后重新抛出相同异常
- 在合适层级处理异常

## 规则7.3 不要忽略异常

- 至少记录日志
- 不能捕获后什么都不做

**❌ 错误示例**:
```java
try {
    // 操作
} catch (IOException e) {
    // 空的
}
```

**✅ 正确示例**:
```java
try {
    // 操作
} catch (IOException e) {
    log.error("读取文件失败: {}", filePath, e);
    throw new FileReadException("文件读取失败", e);
}
```

## 规则7.4 日志规范

- 使用日志框架（logback, log4j）
- 不使用System.out/err
- 正确选择日志级别

| 级别 | 使用场景 |
|-----|---------|
| ERROR | 业务异常，需要处理 |
| WARN | 可恢复异常，警告 |
| INFO | 重要业务事件 |
| DEBUG | 调试信息 |

## 规则7.5 异常信息不含敏感数据

- 不在异常消息中暴露：
  - 用户密码
  - 密钥
  - 完整文件路径
  - 系统配置

```java
// ❌ 错误
throw new Exception("密码错误: " + user.getPassword());

// ✅ 正确
log.warn("用户登录失败: userId={}", userId);
throw new AuthenticationException("用户名或密码错误");
```

## 规则7.6 异常与业务结合

- 使用异常处理业务分支
- 避免异常滥用

## 异常处理检查清单

1. ✅ 捕获具体异常类型
2. ✅ 不捕获通用Exception
3. ✅ 不忽略异常，至少记录日志
4. ✅ 使用日志框架
5. ✅ 异常信息不含敏感数据
6. ✅ 异常消息清晰有意义