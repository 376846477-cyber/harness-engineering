---
name: java-security-input
description: Java编码规范 - 输入校验与安全规范。适用于所有Java代码开发场景：(1) SQL注入、命令注入防护；(2) 文件路径安全；(3) XML解析安全；(4) 日志注入防护；(5) 任何外部输入处理。触发关键词：Java、注入、SQL、injection、exec、命令、安全、校验、validation、输入、input
---

# Java输入校验与安全规范

基于Java语言安全编程规范V3.2第1章"数据校验"

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则1.2 | 禁止直接使用不可信数据拼接SQL |
| 规则1.3 | 禁止直接使用不可信数据记录日志 |
| 规则1.5 | 禁止向Runtime.exec()传递不可信数据 |
| 规则1.6 | 文件路径校验前必须先标准化处理 |
| 规则1.8 | 禁止直接使用不可信数据拼接XML |

## SQL注入防护

**❌ 错误示例**:
```java
String sql = "SELECT * FROM users WHERE name = '" + username + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);
```

**✅ 正确示例**:
```java
// 使用参数化查询
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
stmt.setString(1, username);

// 使用存储过程
CallableStatement cstmt = conn.prepareCall("{call getUser(?)}");
cstmt.setString(1, username);
```

## 命令注入防护

**❌ 错误示例**:
```java
Runtime.getRuntime().exec("ls " + userInput);
```

**✅ 正确示例**:
```java
// 使用白名单验证
if (!userInput.matches("^[a-zA-Z0-9_]+$")) {
    throw new IllegalArgumentException("无效的输入");
}
ProcessBuilder pb = new ProcessBuilder("ls", userInput);
```

## 文件操作安全

```java
// 路径标准化
File file = new File(userInput);
String canonicalPath = file.getCanonicalPath();
if (!canonicalPath.startsWith(BASE_DIR)) {
    throw new SecurityException("路径越界");
}
```

## XML注入防护

```java
// 使用DOM解析器，禁止直接拼接
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
DocumentBuilder db = dbf.newDocumentBuilder();
Document doc = db.parse(new InputSource(new StringReader(xmlContent)));
```

## 详细安全规范

见 [references/security-input.md](references/security-input.md)