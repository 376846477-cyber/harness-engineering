---
name: cpp-portability
description: C++编码规范 - 可移植性规范。适用于所有C++代码开发场景：(1) 代码编写时考虑可移植性；(2) 代码审查时检查可移植性；(3) 跨平台实现。触发关键词：C++、可移植、portability、平台、跨平台、编译器、filesystem
---

# C++可移植性规范

基于Clean Code指导书

## 核心原则

C++代码应尽量使用标准库和跨平台库，避免引入平台相关的依赖。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 使用std::filesystem处理路径 |
| 1.2 | 明确字符编码 |
| 1.3 | 使用标准库替代平台API |
| 1.4 | 注意编译器兼容性 |
| 2.1 | 使用跨平台构建系统 |
| 3.1 | 字节序处理 |

## 路径处理

```cpp
#include <filesystem>

namespace fs = std::filesystem;

fs::path config_path = fs::path("data") / "files" / "config.txt";
std::string content = read_file(config_path);

// 临时目录
fs::path temp = fs::temp_directory_path();
```

## 字符编码

```cpp
#include <codecvt>  // C++17已弃用，使用第三方库
#include <locale>

// 使用UTF-8编码
std::string utf8_str = u8"你好世界";

// 使用C++20 char8_t
#if __cplusplus >= 202002L
std::u8string u8str = u8"你好世界";
#endif
```

## 详细规范

见 [references/portability.md](references/portability.md)

## 检查清单

- [ ] 路径是否使用std::filesystem？
- [ ] 是否依赖平台特定API？
- [ ] 编译器是否兼容？
- [ ] 字节序是否处理？
- [ ] 构建系统是否跨平台？
