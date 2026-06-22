# 华为Java加密安全规范详细参考

## 规则9.1 日志敏感数据

- 禁止在日志中保存口令、密钥、Token
- 禁止记录完整身份证、银行卡号

```java
// ❌ 错误
log.info("用户密码: {}", user.getPassword());
log.info("API Key: {}", apiKey);

// ✅ 正确
log.info("用户登录: userId={}", userId);
log.debug("API请求: method={}", method);
```

## 规则9.2 加密算法要求

### 禁止使用
- DES, 3DES
- MD5（密码存储）
- SHA1（密码存储）
- RC4
- ECB模式

### 推荐使用
- 对称加密：AES-256 (GCM模式)
- 非对称：RSA-2048+, ECC
- 哈希：SHA-256, SHA-384, SHA-512
- 消息认证：HMAC-SHA256
- 密钥派生：PBKDF2, bcrypt, scrypt

```java
// ✅ AES-GCM加密
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
cipher.init(Cipher.ENCRYPT_MODE, secretKey);
byte[] ciphertext = cipher.doFinal(plaintext);
```

## 规则9.3 密码存储

- 必须使用盐值
- 必须使用慢哈希算法

```java
// ✅ BCrypt
PasswordEncoder encoder = new BCryptPasswordEncoder();
String hash = encoder.encode(password);
boolean match = encoder.matches(input, hash);

// ✅ PBKDF2
SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
KeySpec spec = new PBEKeySpec(password.toCharArray(), salt, 65536, 256);
byte[] hash = factory.generateSecret(spec).getEncoded();
```

## 规则9.4 敏感信息硬编码

**❌ 禁止**:
```java
private static final String API_KEY = "sk-xxxxxx";
private static final String DB_PASSWORD = "admin123";
private static final String JWT_SECRET = "secret123";
```

**✅ 必须**:
```java
// 使用环境变量
private static final String API_KEY = System.getenv("API_KEY");

// 使用配置中心
private static final String API_KEY = config.getSecret("api.key");

// 使用密钥管理服务
String apiKey = kms.getSecret("api-key");
```

## 规则9.5 随机数安全

- 安全场景必须使用SecureRandom

```java
// ❌ 不安全
Random random = new Random();
int code = random.nextInt(900000) + 100000;  // 伪随机

// ✅ 安全
SecureRandom random = new SecureRandom();
byte[] bytes = new byte[32];
random.nextBytes(bytes);

// 随机数生成
int code = random.nextInt(900000) + 100000;
```

## 规则9.6 SSL/TLS

- 必须使用SSLSocket代替Socket
- 禁用SSLv3, TLS1.0, TLS1.1

```java
// ❌ 不安全
Socket socket = new Socket(host, port);

// ✅ 安全
SSLSocketFactory factory = (SSLSocketFactory) SSLSocketFactory.getDefault();
SSLSocket socket = (SSLSocket) factory.createSocket(host, port);
socket.setEnabledProtocols(new String[]{"TLSv1.2", "TLSv1.3"});
```

## 建议9.1 敏感数据用char[]

```java
// ✅ 使用char[]
char[] password = readPassword();
try {
    // 使用密码
} finally {
    // 立即清零
    Arrays.fill(password, '0');
}
```

## 加密安全检查清单

1. ✅ 日志不含敏感信息
2. ✅ 使用强加密算法
3. ✅ 密码存储加盐
4. ✅ 敏感信息不硬编码
5. ✅ 安全场景用SecureRandom
6. ✅ 使用SSLSocket
7. ✅ 敏感数据用char[]