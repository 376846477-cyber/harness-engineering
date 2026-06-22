# C++加密安全规范详细参考

## 1. 密码哈希

### 1.1 推荐算法优先级

| 优先级 | 算法 | 适用场景 | 参数要求 |
|--------|------|----------|----------|
| 1 | Argon2id | 新项目首选 | time_cost≥2, memory_cost≥64MB, parallelism≥4 |
| 2 | bcrypt | 兼容旧系统 | cost factor≥12 |
| 3 | PBKDF2 | 兼容性要求 | iterations≥100000, HMAC-SHA256 |

### 1.2 Argon2示例（使用libargon2）

```cpp
#include <argon2.h>
#include <vector>
#include <string>
#include <random>

std::string hashPassword(const std::string& password) {
    constexpr uint32_t t_cost = 3;
    constexpr uint32_t m_cost = 65536;
    constexpr uint32_t parallelism = 4;
    constexpr size_t salt_len = 16;
    constexpr size_t hash_len = 32;
    
    std::vector<uint8_t> salt(salt_len);
    std::random_device rd;
    std::generate(salt.begin(), salt.end(), [&]() { return rd(); });
    
    std::vector<uint8_t> hash(hash_len);
    
    int result = argon2id_hash_raw(
        t_cost, m_cost, parallelism,
        password.data(), password.size(),
        salt.data(), salt_len,
        hash.data(), hash_len
    );
    
    if (result != ARGON2_OK) {
        throw std::runtime_error(argon2_error_message(result));
    }
    
    std::string encoded;
    encoded.resize(hash_len * 2 + salt_len * 2 + 64);
    argon2id_hash_encoded(t_cost, m_cost, parallelism,
        password.data(), password.size(),
        salt.data(), salt_len, hash_len,
        encoded.data(), encoded.size());
    
    return encoded;
}
```

### 1.3 bcrypt示例（使用libbcrypt）

```cpp
#include <bcrypt.h>

std::string hashPasswordBcrypt(const std::string& password) {
    char salt[BCRYPT_HASHSIZE];
    char hash[BCRYPT_HASHSIZE];
    
    bcrypt_gensalt(12, salt);
    bcrypt_hashpw(password.c_str(), salt, hash);
    
    return std::string(hash);
}

bool verifyPasswordBcrypt(const std::string& password, const std::string& hash) {
    char newHash[BCRYPT_HASHSIZE];
    bcrypt_hashpw(password.c_str(), hash.c_str(), newHash);
    
    return strcmp(hash.c_str(), newHash) == 0;
}
```

### 1.4 禁止使用

```cpp
// 禁止：MD5用于密码
MD5(password);

// 禁止：SHA1/SHA256直接用于密码
SHA256(password);

// 禁止：无盐哈希
auto hash = sha256(password + "static_salt");
```

## 2. 随机数生成

### 2.1 安全随机数源

```cpp
#include <openssl/rand.h>
#include <random>
#include <array>

std::vector<uint8_t> generateSecureRandom(size_t length) {
    std::vector<uint8_t> buffer(length);
    
    if (RAND_bytes(buffer.data(), static_cast<int>(length)) != 1) {
        throw std::runtime_error("Failed to generate secure random bytes");
    }
    
    return buffer;
}

// C++17方式
std::array<uint8_t, 32> generateRandomArray() {
    std::array<uint8_t, 32> arr{};
    std::random_device rd;
    
    for (auto& byte : arr) {
        byte = static_cast<uint8_t>(rd());
    }
    
    return arr;
}
```

### 2.2 禁止使用的随机数

```cpp
// 禁止：不安全的随机数生成器
srand(time(nullptr));
int r = rand();

// 禁止：不安全的C++随机数
std::rand();

// 禁止：伪随机数生成器用于安全目的
std::mt19937 rng(seed);
```

### 2.3 加密安全的随机数用途

- 生成加密密钥
- 生成IV/Nonce
- 生成盐值
- 生成会话ID
- 生成CSRF Token

## 3. 对称加密

