# C语言序列化安全规范详细参考

## 一、二进制序列化安全

### 1.1 网络字节序转换

二进制序列化必须使用网络字节序（大端序），确保跨平台兼容性。

```c
#include <stdint.h>
#include <arpa/inet.h>

typedef struct {
    uint32_t magic;
    uint16_t version;
    uint32_t length;
    uint8_t data[];
} packet_header_t;

int serialize_packet(uint8_t *buffer, size_t buf_size, 
                     const packet_header_t *header, const uint8_t *data) {
    if (buf_size < sizeof(uint32_t) * 2 + sizeof(uint16_t) + header->length) {
        return -1;
    }
    
    size_t offset = 0;
    uint32_t magic_net = htonl(header->magic);
    memcpy(buffer + offset, &magic_net, sizeof(magic_net));
    offset += sizeof(magic_net);
    
    uint16_t version_net = htons(header->version);
    memcpy(buffer + offset, &version_net, sizeof(version_net));
    offset += sizeof(version_net);
    
    uint32_t length_net = htonl(header->length);
    memcpy(buffer + offset, &length_net, sizeof(length_net));
    offset += sizeof(length_net);
    
    memcpy(buffer + offset, data, header->length);
    
    return offset + header->length;
}

int deserialize_packet(const uint8_t *buffer, size_t buf_size,
                       packet_header_t *header, uint8_t *data, size_t data_size) {
    if (buf_size < sizeof(uint32_t) * 2 + sizeof(uint16_t)) {
        return -1;
    }
    
    size_t offset = 0;
    memcpy(&header->magic, buffer + offset, sizeof(uint32_t));
    header->magic = ntohl(header->magic);
    offset += sizeof(uint32_t);
    
    memcpy(&header->version, buffer + offset, sizeof(uint16_t));
    header->version = ntohs(header->version);
    offset += sizeof(uint16_t);
    
    memcpy(&header->length, buffer + offset, sizeof(uint32_t));
    header->length = ntohl(header->length);
    offset += sizeof(uint32_t);
    
    if (buf_size < offset + header->length || data_size < header->length) {
        return -1;
    }
    
    memcpy(data, buffer + offset, header->length);
    return 0;
}
```

### 1.2 避免序列化指针

**绝对禁止**直接序列化指针值，指针在反序列化后无效。

```c
typedef struct {
    char *name;
    size_t name_len;
    int age;
} person_t;

int serialize_person(uint8_t *buffer, size_t buf_size, const person_t *person) {
    size_t total_size = sizeof(uint32_t) + person->name_len + sizeof(int);
    if (buf_size < total_size) {
        return -1;
    }
    
    size_t offset = 0;
    uint32_t name_len_net = htonl((uint32_t)person->name_len);
    memcpy(buffer + offset, &name_len_net, sizeof(name_len_net));
    offset += sizeof(name_len_net);
    
    memcpy(buffer + offset, person->name, person->name_len);
    offset += person->name_len;
    
    memcpy(buffer + offset, &person->age, sizeof(int));
    offset += sizeof(int);
    
    return (int)offset;
}

int deserialize_person(const uint8_t *buffer, size_t buf_size, person_t *person) {
    if (buf_size < sizeof(uint32_t)) {
        return -1;
    }
    
    size_t offset = 0;
    uint32_t name_len_net;
    memcpy(&name_len_net, buffer + offset, sizeof(name_len_net));
    person->name_len = ntohl(name_len_net);
    offset += sizeof(name_len_net);
    
    if (buf_size < offset + person->name_len + sizeof(int)) {
        return -1;
    }
    
    person->name = (char *)malloc(person->name_len + 1);
    if (!person->name) {
        return -1;
    }
    
    memcpy(person->name, buffer + offset, person->name_len);
    person->name[person->name_len] = '\0';
    offset += person->name_len;
    
    memcpy(&person->age, buffer + offset, sizeof(int));
    
    return 0;
}
```

## 二、数据完整性校验

### 2.1 Magic Number校验

使用魔数识别有效数据包。

```c
#define PACKET_MAGIC 0x4D475351

typedef struct {
    uint32_t magic;
    uint16_t version;
    uint16_t flags;
    uint32_t checksum;
    uint32_t payload_len;
    uint8_t payload[];
} secure_packet_t;

int validate_magic(const uint8_t *buffer, size_t buf_size) {
    if (buf_size < sizeof(uint32_t)) {
        return 0;
    }
    
    uint32_t received_magic;
    memcpy(&received_magic, buffer, sizeof(uint32_t));
    received_magic = ntohl(received_magic);
    
    return (received_magic == PACKET_MAGIC);
}
```

