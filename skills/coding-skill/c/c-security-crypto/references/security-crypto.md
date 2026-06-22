# C语言加密安全规范详细参考

## 1. 密码哈希

### 1.1 推荐算法及优先级

1. **Argon2**（首选，2015年密码哈希竞赛冠军）
2. **scrypt**（内存密集型）
3. **bcrypt**（广泛支持）
4. **PBKDF2**（兼容性好，但安全性较低）

### 1.2 bcrypt 示例

```c
#include <bcrypt.h>
#include <string.h>

#define BCRYPT_COST 12

int hash_password(const char *password, char *hash_out, size_t hash_size) {
    char salt[BCRYPT_HASHSIZE];
    
    if (bcrypt_gensalt(BCRYPT_COST, salt) != 0) {
        return -1;
    }
    
    if (bcrypt_hashpw(password, salt, hash_out) != 0) {
        return -1;
    }
    
    return 0;
}

int verify_password(const char *password, const char *hash) {
    char computed_hash[BCRYPT_HASHSIZE];
    
    if (bcrypt_hashpw(password, hash, computed_hash) != 0) {
        return -1;
    }
    
    return (strcmp(computed_hash, hash) == 0) ? 0 : -1;
}
```

### 1.3 Argon2 示例（使用libargon2）

```c
#include <argon2.h>

#define HASHLEN 32
#define SALTLEN 16
#define T_COST 3
#define M_COST 65536
#define PARALLELISM 4

int hash_password_argon2(const char *password, size_t pwd_len,
                         uint8_t *hash_out, uint8_t *salt) {
    uint32_t encodedlen = argon2_encodedlen(T_COST, M_COST, PARALLELISM,
                                             SALTLEN, HASHLEN, Argon2_id);
    char *encoded = malloc(encodedlen + 1);
    
    int result = argon2id_hash_encoded(T_COST, M_COST, PARALLELISM,
                                        password, pwd_len, salt, SALTLEN,
                                        HASHLEN, encoded, encodedlen);
    if (result == ARGON2_OK) {
        strcpy((char*)hash_out, encoded);
    }
    
    free(encoded);
    return (result == ARGON2_OK) ? 0 : -1;
}
```

---

## 2. 随机数生成

### 2.1 安全随机数要求

- **必须**使用密码学安全伪随机数生成器（CSPRNG）
- **禁止**使用 `rand()`、`random()` 等非密码学安全函数
- **禁止**使用 `srand(time(NULL))` 作为种子

### 2.2 使用 OpenSSL RAND_bytes

```c
#include <openssl/rand.h>
#include <stdio.h>

int generate_secure_random(uint8_t *buffer, size_t len) {
    if (RAND_bytes(buffer, len) != 1) {
        fprintf(stderr, "RAND_bytes failed\n");
        return -1;
    }
    return 0;
}

int generate_nonce(uint8_t *nonce, size_t len) {
    return generate_secure_random(nonce, len);
}

int generate_iv(uint8_t *iv, size_t len) {
    return generate_secure_random(iv, len);
}
```

### 2.3 使用 /dev/urandom（POSIX）

```c
#include <fcntl.h>
#include <unistd.h>

int generate_random_urandom(uint8_t *buffer, size_t len) {
    int fd = open("/dev/urandom", O_RDONLY);
    if (fd < 0) {
        return -1;
    }
    
    ssize_t bytes_read = read(fd, buffer, len);
    close(fd);
    
    return (bytes_read == (ssize_t)len) ? 0 : -1;
}
```

### 2.4 Windows 平台

```c
#include <windows.h>
#include <bcrypt.h>

int generate_random_windows(uint8_t *buffer, size_t len) {
    NTSTATUS status = BCryptGenRandom(
        NULL,
        buffer,
        len,
        BCRYPT_USE_SYSTEM_PREFERRED_RNG
    );
    
    return (status == 0) ? 0 : -1;
}
```

---

## 3. 对称加密（AES）

### 3.1 推荐配置

- **算法**：AES-256
- **模式**：GCM（首选）、CBC（需配合HMAC）
- **禁止**：ECB模式

### 3.2 AES-GCM 加密示例（OpenSSL EVP）

