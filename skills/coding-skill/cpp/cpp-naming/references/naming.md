# C++命名规范详细参考

## 一、命名空间（snake_case）

**规则：**
- 使用全小写字母和下划线分隔
- 名称应简短、有意义，反映模块或功能
- 避免过深的命名空间嵌套（建议不超过3层）

**示例：**
```cpp
namespace user_service {
    namespace detail {
        void internal_helper();
    }
}

namespace http_client {
    class Connection;
}

namespace my_company::project::module {
    void process_data();
}
```

**反例：**
```cpp
namespace UserService {}     // 错误：应使用snake_case
namespace HTTPClient {}      // 错误：应使用snake_case
namespace a {}               // 错误：名称过于简短
namespace very::deep::nested::level::internal {}  // 错误：嵌套过深
```

---

## 二、类名（PascalCase）

**规则：**
- 每个单词首字母大写，无分隔符
- 使用名词或名词短语
- 避免使用前缀（如C、T）或后缀（如Class）

**示例：**
```cpp
class UserService {};
class HttpRequest {};
class TcpConnection {};
class XmlParser {};
class ThreadPool {};
struct Point2D {};
struct CustomerInfo {};
```

**反例：**
```cpp
class user_service {}       // 错误：应使用PascalCase
class CHttpRequest {}       // 错误：不应有前缀
class Xmlparser {}          // 错误：Parser的P应大写
class User_Class {}         // 错误：不应有后缀
```

---

## 三、函数名（snake_case）

**规则：**
- 使用全小写字母和下划线分隔
- 以动词开头，表达操作意图
- 布尔返回函数使用 is/has/can/should 等前缀
- Getter可省略get前缀，Setter使用set前缀

**示例：**
```cpp
void get_user_by_id(int id);
void set_user_name(const std::string& name);
bool is_valid() const;
bool has_permission() const;
bool can_write() const;
bool should_retry() const;
void calculate_total_price();
void send_http_request();
```

**反例：**
```cpp
void GetUserById() {}       // 错误：应使用snake_case
void getUserById() {}       // 错误：应使用snake_case
void valid() {}             // 错误：布尔函数应有is/has前缀
void getData() {}           // 风格不佳：可简化为data()
```

---

## 四、变量名（snake_case）

**规则：**
- 使用全小写字母和下划线分隔
- 成员变量尾部加下划线
- 避免使用单字母名称（循环变量除外）
- 布尔变量使用is/has/can等前缀

**示例：**
```cpp
int user_count;
std::string file_name;
bool is_running;

class UserService {
private:
    int user_id_;           // 成员变量尾部下划线
    std::string user_name_;
    bool is_active_;
    
public:
    int user_id() const { return user_id_; }
};
```

**反例：**
```cpp
int UserCount;              // 错误：应使用snake_case
int userCount;              // 错误：应使用snake_case
int m_user_id;              // 风格过时：应使用尾部下划线
int _user_id;               // 错误：不应使用前导下划线
int x;                      // 风格不佳：名称无意义（循环变量除外）
```

---

## 五、常量

**规则：**
- `constexpr`/`const` 常量：`kCamelCase`
- 宏常量：`UPPER_SNAKE_CASE`
- 文件作用域常量以k开头

**示例：**
```cpp
constexpr int kMaxSize = 1024;
constexpr double kPi = 3.14159;
const std::string kDefaultName = "unknown";

#define MAX_BUFFER_SIZE 4096
#define DEFAULT_TIMEOUT_MS 5000

namespace {
constexpr int kInternalLimit = 100;
}
```

**反例：**
```cpp
constexpr int MAX_SIZE = 1024;      // 风格不一致：应使用kCamelCase
const int max_size = 1024;          // 错误：应使用kCamelCase
#define kMaxSize 1024               // 错误：宏应使用UPPER_SNAKE_CASE
```

---

## 六、宏（UPPER_SNAKE_CASE）

**规则：**
- 全大写字母和下划线分隔
- 宏名称应有意义
- 优先使用`constexpr`/`inline`函数替代宏

**示例：**
```cpp
#define DEBUG_MODE 1
#define PLATFORM_WINDOWS 1
#define MAX_CONNECTION_COUNT 100
#define SAFE_DELETE(ptr) do { delete (ptr); (ptr) = nullptr; } while(0)
#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]))
```

**反例：**
```cpp
#define debug_mode 1               // 错误：应使用UPPER_SNAKE_CASE
#define MaxConnectionCount 100     // 错误：应使用UPPER_SNAKE_CASE
#define M 1024                     // 风格不佳：名称无意义
```