### 2.2 Checksum校验

简单的累加校验和实现。

```c
uint16_t calculate_checksum(const uint8_t *data, size_t len) {
    uint32_t sum = 0;
    
    while (len > 1) {
        sum += *((const uint16_t *)data);
        data += 2;
        len -= 2;
    }
    
    if (len > 0) {
        sum += *data;
    }
    
    while (sum >> 16) {
        sum = (sum & 0xFFFF) + (sum >> 16);
    }
    
    return (uint16_t)~sum;
}

int verify_packet_checksum(const secure_packet_t *packet, size_t total_size) {
    if (total_size < sizeof(secure_packet_t)) {
        return 0;
    }
    
    uint16_t stored_checksum = packet->checksum;
    uint16_t calculated = calculate_checksum(
        (const uint8_t *)&packet->payload,
        packet->payload_len
    );
    
    return (stored_checksum == calculated);
}
```

### 2.3 CRC32校验

更可靠的循环冗余校验。

```c
static uint32_t crc32_table[256];
static int crc32_table_initialized = 0;

void init_crc32_table(void) {
    if (crc32_table_initialized) return;
    
    for (uint32_t i = 0; i < 256; i++) {
        uint32_t crc = i;
        for (int j = 0; j < 8; j++) {
            crc = (crc >> 1) ^ ((crc & 1) ? 0xEDB88320 : 0);
        }
        crc32_table[i] = crc;
    }
    crc32_table_initialized = 1;
}

uint32_t calculate_crc32(const uint8_t *data, size_t len) {
    if (!crc32_table_initialized) {
        init_crc32_table();
    }
    
    uint32_t crc = 0xFFFFFFFF;
    
    for (size_t i = 0; i < len; i++) {
        crc = crc32_table[(crc ^ data[i]) & 0xFF] ^ (crc >> 8);
    }
    
    return crc ^ 0xFFFFFFFF;
}
```

## 三、版本兼容性设计

### 3.1 版本号管理

```c
#define PROTOCOL_VERSION_MAJOR 2
#define PROTOCOL_VERSION_MINOR 0

typedef struct {
    uint16_t major;
    uint16_t minor;
} version_t;

int is_version_compatible(const version_t *received) {
    if (received->major != PROTOCOL_VERSION_MAJOR) {
        return 0;
    }
    
    if (received->minor > PROTOCOL_VERSION_MINOR) {
        return 0;
    }
    
    return 1;
}

typedef struct {
    uint32_t magic;
    version_t version;
    uint32_t flags;
    uint32_t header_len;
    uint8_t header_data[];
} versioned_packet_t;

int parse_versioned_packet(const uint8_t *buffer, size_t buf_size) {
    versioned_packet_t packet;
    
    if (buf_size < sizeof(uint32_t) * 2 + sizeof(version_t)) {
        return -1;
    }
    
    memcpy(&packet.magic, buffer, sizeof(uint32_t));
    packet.magic = ntohl(packet.magic);
    
    if (packet.magic != PACKET_MAGIC) {
        return -1;
    }
    
    memcpy(&packet.version, buffer + sizeof(uint32_t), sizeof(version_t));
    packet.version.major = ntohs(packet.version.major);
    packet.version.minor = ntohs(packet.version.minor);
    
    if (!is_version_compatible(&packet.version)) {
        return -1;
    }
    
    return 0;
}
```

## 四、缓冲区边界检查

### 4.1 安全的数据读取

```c
typedef struct {
    const uint8_t *data;
    size_t size;
    size_t offset;
} buffer_reader_t;

int buffer_reader_init(buffer_reader_t *reader, const uint8_t *data, size_t size) {
    if (!reader || !data || size == 0) {
        return -1;
    }
    
    reader->data = data;
    reader->size = size;
    reader->offset = 0;
    
    return 0;
}

int buffer_read_uint32(buffer_reader_t *reader, uint32_t *value) {
    if (!reader || !value) {
        return -1;
    }
    
    if (reader->offset + sizeof(uint32_t) > reader->size) {
        return -1;
    }
    
    memcpy(value, reader->data + reader->offset, sizeof(uint32_t));
    *value = ntohl(*value);
    reader->offset += sizeof(uint32_t);
    
    return 0;
}

int buffer_read_bytes(buffer_reader_t *reader, uint8_t *dest, size_t len) {
    if (!reader || !dest) {
        return -1;
    }
    
    if (reader->offset + len > reader->size) {
        return -1;
    }
    
    memcpy(dest, reader->data + reader->offset, len);
    reader->offset += len;
    
    return 0;
}
```