```c
#include <openssl/evp.h>
#include <openssl/rand.h>

#define KEY_SIZE 32
#define IV_SIZE 12
#define TAG_SIZE 16
#define GCM_TAG_SIZE 16

int aes_gcm_encrypt(const uint8_t *plaintext, int plaintext_len,
                    const uint8_t *key, const uint8_t *iv,
                    uint8_t *ciphertext, int *ciphertext_len,
                    uint8_t *tag) {
    EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();
    if (!ctx) return -1;
    
    int len;
    int ret = -1;
    
    if (EVP_EncryptInit_ex(ctx, EVP_aes_256_gcm(), NULL, NULL, NULL) != 1)
        goto cleanup;
    
    if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_SET_IVLEN, IV_SIZE, NULL) != 1)
        goto cleanup;
    
    if (EVP_EncryptInit_ex(ctx, NULL, NULL, key, iv) != 1)
        goto cleanup;
    
    if (EVP_EncryptUpdate(ctx, ciphertext, &len, plaintext, plaintext_len) != 1)
        goto cleanup;
    
    *ciphertext_len = len;
    
    if (EVP_EncryptFinal_ex(ctx, ciphertext + len, &len) != 1)
        goto cleanup;
    
    *ciphertext_len += len;
    
    if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_GET_TAG, GCM_TAG_SIZE, tag) != 1)
        goto cleanup;
    
    ret = 0;
    
cleanup:
    EVP_CIPHER_CTX_free(ctx);
    return ret;
}

int aes_gcm_decrypt(const uint8_t *ciphertext, int ciphertext_len,
                    const uint8_t *key, const uint8_t *iv,
                    const uint8_t *tag,
                    uint8_t *plaintext, int *plaintext_len) {
    EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();
    if (!ctx) return -1;
    
    int len;
    int ret = -1;
    
    if (EVP_DecryptInit_ex(ctx, EVP_aes_256_gcm(), NULL, NULL, NULL) != 1)
        goto cleanup;
    
    if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_SET_IVLEN, IV_SIZE, NULL) != 1)
        goto cleanup;
    
    if (EVP_DecryptInit_ex(ctx, NULL, NULL, key, iv) != 1)
        goto cleanup;
    
    if (EVP_DecryptUpdate(ctx, plaintext, &len, ciphertext, ciphertext_len) != 1)
        goto cleanup;
    
    *plaintext_len = len;
    
    if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_SET_TAG, GCM_TAG_SIZE, (void*)tag) != 1)
        goto cleanup;
    
    if (EVP_DecryptFinal_ex(ctx, plaintext + len, &len) != 1)
        goto cleanup;
    
    *plaintext_len += len;
    ret = 0;
    
cleanup:
    EVP_CIPHER_CTX_free(ctx);
    return ret;
}
```

---

## 4. 非对称加密

### 4.1 RSA

```c
#include <openssl/rsa.h>
#include <openssl/pem.h>
#include <openssl/err.h>

#define RSA_KEY_BITS 2048

RSA *generate_rsa_keypair(void) {
    RSA *rsa = RSA_new();
    BIGNUM *bn = BN_new();
    
    BN_set_word(bn, RSA_F4);
    RSA_generate_key_ex(rsa, RSA_KEY_BITS, bn, NULL);
    
    BN_free(bn);
    return rsa;
}

int rsa_encrypt_public(RSA *rsa, const uint8_t *plaintext, int len,
                       uint8_t *ciphertext) {
    return RSA_public_encrypt(len, plaintext, ciphertext, rsa,
                              RSA_PKCS1_OAEP_PADDING);
}

int rsa_decrypt_private(RSA *rsa, const uint8_t *ciphertext, int len,
                        uint8_t *plaintext) {
    return RSA_private_decrypt(len, ciphertext, plaintext, rsa,
                               RSA_PKCS1_OAEP_PADDING);
}
```

### 4.2 ECC（椭圆曲线）

```c
#include <openssl/ec.h>
#include <openssl/ecdsa.h>

EC_KEY *generate_ecc_keypair(void) {
    EC_KEY *key = EC_KEY_new_by_curve_name(NID_X9_62_prime256v1);
    if (!key) return NULL;
    
    if (EC_KEY_generate_key(key) != 1) {
        EC_KEY_free(key);
        return NULL;
    }
    
    return key;
}

int ecc_sign(EC_KEY *key, const uint8_t *msg, size_t msg_len,
             uint8_t *sig, unsigned int *sig_len) {
    return ECDSA_sign(0, msg, msg_len, sig, sig_len, key);
}

int ecc_verify(EC_KEY *key, const uint8_t *msg, size_t msg_len,
               const uint8_t *sig, unsigned int sig_len) {
    return ECDSA_verify(0, msg, msg_len, sig, sig_len, key);
}
```

---

## 5. SSL/TLS 安全

### 5.1 配置要求

- **启用**：TLS 1.2、TLS 1.3
- **禁用**：SSLv2、SSLv3、TLS 1.0、TLS 1.1
- **必须**验证服务器证书
- **禁止**禁用证书验证

### 5.2 安全 TLS 客户端配置

```c
#include <openssl/ssl.h>
#include <openssl/err.h>

SSL_CTX *create_tls_client_context(void) {
    SSL_CTX *ctx = SSL_CTX_new(TLS_client_method());
    if (!ctx) return NULL;
    
    SSL_CTX_set_min_proto_version(ctx, TLS1_2_VERSION);
    SSL_CTX_set_max_proto_version(ctx, TLS1_3_VERSION);
    
    SSL_CTX_set_default_verify_paths(ctx);
    SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, NULL);
    
    const char *cipher_list = "ECDHE+AESGCM:DHE+AESGCM";
    SSL_CTX_set_cipher_list(ctx, cipher_list);
    
    return ctx;
}

int tls_connect(SSL_CTX *ctx, const char *hostname, int port) {
    BIO *bio = BIO_new_ssl_connect(ctx);
    SSL *ssl = NULL;
    
    BIO_get_ssl(bio, &ssl);
    SSL_set_tlsext_host_name(ssl, hostname);
    
    char conn_str[256];
    snprintf(conn_str, sizeof(conn_str), "%s:%d", hostname, port);
    BIO_set_conn_hostname(bio, conn_str);
    
    if (BIO_do_connect(bio) != 1) {
        BIO_free_all(bio);
        return -1;
    }
    
    if (SSL_get_verify_result(ssl) != X509_V_OK) {
        BIO_free_all(bio);
        return -1;
    }
    
    return 0;
}
```

