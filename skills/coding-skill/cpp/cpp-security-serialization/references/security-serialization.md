# C++序列化安全规范详细参考

## 一、二进制序列化安全

### 1.1 基本原则

二进制序列化直接操作内存布局，存在以下安全风险：
- 不同平台字节序差异
- 内存对齐问题
- 指针序列化导致悬空指针
- 缓冲区溢出风险

### 1.2 安全实践

```cpp
// 错误示例：直接内存拷贝（危险）
struct UserData {
    char name[32];
    int age;
};
void serialize_bad(const UserData& data, std::vector<char>& buffer) {
    buffer.resize(sizeof(UserData));
    memcpy(buffer.data(), &data, sizeof(UserData));  // 不安全：未考虑对齐和字节序
}

// 正确示例：逐字段序列化
void serialize_safe(const UserData& data, std::vector<char>& buffer) {
    // 写入长度前缀
    uint32_t name_len = static_cast<uint32_t>(strnlen(data.name, 31));
    buffer.insert(buffer.end(), 
                  reinterpret_cast<const char*>(&name_len),
                  reinterpret_cast<const char*>(&name_len) + sizeof(name_len));
    
    // 写入字符串内容
    buffer.insert(buffer.end(), data.name, data.name + name_len);
    
    // 写入整数（网络字节序）
    uint32_t age_net = htonl(static_cast<uint32_t>(data.age));
    buffer.insert(buffer.end(),
                  reinterpret_cast<const char*>(&age_net),
                  reinterpret_cast<const char*>(&age_net) + sizeof(age_net));
}
```

### 1.3 字节序处理

```cpp
// 使用标准化的字节序转换
#include <arpa/inet.h>  // 或 <winsock2.h> on Windows

// 定义可移植的序列化函数
template<typename T>
typename std::enable_if<std::is_integral<T>::value>::type
write_be(std::vector<char>& buf, T value) {
    for (size_t i = 0; i < sizeof(T); ++i) {
        buf.push_back(static_cast<char>((value >> (8 * (sizeof(T) - 1 - i))) & 0xFF));
    }
}

template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
read_be(const char* buf) {
    T value = 0;
    for (size_t i = 0; i < sizeof(T); ++i) {
        value = (value << 8) | static_cast<unsigned char>(buf[i]);
    }
    return value;
}
```

---

## 二、JSON序列化安全

### 2.1 nlohmann/json 安全实践

```cpp
#include <nlohmann/json.hpp>
using json = nlohmann::json;

// 安全反序列化函数
bool parse_json_safe(const std::string& input, json& output) {
    try {
        output = json::parse(input);
        
        // 检查解析后的结构
        if (output.is_object() && output.size() > 1000) {
            return false;  // 对象成员过多
        }
        if (output.is_array() && output.size() > 10000) {
            return false;  // 数组元素过多
        }
        return true;
    } catch (const json::parse_error& e) {
        // 记录错误，但不暴露细节给用户
        return false;
    } catch (const json::out_of_range& e) {
        return false;
    }
}

// 安全的数值类型转换
int32_t get_int32_safe(const json& j, const char* key, int32_t default_val = 0) {
    if (!j.contains(key)) return default_val;
    if (!j[key].is_number_integer()) return default_val;
    
    int64_t val = j[key].get<int64_t>();
    if (val < INT32_MIN || val > INT32_MAX) {
        return default_val;  // 溢出保护
    }
    return static_cast<int32_t>(val);
}
```

### 2.2 RapidJSON 安全实践

```cpp
#include "rapidjson/document.h"
#include "rapidjson/error/en.h"

bool parse_rapidjson_safe(const char* json_str, size_t len, 
                          rapidjson::Document& doc) {
    // 设置解析选项
    constexpr auto parse_flags = 
        rapidjson::kParseDefaultFlags |
        rapidjson::kParseNumbersAsStringsFlag;  // 避免数值溢出
    
    doc.Parse<parse_flags>(json_str, len);
    
    if (doc.HasParseError()) {
        // 不要将解析错误信息返回给客户端
        return false;
    }
    
    // 验证文档结构
    if (!doc.IsObject()) {
        return false;
    }
    
    // 限制成员数量
    if (doc.MemberCount() > 100) {
        return false;
    }
    
    return true;
}
```

### 2.3 JSON注入防护

