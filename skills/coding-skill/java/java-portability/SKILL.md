---
name: java-portability
description: Java编码规范 - 可移植性规范。适用于所有Java代码开发场景：(1) 代码编写时考虑可移植性；(2) 代码审查时检查可移植性；(3) 硬件/OS/编译器解耦实现；(4) 平台无关代码编写。触发关键词：Java、可移植、portability、平台、独立、跨平台、操作系统
---

# Java可移植性规范

基于Clean Code指导书第2章"可移植"特征

## 核心原则

可移植性是指代码在不同硬件、操作系统、编译器环境下运行的能力。Java语言本身具有良好的平台无关性，但编码时仍需注意避免引入平台相关的依赖，确保代码可以在不同环境中运行。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 文件路径使用File.separator或Paths |
| 1.2 | 行分隔符使用System.lineSeparator() |
| 1.3 | 临时目录使用java.io.tmpdir |
| 1.4 | 字符集明确指定UTF-8 |
| 2.1 | 位运算使用Java标准方法 |
| 2.2 | 货币计算使用BigDecimal |
| 2.3 | 字节序使用Java标准类处理 |
| 3.1 | API与JVM版本兼容 |
| 3.2 | 模块系统明确声明依赖 |
| 4.1 | 选择跨平台依赖 |
| 4.2 | 本地库加载考虑多平台 |
| 5.1 | 时区使用UTC或明确指定 |
| 5.2 | Locale明确指定 |

## 路径和分隔符

```java
// ✅ 使用File.separator获取平台分隔符
String path = "data" + File.separator + "files" + File.separator + "config.txt";
Path configPath = Paths.get("data", "files", "config.txt");

// ❌ 硬编码路径分隔符
String path = "data/files/config.txt";      // Linux可以，Windows不行
String path = "data\\files\\config.txt";    // Windows可以，Linux不行

// ✅ 使用系统行分隔符
String line = "line1" + System.lineSeparator() + "line2";
```

## 字符集处理

```java
// ✅ 明确指定字符集
String content = new String(bytes, StandardCharsets.UTF_8);
BufferedReader reader = new BufferedReader(
    new InputStreamReader(new FileInputStream(file), StandardCharsets.UTF_8));
Files.write(path, content.getBytes(StandardCharsets.UTF_8));

// ❌ 使用默认编码
String content = new String(bytes);  // 依赖系统默认编码
```

## 时区处理

```java
// ✅ 存储和传输使用UTC
Instant timestamp = Instant.now();
String utcString = timestamp.toString();  // "2024-01-15T10:30:00Z"

// ✅ 解析时指定时区
ZonedDateTime zdt = ZonedDateTime.parse("2024-01-15T10:30:00+08:00",
    DateTimeFormatter.ISO_DATE_TIME);
```

## 详细规范

见 [references/portability.md](references/portability.md)

## 检查清单

- [ ] 文件路径是否使用了File.separator或Paths？
- [ ] 行分隔符是否使用了System.lineSeparator()？
- [ ] 临时目录是否使用了java.io.tmpdir？
- [ ] 字符集是否明确指定（UTF-8）？
- [ ] 浮点数计算是否使用BigDecimal？
- [ ] API是否与JVM版本兼容？
- [ ] 依赖库是否跨平台？
- [ ] 时间处理是否使用UTC或明确时区？