---

## 6. 密钥管理

### 6.1 安全实践

- **禁止**硬编码密钥在源代码中
- **禁止**将密钥提交到版本控制系统
- **必须**使用环境变量或密钥管理服务
- **必须**在内存中密钥使用后立即清零

### 6.2 从环境变量读取密钥

```c
#include <stdlib.h>
#include <string.h>

int get_key_from_env(const char *env_name, uint8_t *key, size_t key_len) {
    const char *env_val = getenv(env_name);
    if (!env_val) {
        return -1;
    }
    
    size_t env_len = strlen(env_val);
    if (env_len < key_len * 2) {
        return -1;
    }
    
    for (size_t i = 0; i < key_len; i++) {
        unsigned int byte;
        if (sscanf(env_val + i * 2, "%2x", &byte) != 1) {
            return -1;
        }
        key[i] = (uint8_t)byte;
    }
    
    return 0;
}
```

---

## 7. 敏感数据清理

### 7.1 必须清理的场景

- 密钥使用完毕后
- 密码哈希完成后
- 临时缓冲区中的敏感数据

### 7.2 安全清零方法

```c
#include <openssl/crypto.h>

void secure_zero(void *ptr, size_t len) {
    OPENSSL_cleanse(ptr, len);
}

void secure_zero_string(char *str) {
    if (str) {
        secure_zero(str, strlen(str));
    }
}

void secure_free_key(uint8_t *key, size_t key_len) {
    if (key) {
        OPENSSL_cleanse(key, key_len);
        free(key);
    }
}
```

---

## 8. 禁止硬编码密钥

### 8.1 错误示例

```c
// 禁止这样做！
const char *api_key = "sk-1234567890abcdef";
const uint8_t aes_key[32] = {
    0x00, 0x01, 0x02, 0x03,
    // ...
};
char password[] = "admin123";
```

### 8.2 正确做法

```c
// 正确：从环境变量读取
const char *api_key = getenv("API_KEY");
if (!api_key) {
    fprintf(stderr, "API_KEY not set\n");
    exit(1);
}

// 正确：从安全存储读取
int fd = open("/etc/app/secrets/key.bin", O_RDONLY);
read(fd, key, KEY_SIZE);
close(fd);
```

---

## 9. 安全检查清单

### 密码存储

- [ ] 使用 Argon2、bcrypt 或 PBKDF2
- [ ] 不使用 MD5、SHA1、SHA256 直接存储密码
- [ ] 每个密码使用唯一盐值
- [ ] 设置足够的工作因子

### 随机数

- [ ] 使用 RAND_bytes 或 /dev/urandom
- [ ] 不使用 rand() 或 srand()
- [ ] IV/Nonce 不重复使用

### 加密

- [ ] 使用 AES-256-GCM 或 AES-256-CBC+HMAC
- [ ] 不使用 ECB 模式
- [ ] 不使用 DES、3DES、RC4
- [ ] 正确处理加密错误

### TLS

- [ ] 启用 TLS 1.2 或更高版本
- [ ] 验证服务器证书
- [ ] 使用强密码套件
- [ ] 检查证书主机名

### 密钥管理

- [ ] 不硬编码密钥
- [ ] 密钥不提交到版本控制
- [ ] 使用环境变量或密钥管理服务
- [ ] 密钥使用后立即清零

### 敏感数据处理

- [ ] 使用 OPENSSL_cleanse 清零
- [ ] 不将敏感数据写入日志
- [ ] 限制敏感数据的生命周期
- [ ] 正确释放包含敏感数据的内存

---

## 10. 禁止使用的算法

| 算法类型 | 禁止使用 | 原因 | 替代方案 |
|---------|---------|------|---------|
| 哈希 | MD5 | 碰撞攻击 | SHA-256+ |
| 哈希 | SHA1 | 碰撞攻击 | SHA-256+ |
| 密码哈希 | MD5/SHA1/SHA256 | 太快 | Argon2/bcrypt |
| 对称加密 | DES | 56位密钥过短 | AES-256 |
| 对称加密 | 3DES | 速度慢 | AES-256 |
| 对称加密 | RC4 | 弱密钥流 | AES-256 |
| 对称加密 | ECB模式 | 相同明文→相同密文 | GCM/CBC |
| 协议 | SSLv2/v3 | POODLE等攻击 | TLS 1.2+ |
| 协议 | TLS 1.0/1.1 | BEAST等攻击 | TLS 1.2+ |
