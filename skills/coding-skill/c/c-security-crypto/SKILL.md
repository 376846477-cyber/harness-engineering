---
name: c-security-crypto
description: C语言编码规范 - 加密安全规范。适用于所有C语言代码开发场景：(1) 密码哈希和存储；(2) 加密算法选择；(3) 敏感信息硬编码检测；(4) SSL/TLS安全。触发关键词：C语言、加密、encrypt、密码、password、crypto、SSL、TLS、密钥、key、OpenSSL
---

# C语言加密安全规范

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

```c
#include <openssl/evp.h>
#include <openssl/rand.h>

int hash_password(const char *password, unsigned char *output, size_t output_len)
{
    unsigned char salt[32];
    if (RAND_bytes(salt, sizeof(salt)) != 1) {
        return ERROR_CRYPTO;
    }

    if (PKCS5_PBKDF2_HMAC(password, strlen(password),
                           salt, sizeof(salt),
                           100000, EVP_sha256(),
                           output_len, output) != 1) {
        return ERROR_CRYPTO;
    }

    return SUCCESS;
}
```

## 敏感信息

```c
/* 错误：硬编码 */
const char *api_key = "sk-xxxxxx";

/* 正确：环境变量 */
const char *api_key = getenv("API_KEY");
if (api_key == NULL) {
    return ERROR_CONFIG;
}
```

## 详细规范

见 [references/security-crypto.md](references/security-crypto.md)