### 3.1 AES-GCM推荐示例

```cpp
#include <openssl/evp.h>
#include <vector>
#include <array>

class AesGcmEncryptor {
public:
    static constexpr size_t KEY_SIZE = 32;
    static constexpr size_t IV_SIZE = 12;
    static constexpr size_t TAG_SIZE = 16;
    
    struct EncryptedData {
        std::vector<uint8_t> iv;
        std::vector<uint8_t> ciphertext;
        std::vector<uint8_t> tag;
    };
    
    static EncryptedData encrypt(const std::vector<uint8_t>& plaintext,
                                  const std::vector<uint8_t>& key,
                                  const std::vector<uint8_t>& aad = {}) {
        if (key.size() != KEY_SIZE) {
            throw std::invalid_argument("Key must be 32 bytes");
        }
        
        EncryptedData result;
        result.iv = generateSecureRandom(IV_SIZE);
        result.ciphertext.resize(plaintext.size());
        result.tag.resize(TAG_SIZE);
        
        EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
        if (!ctx) throw std::runtime_error("Failed to create context");
        
        try {
            if (EVP_EncryptInit_ex(ctx, EVP_aes_256_gcm(), nullptr, nullptr, nullptr) != 1) {
                throw std::runtime_error("Failed to init encrypt");
            }
            
            if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_SET_IVLEN, IV_SIZE, nullptr) != 1) {
                throw std::runtime_error("Failed to set IV length");
            }
            
            if (EVP_EncryptInit_ex(ctx, nullptr, nullptr, key.data(), result.iv.data()) != 1) {
                throw std::runtime_error("Failed to set key/IV");
            }
            
            if (!aad.empty()) {
                int len;
                if (EVP_EncryptUpdate(ctx, nullptr, &len, aad.data(), static_cast<int>(aad.size())) != 1) {
                    throw std::runtime_error("Failed to process AAD");
                }
            }
            
            int len;
            if (EVP_EncryptUpdate(ctx, result.ciphertext.data(), &len,
                                  plaintext.data(), static_cast<int>(plaintext.size())) != 1) {
                throw std::runtime_error("Failed to encrypt");
            }
            
            if (EVP_EncryptFinal_ex(ctx, result.ciphertext.data() + len, &len) != 1) {
                throw std::runtime_error("Failed to finalize encrypt");
            }
            
            if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_GET_TAG, TAG_SIZE, result.tag.data()) != 1) {
                throw std::runtime_error("Failed to get tag");
            }
            
            EVP_CIPHER_CTX_free(ctx);
        } catch (...) {
            EVP_CIPHER_CTX_free(ctx);
            throw;
        }
        
        return result;
    }
    
    static std::vector<uint8_t> decrypt(const EncryptedData& data,
                                         const std::vector<uint8_t>& key,
                                         const std::vector<uint8_t>& aad = {}) {
        if (key.size() != KEY_SIZE) {
            throw std::invalid_argument("Key must be 32 bytes");
        }
        
        std::vector<uint8_t> plaintext(data.ciphertext.size());
        
        EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
        if (!ctx) throw std::runtime_error("Failed to create context");
        
        try {
            if (EVP_DecryptInit_ex(ctx, EVP_aes_256_gcm(), nullptr, nullptr, nullptr) != 1) {
                throw std::runtime_error("Failed to init decrypt");
            }
            
            if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_SET_IVLEN, IV_SIZE, nullptr) != 1) {
                throw std::runtime_error("Failed to set IV length");
            }
            
            if (EVP_DecryptInit_ex(ctx, nullptr, nullptr, key.data(), data.iv.data()) != 1) {
                throw std::runtime_error("Failed to set key/IV");
            }
            
            if (!aad.empty()) {
                int len;
                if (EVP_DecryptUpdate(ctx, nullptr, &len, aad.data(), static_cast<int>(aad.size())) != 1) {
                    throw std::runtime_error("Failed to process AAD");
                }
            }
            
            int len;
            if (EVP_DecryptUpdate(ctx, plaintext.data(), &len,
                                  data.ciphertext.data(), static_cast<int>(data.ciphertext.size())) != 1) {
                throw std::runtime_error("Failed to decrypt");
            }
            
            if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_SET_TAG, TAG_SIZE,
                                    const_cast<uint8_t*>(data.tag.data())) != 1) {
                throw std::runtime_error("Failed to set tag");
            }
            
            if (EVP_DecryptFinal_ex(ctx, plaintext.data() + len, &len) != 1) {
                throw std::runtime_error("Authentication failed - tag mismatch");
            }
            
            EVP_CIPHER_CTX_free(ctx);
        } catch (...) {
            EVP_CIPHER_CTX_free(ctx);
            throw;
        }
        
        return plaintext;
    }
    
private:
    static std::vector<uint8_t> generateSecureRandom(size_t length) {
        std::vector<uint8_t> buffer(length);
        if (RAND_bytes(buffer.data(), static_cast<int>(length)) != 1) {
            throw std::runtime_error("Failed to generate random bytes");
        }
        return buffer;
    }
};
```

