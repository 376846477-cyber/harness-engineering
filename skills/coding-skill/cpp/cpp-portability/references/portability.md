# C++可移植性规范详细参考

## 1. 使用标准C++

优先使用ISO标准C++特性，确保代码在不同编译器和平台上的行为一致。

### 规则

- 使用标准库而非平台特定实现
- 明确指定C++标准版本（C++17/20/23）
- 避免依赖未定义行为或实现定义行为

### 示例

```cpp
// 推荐：使用标准库
#include <filesystem>
#include <chrono>
#include <thread>

// 避免：平台特定API
#ifdef _WIN32
    Sleep(1000);  // Windows API
#else
    usleep(1000000);  // POSIX
#endif

// 推荐：标准库替代
std::this_thread::sleep_for(std::chrono::milliseconds(1000));
```

## 2. 避免平台特定API

将平台相关代码隔离到独立模块中。

### 规则

- 使用抽象层封装平台差异
- 使用Pimpl模式隐藏实现细节
- 优先使用跨平台库

### 示例

```cpp
// platform_utils.hpp
class PlatformUtils {
public:
    static std::string getConfigPath();
    static void setHighPriority();
private:
    struct Impl;
    static std::unique_ptr<Impl> impl_;
};

// platform_utils_windows.cpp
std::string PlatformUtils::getConfigPath() {
    return std::filesystem::getenv("APPDATA");
}

// platform_utils_linux.cpp
std::string PlatformUtils::getConfigPath() {
    return std::filesystem::getenv("HOME") + "/.config";
}
```

## 3. std::filesystem（C++17）

使用现代文件系统库处理路径和文件操作。

### 规则

- 使用`std::filesystem::path`处理路径
- 使用`std::filesystem::path::preferred_separator`或`/`运算符拼接路径
- 使用`u8path()`处理UTF-8路径

### 示例

```cpp
#include <filesystem>
namespace fs = std::filesystem;

// 路径拼接
fs::path configPath = fs::path("config") / "settings.json";

// 获取文件名和扩展名
fs::path file = "/home/user/data.txt";
std::cout << file.stem() << "\n";      // "data"
std::cout << file.extension() << "\n"; // ".txt"

// 跨平台路径比较
if (fs::exists(configPath)) {
    fs::copy_file(src, dst, fs::copy_options::overwrite_existing);
}

// 临时目录
fs::path temp = fs::temp_directory_path();
```

## 4. 字节序处理

正确处理不同架构的字节序差异。

### 规则

- 使用`std::endian`（C++20）检测字节序
- 网络通信使用网络字节序（大端）
- 使用`std::byteswap`（C++23）或自定义函数转换

### 示例

```cpp
#include <bit>
#include <cstdint>

// C++20 字节序检测
constexpr bool is_little_endian() {
    return std::endian::native == std::endian::little;
}

// 手动转换（兼容C++17）
inline uint32_t swap_endian(uint32_t value) {
    return ((value & 0xFF000000) >> 24) |
           ((value & 0x00FF0000) >> 8)  |
           ((value & 0x0000FF00) << 8)  |
           ((value & 0x000000FF) << 24);
}

// 网络字节序转换（标准函数）
#include <cstdint>
uint32_t host_to_network(uint32_t host) {
#if __cpp_lib_byteswap >= 202110L
    if (is_little_endian()) return std::byteswap(host);
    return host;
#else
    return htonl(host);  // POSIX/Windows兼容
#endif
}
```

## 5. 固定宽度类型（cstdint）

使用`<cstdint>`确保类型大小一致。

### 规则

- 使用`int32_t`、`uint64_t`等固定宽度类型
- 使用`intptr_t`、`uintptr_t`存储指针
- 使用`size_t`表示大小和索引

### 示例

```cpp
#include <cstdint>

struct PacketHeader {
    uint16_t version;      // 明确2字节
    uint32_t sequence_id;  // 明确4字节
    uint64_t timestamp;    // 明确8字节
    int32_t  payload_size;
};

// 避免使用平台相关类型
// int        // 大小不固定
// long       // Windows 4字节，Linux 64位 8字节
// long long  // 通常8字节，但不保证

// 存储指针值
void* ptr = malloc(100);
uintptr_t ptr_value = reinterpret_cast<uintptr_t>(ptr);
```