### 4.2 最大长度限制

```c
#define MAX_PACKET_SIZE (1024 * 1024)
#define MAX_STRING_LEN (64 * 1024)
#define MAX_ARRAY_ELEMENTS 10000

int parse_length_prefixed_string(buffer_reader_t *reader, char **str, size_t *len) {
    uint32_t str_len;
    
    if (buffer_read_uint32(reader, &str_len) != 0) {
        return -1;
    }
    
    if (str_len > MAX_STRING_LEN) {
        return -1;
    }
    
    char *buffer = (char *)malloc(str_len + 1);
    if (!buffer) {
        return -1;
    }
    
    if (buffer_read_bytes(reader, (uint8_t *)buffer, str_len) != 0) {
        free(buffer);
        return -1;
    }
    
    buffer[str_len] = '\0';
    *str = buffer;
    *len = str_len;
    
    return 0;
}
```

## 五、JSON解析安全

### 5.1 cJSON安全使用

```c
#include < cJSON.h>

#define MAX_JSON_DEPTH 20
#define MAX_JSON_STRING_LEN (64 * 1024)

int safe_parse_json(const char *json_str, size_t json_len, cJSON **root) {
    if (!json_str || json_len == 0 || !root) {
        return -1;
    }
    
    if (json_len > MAX_JSON_STRING_LEN) {
        return -1;
    }
    
    cJSON *parsed = cJSON_ParseWithLengthOpts(
        json_str, 
        json_len,
        NULL,
        0
    );
    
    if (!parsed) {
        const char *error_ptr = cJSON_GetErrorPtr();
        if (error_ptr) {
            fprintf(stderr, "JSON parse error at: %s\n", error_ptr);
        }
        return -1;
    }
    
    *root = parsed;
    return 0;
}

int safe_get_string(cJSON *obj, const char *key, char *buffer, size_t buf_size) {
    cJSON *item = cJSON_GetObjectItemCaseSensitive(obj, key);
    if (!cJSON_IsString(item)) {
        return -1;
    }
    
    size_t item_len = strlen(item->valuestring);
    if (item_len >= buf_size) {
        return -1;
    }
    
    strncpy(buffer, item->valuestring, buf_size - 1);
    buffer[buf_size - 1] = '\0';
    
    return 0;
}

int safe_get_array_size(cJSON *array, size_t *size) {
    if (!cJSON_IsArray(array)) {
        return -1;
    }
    
    int count = cJSON_GetArraySize(array);
    if (count < 0 || count > MAX_ARRAY_ELEMENTS) {
        return -1;
    }
    
    *size = (size_t)count;
    return 0;
}
```

## 六、防止反序列化攻击

### 6.1 输入验证框架

```c
typedef struct {
    uint32_t type;
    uint32_t length;
    uint32_t checksum;
    uint8_t *data;
} deserialized_object_t;

int validate_deserialized_object(const deserialized_object_t *obj) {
    if (obj->type > 100) {
        return -1;
    }
    
    if (obj->length > MAX_PACKET_SIZE) {
        return -1;
    }
    
    uint32_t calc_checksum = calculate_crc32(obj->data, obj->length);
    if (calc_checksum != obj->checksum) {
        return -1;
    }
    
    return 0;
}

int safe_deserialize(const uint8_t *buffer, size_t buf_size,
                     deserialized_object_t *obj) {
    buffer_reader_t reader;
    
    if (buffer_reader_init(&reader, buffer, buf_size) != 0) {
        return -1;
    }
    
    if (buffer_read_uint32(&reader, &obj->type) != 0) {
        return -1;
    }
    
    if (buffer_read_uint32(&reader, &obj->length) != 0) {
        return -1;
    }
    
    if (obj->length > MAX_PACKET_SIZE) {
        return -1;
    }
    
    obj->data = (uint8_t *)malloc(obj->length);
    if (!obj->data) {
        return -1;
    }
    
    if (buffer_read_bytes(&reader, obj->data, obj->length) != 0) {
        free(obj->data);
        obj->data = NULL;
        return -1;
    }
    
    if (buffer_read_uint32(&reader, &obj->checksum) != 0) {
        free(obj->data);
        obj->data = NULL;
        return -1;
    }
    
    if (validate_deserialized_object(obj) != 0) {
        free(obj->data);
        obj->data = NULL;
        return -1;
    }
    
    return 0;
}
```