### 3.2 禁止的加密模式

```cpp
// 禁止：ECB模式（相同明文产生相同密文）
EVP_aes_256_ecb();

// 禁止：CBC模式未认证（存在填充攻击）
EVP_aes_256_cbc();

// 禁止：无完整性校验
EVP_aes_256_ctr();
```

## 4. 非对称加密

### 4.1 RSA推荐用法

```cpp
#include <openssl/rsa.h>
#include <openssl/pem.h>
#include <openssl/err.h>

class RsaCrypto {
public:
    static constexpr int KEY_BITS = 4096;
    static constexpr int PADDING = RSA_PKCS1_OAEP_PADDING;
    
    static std::pair<std::string, std::string> generateKeyPair() {
        RSA* rsa = RSA_new();
        BIGNUM* bn = BN_new();
        BN_set_word(bn, RSA_F4);
        
        if (RSA_generate_key_ex(rsa, KEY_BITS, bn, nullptr) != 1) {
            RSA_free(rsa);
            BN_free(bn);
            throw std::runtime_error("Failed to generate RSA key");
        }
        
        BIO* pubBio = BIO_new(BIO_s_mem());
        BIO* privBio = BIO_new(BIO_s_mem());
        
        PEM_write_bio_RSAPublicKey(pubBio, rsa);
        PEM_write_bio_RSAPrivateKey(privBio, rsa, nullptr, nullptr, 0, nullptr, nullptr);
        
        std::string publicKey = bioToString(pubBio);
        std::string privateKey = bioToString(privBio);
        
        BIO_free(pubBio);
        BIO_free(privBio);
        RSA_free(rsa);
        BN_free(bn);
        
        return {publicKey, privateKey};
    }
    
    static std::vector<uint8_t> encrypt(const std::vector<uint8_t>& data,
                                         const std::string& publicKeyPem) {
        BIO* bio = BIO_new_mem_buf(publicKeyPem.data(), -1);
        RSA* rsa = PEM_read_bio_RSAPublicKey(bio, nullptr, nullptr, nullptr);
        BIO_free(bio);
        
        if (!rsa) throw std::runtime_error("Failed to load public key");
        
        std::vector<uint8_t> encrypted(RSA_size(rsa));
        int result = RSA_public_encrypt(
            static_cast<int>(data.size()),
            data.data(),
            encrypted.data(),
            rsa,
            PADDING
        );
        
        RSA_free(rsa);
        
        if (result == -1) {
            throw std::runtime_error("RSA encryption failed");
        }
        
        encrypted.resize(result);
        return encrypted;
    }
    
private:
    static std::string bioToString(BIO* bio) {
        char* data = nullptr;
        long len = BIO_get_mem_data(bio, &data);
        return std::string(data, len);
    }
};
```

### 4.2 ECC推荐（Ed25519签名）