## 6. 路径分隔符

统一处理不同操作系统的路径分隔符。

### 规则

- 使用`std::filesystem::path`自动处理
- 避免硬编码`\\`或`/`
- 解析外部路径时标准化

### 示例

```cpp
#include <filesystem>

fs::path normalize_path(const std::string& input) {
    fs::path p(input);
    p = p.lexically_normal();  // 移除冗余的 . 和 ..
    return p.make_preferred(); // 转换为系统首选分隔符
}

// 配置文件路径
fs::path get_config_file() {
    fs::path config_dir;
    
#ifdef _WIN32
    config_dir = fs::path(std::getenv("APPDATA")) / "MyApp";
#else
    const char* home = std::getenv("HOME");
    if (home) {
        config_dir = fs::path(home) / ".config" / "myapp";
    } else {
        config_dir = fs::path("/etc") / "myapp";
    }
#endif
    
    std::filesystem::create_directories(config_dir);
    return config_dir / "config.json";
}
```

## 7. 编译器差异处理

处理不同编译器的行为差异。

### 规则

- 使用编译器宏检测平台和编译器
- 使用CMake管理编译器选项
- 避免#pragma warning，使用CMake控制

### 示例

```cpp
// 编译器检测
#if defined(__clang__)
    #define COMPILER_CLANG
#elif defined(__GNUC__)
    #define COMPILER_GCC
#elif defined(_MSC_VER)
    #define COMPILER_MSVC
#endif

// 平台检测
#if defined(_WIN32) || defined(_WIN64)
    #define PLATFORM_WINDOWS
#elif defined(__linux__)
    #define PLATFORM_LINUX
#elif defined(__APPLE__)
    #define PLATFORM_MACOS
#endif

// 属性兼容
#if defined(COMPILER_GCC) || defined(COMPILER_CLANG)
    #define LIKELY(x)   __builtin_expect(!!(x), 1)
    #define UNLIKELY(x) __builtin_expect(!!(x), 0)
#elif defined(COMPILER_MSVC)
    #define LIKELY(x)   (x)
    #define UNLIKELY(x) (x)
#endif
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(MyProject CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)  # 使用-std=c++17而非-gnu++17

# 编译器特定选项
if(MSVC)
    add_compile_options(/W4 /utf-8)
else()
    add_compile_options(-Wall -Wextra -Wpedantic)
endif()
```

## 8. 跨平台库推荐

### 文件系统

- **std::filesystem**（C++17，推荐）
- **Boost.Filesystem**（C++03兼容）

### 网络通信

- **Boost.Asio** - 异步网络和I/O
- **POCO** - 完整网络框架

### 线程与并发

- **std::thread**（C++11标准）
- **std::jthread**（C++20，自动join）

### GUI框架

- **Qt** - 功能完整，商业友好
- **wxWidgets** - 原生外观
- **Dear ImGui** - 即时模式GUI

### 其他

- **Boost** - 通用跨平台库集合
- **fmt** - 现代格式化库（C++20标准前置）
- **spdlog** - 跨平台日志库

## 检查清单

### 编码阶段

- [ ] 使用`<cstdint>`固定宽度类型定义二进制数据结构
- [ ] 路径使用`std::filesystem::path`而非字符串
- [ ] 字符串字面量使用`u8""`前缀（C++20前）或明确UTF-8编码
- [ ] 线程休眠使用`std::this_thread::sleep_for`
- [ ] 时间处理使用`<chrono>`而非`time.h`

### 构建阶段

- [ ] CMake明确指定`CMAKE_CXX_STANDARD`
- [ ] 禁用编译器扩展`CMAKE_CXX_EXTENSIONS OFF`
- [ ] 源文件编码设为UTF-8（MSVC: `/utf-8`）
- [ ] CI在多平台（Windows/Linux/macOS）测试

### 代码审查

- [ ] 无硬编码路径分隔符
- [ ] 无平台特定头文件在通用代码中
- [ ] 网络数据有序列化和字节序转换
- [ ] 文件权限处理考虑多平台差异

### 常见陷阱

- `long`类型大小不一致
- `wchar_t`大小不一致（Windows 2字节，Linux 4字节）
- `std::string`编码不明确
- 文件路径大小写敏感（Linux敏感，Windows不敏感）
- 换行符差异（`\n` vs `\r\n`）
