---
name: cpp-security-serialization
description: C++编码规范 - 序列化安全规范。适用于所有C++代码开发场景：(1) 反序列化安全；(2) 数据校验；(3) 敏感数据保护。触发关键词：C++、序列化、反序列化、deserialize、protobuf、JSON、安全
---

# C++序列化安全规范

基于安全编程规范

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 反序列化前必须校验数据 |
| 规则1.2 | 禁止反序列化不可信的二进制数据 |
| 规则1.3 | 序列化数据必须包含版本信息 |
| 规则1.4 | 禁止序列化未加密的敏感数据 |

## 安全序列化

```cpp
// 使用protobuf（推荐）
#include "user.pb.h"

user::User user;
if (!user.ParseFromString(data)) {
    return Error("反序列化失败");
}

// 校验字段
if (user.name().size() > kMaxNameLength) {
    return Error("名称过长");
}
```

## JSON安全

```cpp
// 使用nlohmann/json
#include <nlohmann/json.hpp>

auto data = nlohmann::json::parse(input);
if (!data.contains("name") || !data["name"].is_string()) {
    return Error("缺少name字段");
}
std::string name = data["name"].get<std::string>();
if (name.size() > kMaxNameLength) {
    return Error("名称过长");
}
```

## 详细规范

见 [references/security-serialization.md](references/security-serialization.md)