```cpp
#include <openssl/evp.h>
#include <openssl/err.h>

class Ed25519Signer {
public:
    struct KeyPair {
        std::vector<uint8_t> privateKey;
        std::vector<uint8_t> publicKey;
    };
    
    static KeyPair generateKeyPair() {
        EVP_PKEY* pkey = nullptr;
        EVP_PKEY_CTX* ctx = EVP_PKEY_CTX_new_id(EVP_PKEY_ED25519, nullptr);
        
        if (!ctx) throw std::runtime_error("Failed to create context");
        
        if (EVP_PKEY_keygen_init(ctx) <= 0) {
            EVP_PKEY_CTX_free(ctx);
            throw std::runtime_error("Failed to init keygen");
        }
        
        if (EVP_PKEY_keygen(ctx, &pkey) <= 0) {
            EVP_PKEY_CTX_free(ctx);
            throw std::runtime_error("Failed to generate key");
        }
        
        KeyPair kp;
        kp.privateKey.resize(32);
        kp.publicKey.resize(32);
        
        size_t privLen = 32, pubLen = 32;
        EVP_PKEY_get_raw_private_key(pkey, kp.privateKey.data(), &privLen);
        EVP_PKEY_get_raw_public_key(pkey, kp.publicKey.data(), &pubLen);
        
        EVP_PKEY_free(pkey);
        EVP_PKEY_CTX_free(ctx);
        
        return kp;
    }
    
    static std::vector<uint8_t> sign(const std::vector<uint8_t>& message,
                                      const std::vector<uint8_t>& privateKey) {
        EVP_PKEY* pkey = EVP_PKEY_new_raw_private_key(
            EVP_PKEY_ED25519, nullptr,
            privateKey.data(), privateKey.size()
        );
        
        if (!pkey) throw std::runtime_error("Failed to load private key");
        
        EVP_MD_CTX* mdCtx = EVP_MD_CTX_new();
        std::vector<uint8_t> signature(64);
        size_t sigLen = 64;
        
        if (EVP_DigestSignInit(mdCtx, nullptr, nullptr, nullptr, pkey) <= 0) {
            EVP_MD_CTX_free(mdCtx);
            EVP_PKEY_free(pkey);
            throw std::runtime_error("Failed to init sign");
        }
        
        if (EVP_DigestSign(mdCtx, signature.data(), &sigLen,
                           message.data(), message.size()) <= 0) {
            EVP_MD_CTX_free(mdCtx);
            EVP_PKEY_free(pkey);
            throw std::runtime_error("Failed to sign");
        }
        
        EVP_MD_CTX_free(mdCtx);
        EVP_PKEY_free(pkey);
        
        return signature;
    }
    
    static bool verify(const std::vector<uint8_t>& message,
                       const std::vector<uint8_t>& signature,
                       const std::vector<uint8_t>& publicKey) {
        EVP_PKEY* pkey = EVP_PKEY_new_raw_public_key(
            EVP_PKEY_ED25519, nullptr,
            publicKey.data(), publicKey.size()
        );
        
        if (!pkey) return false;
        
        EVP_MD_CTX* mdCtx = EVP_MD_CTX_new();
        
        if (EVP_DigestVerifyInit(mdCtx, nullptr, nullptr, nullptr, pkey) <= 0) {
            EVP_MD_CTX_free(mdCtx);
            EVP_PKEY_free(pkey);
            return false;
        }
        
        int result = EVP_DigestVerify(mdCtx, signature.data(), signature.size(),
                                       message.data(), message.size());
        
        EVP_MD_CTX_free(mdCtx);
        EVP_PKEY_free(pkey);
        
        return result == 1;
    }
};
```

## 5. SSL/TLS配置

### 5.1 OpenSSL安全配置

