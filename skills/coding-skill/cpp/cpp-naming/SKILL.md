---
name: cpp-naming
description: C++编码规范 - 命名规范。适用于所有C++代码开发场景：(1) 编写任何命名空间、类、函数时的命名检查；(2) 代码审查时检查命名规范；(3) 重构时确保命名符合规范。触发关键词：C++、命名、snake_case、PascalCase、camelCase、常量、变量、函数、类、naming
---

# C++命名规范

基于C++ Core Guidelines和Google C++ Style Guide

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 命名空间小写+下划线 |
| 规则1.2 | 类名PascalCase |
| 规则1.3 | 函数名snake_case |
| 规则1.4 | 变量名snake_case |
| 规则1.5 | 常量kCamelCase或UPPER_SNAKE_CASE |
| 规则1.6 | 宏UPPER_SNAKE_CASE |
| 规则1.7 | 模板参数PascalCase |
| 规则1.8 | 成员变量尾部下划线 |

## 命名风格对照

| 类型 | 风格 | 示例 |
|-----|------|-----|
| 命名空间 | snake_case | `user_service`, `http_client` |
| 类/结构体 | PascalCase | `UserService`, `HttpRequest` |
| 函数 | snake_case | `get_user_by_id()` |
| 局部变量 | snake_case | `user_name`, `record_count` |
| 成员变量 | snake_case_ | `user_name_`, `cache_size_` |
| 常量 | kCamelCase | `kMaxConnections` |
| 宏 | UPPER_SNAKE_CASE | `MAX_BUFFER_SIZE` |
| 模板参数 | PascalCase | `typename Allocator` |
| 枚举值 | kCamelCase或UPPER | `kColorRed` |
| 头文件保护 | UPPER_SNAKE_CASE_H_ | `USER_SERVICE_H_` |

## 常见错误

**错误示例**:
```cpp
int UserID;              // 变量应snake_case
void GetUser();          // 函数应snake_case
class user_service;      // 类应PascalCase
int maxconnections;      // 常量应kCamelCase
```

**正确示例**:
```cpp
int user_id;
void get_user();
class UserService;
constexpr int kMaxConnections = 100;
```

## 详细规范

见 [references/naming.md](references/naming.md)