```cpp
// 错误示例：直接拼接JSON字符串
std::string build_json_bad(const std::string& name, int age) {
    return "{\"name\":\"" + name + "\",\"age\":" + std::to_string(age) + "}";
    // name="\"};alert('xss');//" 会破坏JSON结构
}

// 正确示例：使用库构建
std::string build_json_safe(const std::string& name, int age) {
    json j;
    j["name"] = name;  // 自动转义特殊字符
    j["age"] = age;
    return j.dump();
}
```

---

## 三、Protocol Buffers 安全

### 3.1 安全配置

```proto
// 安全的消息定义示例
syntax = "proto3";

message UserRequest {
    // 字段编号从1开始，保留1-15给频繁使用的字段
    string username = 1;
    int32 age = 2;
    
    // 限制字符串长度（需要运行时验证）
    // 最大长度限制通过注释标注
    string email = 3;  // max: 256 bytes
    
    // 使用bytes而非string处理二进制数据
    bytes avatar = 4;  // max: 1MB
}
```

### 3.2 C++安全反序列化

```cpp
#include "user.pb.h"

bool parse_protobuf_safe(const std::string& data, UserRequest& req) {
    // 1. 限制输入大小
    constexpr size_t MAX_MESSAGE_SIZE = 10 * 1024 * 1024;  // 10MB
    if (data.size() > MAX_MESSAGE_SIZE) {
        return false;
    }
    
    // 2. 尝试解析
    if (!req.ParseFromString(data)) {
        return false;
    }
    
    // 3. 验证字段
    if (req.username().size() > 64) {
        return false;  // 用户名过长
    }
    if (req.email().size() > 256) {
        return false;
    }
    if (req.avatar().size() > 1024 * 1024) {
        return false;  // 头像过大
    }
    if (req.age() < 0 || req.age() > 150) {
        return false;  // 年龄范围不合理
    }
    
    return true;
}
```

### 3.3 防止无限递归

```cpp
// 对于递归消息结构，限制嵌套深度
bool validate_recursive_message(const MyMessage& msg, int depth = 0) {
    constexpr int MAX_DEPTH = 10;
    if (depth > MAX_DEPTH) {
        return false;
    }
    
    for (const auto& child : msg.children()) {
        if (!validate_recursive_message(child, depth + 1)) {
            return false;
        }
    }
    return true;
}
```

---

## 四、避免序列化指针

### 4.1 问题说明

```cpp
// 错误示例：序列化指针
struct Node {
    int value;
    Node* next;  // 危险：指针反序列化后无效
};

// 反序列化后，next指向的内存可能已被释放或不存在
```

### 4.2 正确做法

```cpp
// 方案1：序列化数据而非指针
struct NodeData {
    int value;
    int32_t next_index;  // 使用索引代替指针，-1表示空
};

// 方案2：序列化完整对象链
void serialize_list(const Node* head, std::vector<NodeData>& out) {
    std::unordered_map<const Node*, int> node_to_index;
    const Node* curr = head;
    int index = 0;
    
    // 第一遍：建立映射
    while (curr) {
        node_to_index[curr] = index++;
        curr = curr->next;
    }
    
    // 第二遍：序列化
    curr = head;
    while (curr) {
        NodeData data;
        data.value = curr->value;
        data.next_index = curr->next ? node_to_index[curr->next] : -1;
        out.push_back(data);
        curr = curr->next;
    }
}

// 方案3：使用智能指针（仅限单进程内存）
// 注意：智能指针本身不能直接序列化，需要特殊处理引用关系
```

---

## 五、数据校验

### 5.1 校验层级

```
┌─────────────────────────────────────┐
│     语法校验（格式是否正确）           │
├─────────────────────────────────────┤
│     结构校验（字段是否存在、类型正确）  │
├─────────────────────────────────────┤
│     语义校验（值是否在合理范围）        │
├─────────────────────────────────────┤
│     业务校验（业务规则）               │
└─────────────────────────────────────┘
```

### 5.2 校验实现

```cpp
class DataValidator {
public:
    static bool validate_string(const std::string& str, 
                                size_t min_len, size_t max_len,
                                const std::string& charset = "") {
        if (str.size() < min_len || str.size() > max_len) {
            return false;
        }
        if (!charset.empty()) {
            return str.find_first_not_of(charset) == std::string::npos;
        }
        return true;
    }
    
    static bool validate_int(int64_t value, int64_t min_val, int64_t max_val) {
        return value >= min_val && value <= max_val;
    }
    
    static bool validate_email(const std::string& email) {
        // 简化验证：包含@且长度合理
        if (email.size() > 256) return false;
        size_t at_pos = email.find('@');
        return at_pos > 0 && at_pos < email.size() - 1;
    }
    
    static bool validate_url(const std::string& url) {
        // 防止文件协议注入
        if (url.find("file://") == 0) return false;
        if (url.find("javascript:") == 0) return false;
        return url.size() <= 2048;
    }
};
```