```cpp
#include <openssl/ssl.h>

SSL_CTX* createSecureSslCtx() {
    SSL_CTX* ctx = SSL_CTX_new(TLS_server_method());
    if (!ctx) throw std::runtime_error("Failed to create SSL context");
    
    SSL_CTX_set_min_proto_version(ctx, TLS1_2_VERSION);
    
    SSL_CTX_set_options(ctx, 
        SSL_OP_NO_SSLv2 | SSL_OP_NO_SSLv3 |
        SSL_OP_NO_TLSv1 | SSL_OP_NO_TLSv1_1 |
        SSL_OP_NO_COMPRESSION |
        SSL_OP_CIPHER_SERVER_PREFERENCE
    );
    
    const char* cipherList = 
        "ECDHE-ECDSA-AES256-GCM-SHA384:"
        "ECDHE-RSA-AES256-GCM-SHA384:"
        "ECDHE-ECDSA-CHACHA20-POLY1305:"
        "ECDHE-RSA-CHACHA20-POLY1305:"
        "ECDHE-ECDSA-AES128-GCM-SHA256:"
        "ECDHE-RSA-AES128-GCM-SHA256";
    
    if (SSL_CTX_set_cipher_list(ctx, cipherList) != 1) {
        SSL_CTX_free(ctx);
        throw std::runtime_error("Failed to set cipher list");
    }
    
    SSL_CTX_set_verify(ctx, 
        SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT,
        nullptr
    );
    
    return ctx;
}
```

### 5.2 TLS版本要求

| 版本 | 状态 |
|------|------|
| SSL 2.0 | 禁止 |
| SSL 3.0 | 禁止 |
| TLS 1.0 | 禁止 |
| TLS 1.1 | 禁止 |
| TLS 1.2 | 推荐 |
| TLS 1.3 | 首选 |

## 6. 密钥管理

### 6.1 安全的密钥存储

```cpp
#include <cstdlib>
#include <string>
#include <optional>

class SecureKeyStorage {
public:
    static std::optional<std::string> getFromEnv(const std::string& keyName) {
        const char* value = std::getenv(keyName.c_str());
        if (value == nullptr) {
            return std::nullopt;
        }
        return std::string(value);
    }
    
    static void secureClear(std::string& str) {
        if (!str.empty()) {
            volatile char* p = const_cast<volatile char*>(str.data());
            for (size_t i = 0; i < str.size(); ++i) {
                p[i] = '\0';
            }
            str.clear();
        }
    }
    
    static void secureClear(std::vector<uint8_t>& data) {
        if (!data.empty()) {
            volatile uint8_t* p = data.data();
            for (size_t i = 0; i < data.size(); ++i) {
                p[i] = 0;
            }
            data.clear();
        }
    }
};
```

### 6.2 禁止硬编码密钥

```cpp
// 禁止：硬编码密钥
const char* API_KEY = "sk-1234567890abcdef";
const char DB_PASSWORD[] = "admin123";
std::string secretKey = "mySecretKey123!";

// 正确：从安全源获取
auto apiKey = SecureKeyStorage::getFromEnv("API_KEY");
if (!apiKey) {
    throw std::runtime_error("API_KEY not configured");
}
```

## 7. 敏感数据清理

### 7.1 安全清理函数

```cpp
#include <cstring>
#include <vector>
#include <string>
#include <algorithm>

namespace secure {

template<typename T>
void zeroMemory(T& container) {
    if (container.empty()) return;
    
    volatile typename T::value_type* p = container.data();
    for (size_t i = 0; i < container.size(); ++i) {
        p[i] = typename T::value_type{};
    }
}

template<typename T, size_t N>
void zeroMemory(T (&array)[N]) {
    volatile T* p = array;
    for (size_t i = 0; i < N; ++i) {
        p[i] = T{};
    }
}

void secureStringClear(std::string& str) {
    if (str.empty()) return;
    
    volatile char* p = &str[0];
    for (size_t i = 0; i < str.size(); ++i) {
        p[i] = '\0';
    }
    str.clear();
    str.shrink_to_fit();
}

}

class SecureBuffer {
public:
    explicit SecureBuffer(size_t size) : data_(size) {}
    ~SecureBuffer() { secure::zeroMemory(data_); }
    
    SecureBuffer(const SecureBuffer&) = delete;
    SecureBuffer& operator=(const SecureBuffer&) = delete;
    
    uint8_t* data() { return data_.data(); }
    const uint8_t* data() const { return data_.data(); }
    size_t size() const { return data_.size(); }
    
private:
    std::vector<uint8_t> data_;
};
```