---

## 七、模板参数（PascalCase）

**规则：**
- 使用PascalCase
- 类型参数使用描述性名称
- 简单模板可使用单字母T、U、V

**示例：**
```cpp
template<typename Allocator>
class Container {};

template<typename Key, typename Value>
class HashMap {};

template<typename T, typename U>
auto add(T a, U b) -> decltype(a + b);

template<int kBufferSize>
class Buffer {};

template<typename InputIterator, typename OutputIterator>
void copy(InputIterator first, InputIterator last, OutputIterator result);
```

**反例：**
```cpp
template<typename allocator>       // 错误：应使用PascalCase
template<typename TYPE>            // 错误：应使用PascalCase
template<typename my_type>         // 错误：应使用PascalCase
```

---

## 八、枚举值

**规则：**
- 枚举类名使用PascalCase
- 枚举值使用`kCamelCase`或`UPPER_SNAKE_CASE`
- 优先使用`enum class`

**示例：**
```cpp
enum class Color {
    kRed,
    kGreen,
    kBlue
};

enum class HttpStatus {
    kOk = 200,
    kNotFound = 404,
    kInternalServerError = 500
};

enum class LogLevel {
    DEBUG,
    INFO,
    WARNING,
    ERROR
};

// 传统枚举（兼容旧代码）
enum ErrorCode {
    ERROR_NONE = 0,
    ERROR_INVALID_PARAM = 1,
    ERROR_OUT_OF_MEMORY = 2
};
```

**反例：**
```cpp
enum class color {};               // 错误：应使用PascalCase
enum class Color { red, green };   // 风格不一致：应使用kCamelCase
enum { red, green, blue };         // 风格不佳：应使用enum class
```

---

## 九、头文件保护（UPPER_SNAKE_CASE_H_）

**规则：**
- 优先使用`#pragma once`
- 传统保护符使用命名空间路径的`UPPER_SNAKE_CASE_H_`
- 避免使用前导下划线

**示例：**
```cpp
// 方式1：#pragma once（推荐）
#pragma once

namespace user_service {
class UserService {};
}

// 方式2：传统保护符
#ifndef USER_SERVICE_USER_SERVICE_H_
#define USER_SERVICE_USER_SERVICE_H_

namespace user_service {
class UserService {};
}

#endif  // USER_SERVICE_USER_SERVICE_H_

// 文件路径：src/http_client/connection.h
#ifndef SRC_HTTP_CLIENT_CONNECTION_H_
#define SRC_HTTP_CLIENT_CONNECTION_H_

namespace http_client {
class Connection {};
}

#endif  // SRC_HTTP_CLIENT_CONNECTION_H_
```

**反例：**
```cpp
#ifndef _USER_SERVICE_H_           // 错误：前导下划线保留给编译器
#define _USER_SERVICE_H_

#ifndef UserService_h_             // 错误：应使用UPPER_SNAKE_CASE
#define UserService_h_
```

---

## 检查清单

| 类型 | 规范 | 示例 | 检查项 |
|------|------|------|--------|
| 命名空间 | snake_case | `user_service` | 全小写+下划线，无缩写 |
| 类名 | PascalCase | `UserService` | 名词，无前缀后缀 |
| 函数名 | snake_case | `get_user_by_id()` | 动词开头，布尔用is/has |
| 变量名 | snake_case | `user_count` | 有意义，避免单字母 |
| 成员变量 | snake_case_ | `user_id_` | 尾部下划线 |
| 常量 | kCamelCase | `kMaxSize` | 以k开头 |
| 宏常量 | UPPER_SNAKE_CASE | `MAX_SIZE` | 全大写+下划线 |
| 模板参数 | PascalCase | `Allocator` | 类型用描述名，简单用T |
| 枚举类名 | PascalCase | `Color` | 使用enum class |
| 枚举值 | kCamelCase | `kRed` | 统一风格 |
| 头文件保护 | UPPER_SNAKE_CASE_H_ | `USER_SERVICE_H_` | 对应文件路径 |

**快速检查命令：**
```bash
# 检查命名不符合规范的类（应为PascalCase）
grep -E "class\s+[a-z]" *.hpp *.cpp

# 检查命名不符合规范的函数（应为snake_case）
grep -E "void\s+[A-Z]|int\s+[A-Z]|bool\s+[A-Z]" *.hpp *.cpp

# 检查成员变量缺少尾部下划线
grep -E "private:|protected:" -A 5 *.hpp | grep -v "_;"
```