---

## 六、版本兼容

### 6.1 版本号设计

```cpp
struct MessageHeader {
    uint16_t version;      // 消息格式版本
    uint16_t flags;        // 标志位
    uint32_t payload_len;  // 载荷长度
    uint32_t checksum;     // 校验和
};

// 版本兼容处理
bool process_message(const std::vector<char>& buffer) {
    if (buffer.size() < sizeof(MessageHeader)) {
        return false;
    }
    
    MessageHeader header;
    memcpy(&header, buffer.data(), sizeof(MessageHeader));
    
    // 版本检查
    constexpr uint16_t MIN_VERSION = 1;
    constexpr uint16_t MAX_VERSION = 3;
    
    if (header.version < MIN_VERSION || header.version > MAX_VERSION) {
        return false;  // 不支持的版本
    }
    
    // 根据版本分发处理逻辑
    switch (header.version) {
        case 1: return process_v1(buffer);
        case 2: return process_v2(buffer);
        case 3: return process_v3(buffer);
        default: return false;
    }
}
```

### 6.2 字段演进策略

```cpp
// 新版本添加字段时使用默认值
struct ConfigV1 {
    int timeout_ms = 5000;
    int retry_count = 3;
};

struct ConfigV2 {
    int timeout_ms = 5000;
    int retry_count = 3;
    int max_connections = 10;  // 新字段，有默认值
    bool enable_cache = true;  // 新字段，有默认值
};

// 反序列化时处理缺失字段
ConfigV2 parse_config(const json& j) {
    ConfigV2 cfg;
    if (j.contains("timeout_ms")) {
        cfg.timeout_ms = get_int32_safe(j, "timeout_ms", 5000);
    }
    if (j.contains("retry_count")) {
        cfg.retry_count = get_int32_safe(j, "retry_count", 3);
    }
    // 新字段：旧数据缺失时使用默认值
    cfg.max_connections = get_int32_safe(j, "max_connections", 10);
    cfg.enable_cache = j.value("enable_cache", true);
    return cfg;
}
```

---

## 七、缓冲区边界检查

### 7.1 反序列化时的边界检查

```cpp
class BufferReader {
private:
    const char* data_;
    size_t size_;
    size_t pos_;
    
public:
    BufferReader(const char* data, size_t size) 
        : data_(data), size_(size), pos_(0) {}
    
    bool read_bytes(void* out, size_t count) {
        if (pos_ + count > size_) {
            return false;  // 越界
        }
        memcpy(out, data_ + pos_, count);
        pos_ += count;
        return true;
    }
    
    bool read_uint32(uint32_t& out) {
        return read_bytes(&out, sizeof(out));
    }
    
    bool read_string(std::string& out, size_t max_len) {
        uint32_t len;
        if (!read_uint32(len)) return false;
        if (len > max_len) return false;  // 长度限制
        if (pos_ + len > size_) return false;
        
        out.assign(data_ + pos_, len);
        pos_ += len;
        return true;
    }
    
    size_t remaining() const { return size_ - pos_; }
};
```

### 7.2 防止整数溢出攻击

```cpp
// 错误示例：长度字段溢出
void process_message_bad(const char* data, size_t len) {
    uint32_t payload_len;
    memcpy(&payload_len, data, sizeof(payload_len));
    
    // payload_len可能为0xFFFFFFFF，导致整数溢出
    char* buffer = new char[payload_len + 1];  // 可能溢出为0或很小的值
    memcpy(buffer, data + 4, len - 4);  // 实际复制可能越界
}

// 正确示例：安全的大小计算
bool process_message_safe(const char* data, size_t len) {
    if (len < sizeof(uint32_t)) return false;
    
    uint32_t payload_len;
    memcpy(&payload_len, data, sizeof(payload_len));
    
    // 检查整数溢出
    constexpr size_t MAX_PAYLOAD = 100 * 1024 * 1024;  // 100MB
    if (payload_len > MAX_PAYLOAD) return false;
    
    // 检查是否有足够的数据
    if (sizeof(uint32_t) + payload_len > len) return false;
    
    // 安全处理
    std::vector<char> buffer(payload_len);
    memcpy(buffer.data(), data + sizeof(uint32_t), payload_len);
    return true;
}
```