### 7.2 RAII敏感数据管理

```cpp
template<typename T>
class ScopedSensitive {
public:
    explicit ScopedSensitive(T value) : value_(std::move(value)) {}
    ~ScopedSensitive() { secure::zeroMemory(value_); }
    
    ScopedSensitive(const ScopedSensitive&) = delete;
    ScopedSensitive& operator=(const ScopedSensitive&) = delete;
    
    const T& get() const { return value_; }
    T& get() { return value_; }
    
private:
    T value_;
};
```

## 8. 安全检查清单

### 8.1 密码存储检查

- [ ] 使用Argon2id/bcrypt/PBKDF2哈希密码
- [ ] 每个密码使用唯一盐值
- [ ] 盐值长度≥16字节
- [ ] 不使用MD5/SHA1/SHA256直接存储密码
- [ ] 验证失败使用恒定时间比较

### 8.2 加密检查

- [ ] 使用AES-256-GCM或ChaCha20-Poly1305
- [ ] 禁止ECB模式
- [ ] 每次加密使用唯一IV/Nonce
- [ ] IV长度：GCM=12字节，CTR=16字节
- [ ] 包含完整性校验（MAC/Tag）

### 8.3 密钥管理检查

- [ ] 密钥从不硬编码
- [ ] 密钥从安全源获取（环境变量/KMS）
- [ ] 密钥使用后立即清理
- [ ] 密钥传输使用TLS保护
- [ ] 定期轮换密钥

### 8.4 TLS配置检查

- [ ] 最低TLS 1.2
- [ ] 禁用SSLv2/SSLv3/TLS1.0/TLS1.1
- [ ] 使用强密码套件
- [ ] 启用证书验证
- [ ] 禁用压缩

### 8.5 随机数检查

- [ ] 使用RAND_bytes或std::random_device
- [ ] 禁止使用rand()/std::rand()
- [ ] 加密用途使用CSPRNG
- [ ] 不使用时间戳作为随机种子

### 8.6 代码审查要点

| 检查项 | 合格 | 不合格 |
|--------|------|--------|
| 密钥硬编码 | 无 | 存在硬编码密钥 |
| 密码哈希算法 | Argon2/bcrypt/PBKDF2 | MD5/SHA1/明文 |
| 加密模式 | GCM/CCM/ChaCha20-Poly1305 | ECB/CBC无MAC |
| TLS版本 | ≥1.2 | ≤1.1 |
| 随机数 | RAND_bytes | rand() |

## 9. 常见漏洞示例

### 9.1 硬编码凭证检测

```cpp
// 危险：硬编码凭证
const char* DB_CREDENTIALS = "admin:P@ssw0rd";
static const char API_SECRET[] = "sk_live_xxxxx";

// 检测正则
// (password|secret|key|token|credential)\s*=\s*["'][^"']+["']
```

### 9.2 弱加密检测

```cpp
// 危险：弱加密
EVP_aes_128_ecb();        // ECB模式
MD5(data, len, hash);     // MD5哈希
DES_ncbc_encrypt();       // DES加密

// 检测关键词
// ECB, MD5, SHA1, DES, RC4, rand()
```

## 10. 依赖库版本要求

| 库 | 最低版本 | 推荐版本 |
|----|----------|----------|
| OpenSSL | 1.1.1 | 3.0+ |
| libargon2 | 20190702 | 最新 |
| libsodium | 1.0.18 | 最新 |
| Botan | 2.19 | 3.x |

## 11. 参考资料

- OWASP Cryptographic Storage Cheat Sheet
- NIST SP 800-175B: Guideline for Using Cryptographic Standards
- OpenSSL Documentation: https://www.openssl.org/docs/
- libsodium Documentation: https://doc.libsodium.org/
