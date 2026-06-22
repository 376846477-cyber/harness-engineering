# 华为Java输入校验与安全规范详细参考

## 规则1.1 禁止直接边界传递不可信数据

- 所有外部输入都是不可信的
- 必须经过验证才能使用

## 规则1.2 SQL注入防护（关键）

### 禁止
```java
// ❌ 字符串拼接
String sql = "SELECT * FROM users WHERE name = '" + username + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);

// ❌ 使用字符串格式化
String sql = String.format("SELECT * FROM users WHERE name = '%s'", username);
```

### 必须使用
```java
// ✅ 参数化查询
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
stmt.setString(1, username);

// ✅ 存储过程
CallableStatement cstmt = conn.prepareCall("{call getUser(?)}");
cstmt.setString(1, username);

// ✅ 使用ORM框架
User user = userRepository.findByName(username);
```

## 规则1.3 日志注入防护

- 不可信数据不能直接记录日志
- 需要过滤换行符、特殊字符

```java
// ❌ 错误：日志注入
log.info("用户名: " + userInput);

// ✅ 正确：过滤特殊字符
String safeInput = userInput.replaceAll("[\r\n\t]", "");
log.info("用户名: {}", safeInput);

// ✅ 使用参数化日志
log.info("用户登录: userId={}, ip={}", userId, ip);
```

## 规则1.4 格式化字符串安全

- 禁止使用不可信数据构造格式化字符串

```java
// ❌ 错误：Log4j漏洞
logger.info(userInput);  // 可能执行恶意代码

// ✅ 正确：参数化
logger.info("{}", userInput);
logger.info("用户: {}", safeValue);
```

## 规则1.5 命令注入防护

```java
// ❌ 错误
Runtime.getRuntime().exec("ls " + userInput);

// ✅ 正确：使用ProcessBuilder
ProcessBuilder pb = new ProcessBuilder("ls", userInput);
pb.start();

// ✅ 正确：白名单验证
if (!userInput.matches("^[a-zA-Z0-9_]+$")) {
    throw new IllegalArgumentException("无效输入");
}
```

## 规则1.6 文件路径安全

```java
// ✅ 路径标准化
File file = new File(baseDir, userPath);
String canonicalPath = file.getCanonicalPath();

// ✅ 检查路径越界
if (!canonicalPath.startsWith(baseDir)) {
    throw new SecurityException("路径越界");
}
```

## 规则1.7 Zip解压安全

```java
// ✅ 检查解压后大小
ZipEntry entry = zis.getNextEntry();
if (entry.getSize() > MAX_FILE_SIZE) {
    throw new SecurityException("文件过大");
}

// ✅ 检查解压后路径
String entryName = entry.getName();
if (entryName.contains("..")) {
    throw new SecurityException("路径异常");
}
```

## 规则1.8 XML注入防护

```java
// ✅ 使用DOM解析器
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
DocumentBuilder db = dbf.newDocumentBuilder();
Document doc = db.parse(inputSource);
```

## 规则1.9-1.11 XXE和正则DoS防护

- 禁用外部实体
- 限制正则复杂度
- 超时控制

## 输入安全检查清单

1. ✅ SQL使用参数化查询
2. ✅ 日志过滤特殊字符
3. ✅ 命令执行使用白名单
4. ✅ 文件路径标准化
5. ✅ XML禁用外部实体
6. ✅ 正则复杂度限制