---

## 八、敏感数据保护

### 8.1 脱敏序列化

```cpp
class SensitiveDataFilter {
private:
    static std::string mask_string(const std::string& str, 
                                    size_t visible_start, size_t visible_end) {
        if (str.size() <= visible_start + visible_end) {
            return std::string(str.size(), '*');
        }
        return str.substr(0, visible_start) + 
               std::string(str.size() - visible_start - visible_end, '*') +
               str.substr(str.size() - visible_end);
    }
    
public:
    static json filter_for_logging(const json& input) {
        json output = input;
        
        // 脱敏密码字段
        if (output.contains("password")) {
            output["password"] = "******";
        }
        
        // 脱敏手机号：显示前3后4
        if (output.contains("phone")) {
            output["phone"] = mask_string(output["phone"].get<std::string>(), 3, 4);
        }
        
        // 脱敏邮箱：显示用户名首字符和域名
        if (output.contains("email")) {
            std::string email = output["email"].get<std::string>();
            size_t at_pos = email.find('@');
            if (at_pos != std::string::npos && at_pos > 0) {
                output["email"] = email[0] + std::string(at_pos - 1, '*') + 
                                  email.substr(at_pos);
            }
        }
        
        // 移除敏感字段
        output.erase("credit_card");
        output.erase("ssn");
        
        return output;
    }
};
```

### 8.2 加密序列化

```cpp
// 序列化前加密敏感字段
#include <openssl/aes.h>

bool serialize_with_encryption(const UserData& data, 
                                const std::string& key,
                                std::vector<char>& output) {
    json j;
    j["username"] = data.username;
    
    // 敏感字段加密
    std::string plaintext = data.password;
    std::string ciphertext;
    if (!aes_encrypt(plaintext, key, ciphertext)) {
        return false;
    }
    j["password_encrypted"] = base64_encode(ciphertext);
    
    // 内存清理
    OPENSSL_cleanse(&plaintext[0], plaintext.size());
    
    std::string json_str = j.dump();
    output.assign(json_str.begin(), json_str.end());
    return true;
}
```

---

## 九、安全检查清单

### 序列化前检查
- [ ] 确定是否需要序列化敏感数据
- [ ] 敏感数据是否已脱敏或加密
- [ ] 数据大小是否在限制范围内
- [ ] 字段值是否在合法范围内

### 反序列化时检查
- [ ] 输入数据大小是否超过限制
- [ ] 数据格式是否正确（语法校验）
- [ ] 必需字段是否存在
- [ ] 字段类型是否匹配
- [ ] 字符串长度是否在限制内
- [ ] 数值范围是否合理
- [ ] 嵌套深度是否过深
- [ ] 数组/对象成员数量是否过多
- [ ] 版本号是否支持

### 缓冲区安全检查
- [ ] 所有长度字段是否检查上限
- [ ] 长度计算是否防止整数溢出
- [ ] 是否有边界越界检查
- [ ] 内存分配是否检查失败

### 指针处理检查
- [ ] 是否避免了序列化原始指针
- [ ] 是否避免了序列化智能指针的裸指针值
- [ ] 引用关系是否转换为索引或ID

### 版本兼容检查
- [ ] 是否包含版本号
- [ ] 新版本是否能读取旧数据
- [ ] 旧版本是否能合理处理新数据
- [ ] 缺失字段是否有默认值

### 日志与调试检查
- [ ] 日志中是否过滤了敏感数据
- [ ] 错误信息是否泄露实现细节
- [ ] 调试信息是否只在开发模式启用

---

## 十、推荐的序列化格式选择

| 场景 | 推荐格式 | 理由 |
|------|----------|------|
| 跨语言RPC通信 | Protocol Buffers | 类型安全、版本兼容、高效 |
| 配置文件 | JSON | 可读性好、工具支持完善 |
| 高性能内部通信 | FlatBuffers / Cap'n Proto | 零拷贝、无需解析 |
| 持久化存储 | Protocol Buffers + 加密 | 紧凑、支持演进 |
| Web API | JSON | 通用性强、调试方便 |

---

## 参考资料

- OWASP Deserialization Cheat Sheet
- Protocol Buffers Security Best Practices
- nlohmann/json Documentation
- RapidJSON Documentation
- C++ Core Guidelines - I/O safety
