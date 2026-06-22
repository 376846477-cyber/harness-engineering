---
name: java-security-crypto
description: Java编码规范 - 加密安全规范。适用于所有Java代码开发场景：(1) 密码加密和存储；(2) 加密算法选择；(3) 敏感信息硬编码检测；(4) SSL/TLS安全；(5) 随机数安全。触发关键词：Java、加密、encrypt、密码、password、加密算法、AES、RSA、SSL、TLS、SecureRandom、密钥、key
---

# Java加密安全规范

基于Java语言安全编程规范V3.2第9章"其他"

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则9.1 | 禁止在日志中保存口令、密钥 |
| 规则9.2 | 禁止使用私有或弱加密算法 |
| 规则9.3 | 口令存储必须加入盐值 |
| 规则9.4 | 禁止将敏感信息硬编码 |
| 规则9.5 | 安全场景必须使用强随机数 |
| 规则9.6 | 必须使用SSLSocket代替Socket |

## 加密算法要求

**❌ 禁止使用**:
- DES, 3DES
- MD5（用于密码存储）
- SHA1（用于密码存储）
- ECB模式

**✅ 推荐使用**:
- AES-256
- SHA-256+（用于密码存储）
- HMAC-SHA256
- GCM模式（对称加密）

## 密码存储安全

**✅ 正确示例**:
```java
// 使用BCrypt
PasswordEncoder encoder = new BCryptPasswordEncoder();
String hashed = encoder.encode(password);
boolean matches = encoder.matches(input, hashed);

// 或使用PBKDF2
SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
KeySpec spec = new PBEKeySpec(password.toCharArray(), salt, 65536, 256);
SecretKey key = new SecretKeySpec(factory.generateSecret(spec).getEncoded(), "AES");
```

## 敏感信息硬编码

**❌ 错误示例**:
```java
private static final String API_KEY = "sk-xxxxxx";  // 硬编码
private static final String DB_PASSWORD = "admin123";  // 硬编码
```

**✅ 正确示例**:
```java
// 使用配置中心或密钥管理系统
private static final String API_KEY = System.getenv("API_KEY");
private static final String DB_PASSWORD = secureConfig.getPassword("db");
```

## 随机数安全

**✅ 正确示例**:
```java
// 安全随机数
SecureRandom random = new SecureRandom();
byte[] bytes = new byte[32];
random.nextBytes(bytes);
```

## SSL/TLS安全

**✅ 正确示例**:
```java
// 使用SSLSocket
SSLSocketFactory factory = (SSLSocketFactory) SSLSocketFactory.getDefault();
SSLSocket socket = (SSLSocket) factory.createSocket(host, port);

// 配置强TLS版本
socket.setEnabledProtocols(new String[]{"TLSv1.2", "TLSv1.3"});
```

## 详细加密安全规范

见 [references/security-crypto.md](references/security-crypto.md)