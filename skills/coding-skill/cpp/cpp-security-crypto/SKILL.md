---
name: cpp-security-crypto
description: C++编码规范 - 加密安全规范。适用于所有C++代码开发场景：(1) 密码哈希和存储；(2) 加密算法选择；(3) 敏感信息硬编码检测；(4) SSL/TLS安全。触发关键词：C++、加密、encrypt、密码、password、crypto、SSL、TLS、密钥、key、OpenSSL
---

# C++加密安全规范

基于安全编程规范

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 禁止在日志中保存口令、密钥 |
| 规则1.2 | 禁止使用弱加密算法（MD5/SHA1存密码） |
| 规则1.3 | 密码存储必须加盐 |
| 规则1.4 | 禁止将敏感信息硬编码 |
| 规则1.5 | 使用安全随机数生成器 |

## 密码存储

```cpp
#include <openssl/evp.h>
#include <openssl/rand.h>

std::vector<uint8_t> hash_password(const std::string& password) {
    std::array<uint8_t, 32> salt;
    RAND_bytes(salt.data(), salt.size());

    std::vector<uint8_t> key(32);
    PKCS5_PBKDF2_HMAC(
        password.c_str(), password.size(),
        salt.data(), salt.size(),
        100000, EVP_sha256(),
        key.size(), key.data()
    );

    std::vector<uint8_t> result;
    result.insert(result.end(), salt.begin(), salt.end());
    result.insert(result.end(), key.begin(), key.end());
    return result;
}
```

## 敏感信息

```cpp
// 错误：硬编码
const char* API_KEY = "sk-xxxxxx";

// 正确：环境变量
const char* api_key = std::getenv("API_KEY");
if (!api_key) {
    throw std::runtime_error("API_KEY环境变量未设置");
}
```

## 详细规范

见 [references/security-crypto.md](references/security-crypto.md)