### 6.2 类型安全检查

```c
typedef enum {
    FIELD_TYPE_UINT32 = 1,
    FIELD_TYPE_STRING = 2,
    FIELD_TYPE_BYTES = 3,
    FIELD_TYPE_ARRAY = 4
} field_type_t;

typedef struct {
    field_type_t type;
    uint32_t offset;
    uint32_t size;
} field_descriptor_t;

int validate_field_type(const field_descriptor_t *desc,
                        const uint8_t *buffer, size_t buf_size) {
    if (!desc || !buffer) {
        return -1;
    }
    
    if (desc->offset >= buf_size) {
        return -1;
    }
    
    if (desc->offset + desc->size > buf_size) {
        return -1;
    }
    
    switch (desc->type) {
        case FIELD_TYPE_UINT32:
            if (desc->size != sizeof(uint32_t)) {
                return -1;
            }
            break;
            
        case FIELD_TYPE_STRING:
            if (desc->size > MAX_STRING_LEN) {
                return -1;
            }
            if (buffer[desc->offset + desc->size - 1] != '\0') {
                return -1;
            }
            break;
            
        case FIELD_TYPE_BYTES:
            if (desc->size > MAX_PACKET_SIZE) {
                return -1;
            }
            break;
            
        case FIELD_TYPE_ARRAY:
            if (desc->size > MAX_ARRAY_ELEMENTS * sizeof(uint32_t)) {
                return -1;
            }
            break;
            
        default:
            return -1;
    }
    
    return 0;
}
```

## 七、检查清单

### 7.1 序列化前检查

- [ ] 使用网络字节序（htonl/htons）
- [ ] 不序列化指针，序列化指针指向的数据
- [ ] 包含魔数标识
- [ ] 添加版本号字段
- [ ] 计算并添加校验和或CRC
- [ ] 敏感数据已脱敏或加密
- [ ] 所有字符串以长度前缀形式序列化

### 7.2 反序列化前检查

- [ ] 验证魔数是否正确
- [ ] 检查版本兼容性
- [ ] 验证数据总长度不超过最大限制
- [ ] 验证校验和或CRC
- [ ] 检查所有字段长度
- [ ] 限制字符串长度
- [ ] 限制数组元素数量
- [ ] 验证字段类型一致性

### 7.3 缓冲区安全检查

- [ ] 所有memcpy前检查目标缓冲区大小
- [ ] 使用安全的buffer_reader结构
- [ ] 设置合理的MAX_*常量
- [ ] 检查整数溢出可能性
- [ ] 使用strncpy替代strcpy
- [ ] 确保字符串以'\0'结尾

### 7.4 JSON安全检查

- [ ] 限制JSON输入长度
- [ ] 使用cJSON_ParseWithLengthOpts
- [ ] 验证字段类型（cJSON_IsString等）
- [ ] 限制字符串字段长度
- [ ] 限制数组元素数量
- [ ] 检查JSON嵌套深度
- [ ] 使用cJSON_GetObjectItemCaseSensitive

### 7.5 内存管理检查

- [ ] 反序列化成功后释放临时缓冲区
- [ ] 错误路径释放已分配内存
- [ ] 使用free后设置指针为NULL
- [ ] 敏感数据使用后memset清零
- [ ] 避免内存泄漏

### 7.6 错误处理检查

- [ ] 所有错误路径返回明确错误码
- [ ] 错误日志不包含敏感数据
- [ ] 校验失败立即拒绝数据
- [ ] 版本不兼容时优雅降级
- [ ] 记录异常反序列化尝试

## 八、常用安全库

| 库名 | 用途 | 链接 |
|------|------|------|
| cJSON | JSON解析 | https://github.com/DaveGamble/cJSON |
| protobuf-c | Protocol Buffers | https://github.com/protobuf-c/protobuf-c |
| msgpack-c | MessagePack | https://github.com/msgpack/msgpack-c |
| zlib | 数据压缩与CRC | https://zlib.net/ |
| OpenSSL | 加密序列化数据 | https://www.openssl.org/ |
