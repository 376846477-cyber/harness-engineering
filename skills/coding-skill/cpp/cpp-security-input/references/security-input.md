# C++输入校验与安全规范详细参考

## 一、安全红线规则

### 绝对禁止

1. **禁止使用C风格危险函数**：gets、sprintf、strcpy、strcat、scanf("%s")
2. **禁止用户输入作为格式化字符串**：printf(user_input)、std::cout << user_input（非字符串字面量格式）
3. **禁止直接拼接命令执行**：system(cmd)、popen(cmd, "r")
4. **禁止未验证的路径操作**：直接使用用户提供的文件路径
5. **禁止裸指针管理资源**：使用智能指针替代
6. **禁止整数溢出后继续操作**：使用安全整数运算库

### 必须遵守

1. 所有字符串操作使用std::string
2. 所有容器操作使用STL容器（std::vector、std::array）
3. 所有资源管理使用RAII
4. 所有外部输入必须验证（类型、长度、范围、格式）
5. 所有错误通过异常或std::optional处理

---

## 二、危险函数→安全替代对照表

### 字符串处理

| 危险函数 | 风险 | 安全替代 | 说明 |
|---------|------|---------|------|
| `char[] + strcpy` | 缓冲区溢出 | `std::string` | 自动管理内存 |
| `char[] + sprintf` | 缓冲区溢出 | `std::format`(C++20) | 类型安全 |
| `char[] + strcat` | 缓冲区溢出 | `std::string::append` | 自动扩展 |
| `scanf("%s", buf)` | 缓冲区溢出 | `std::getline` | 安全读取 |
| `gets(buf)` | 缓冲区溢出 | `std::getline` | 已从C++11移除 |

### 格式化输出

| 危险用法 | 风险 | 安全替代 | 说明 |
|---------|------|---------|------|
| `printf(user_input)` | 格式化字符串漏洞 | `std::cout << user_input` | 类型安全 |
| `fprintf(fp, user_input)` | 格式化字符串漏洞 | `fmt::print(fp, "{}", user_input)` | 现代格式化 |

---

## 三、详细代码示例

### 3.1 缓冲区溢出防护

#### 错误示例

```cpp
void unsafe_copy(const char* input) {
    char buffer[256];
    strcpy(buffer, input);
    std::cout << buffer << std::endl;
}

void unsafe_format(const std::string& name, int age) {
    char buffer[100];
    sprintf(buffer, "Name: %s, Age: %d", name.c_str(), age);
    std::cout << buffer << std::endl;
}
```

#### 正确示例

```cpp
#include <string>
#include <format>
#include <iostream>

void safe_copy(const std::string& input) {
    std::string buffer = input;
    std::cout << buffer << std::endl;
}

void safe_format(const std::string& name, int age) {
    std::string buffer = std::format("Name: {}, Age: {}", name, age);
    std::cout << buffer << std::endl;
}

void safe_read_input() {
    std::string line;
    if (std::getline(std::cin, line)) {
        std::cout << "Read: " << line << std::endl;
    }
}
```

### 3.2 使用std::vector替代C数组

#### 错误示例

```cpp
void unsafe_array(int* data, size_t count) {
    int buffer[100];
    for (size_t i = 0; i < count; ++i) {
        buffer[i] = data[i];
    }
}

void unsafe_dynamic(size_t count) {
    int* arr = new int[count];
    delete[] arr;
}
```

#### 正确示例

```cpp
#include <vector>
#include <algorithm>

void safe_array(const std::vector<int>& data) {
    std::vector<int> buffer = data;
}

void safe_dynamic(size_t count) {
    std::vector<int> arr(count);
    auto buffer = std::make_unique<int[]>(count);
}

void safe_bounds_check(const std::vector<int>& data, size_t index) {
    if (index < data.size()) {
        int value = data.at(index);
    }
}
```

### 3.3 SQL注入防护

#### 错误示例

```cpp
#include <sqlite3.h>

void unsafe_query(sqlite3* db, const std::string& username) {
    std::string sql = "SELECT * FROM users WHERE name='" + username + "'";
    sqlite3_exec(db, sql.c_str(), nullptr, nullptr, nullptr);
}
```

#### 正确示例

```cpp
#include <sqlite3.h>
#include <memory>

void safe_query(sqlite3* db, const std::string& username) {
    const char* sql = "SELECT * FROM users WHERE name = ?";
    
    sqlite3_stmt* stmt = nullptr;
    int rc = sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr);
    if (rc != SQLITE_OK) {
        std::cerr << "Prepare failed: " << sqlite3_errmsg(db) << std::endl;
        return;
    }
    
    auto stmt_guard = std::unique_ptr<sqlite3_stmt, decltype(&sqlite3_finalize)>(
        stmt, &sqlite3_finalize);
    
    rc = sqlite3_bind_text(stmt, 1, username.c_str(), -1, SQLITE_TRANSIENT);
    if (rc != SQLITE_OK) {
        std::cerr << "Bind failed" << std::endl;
        return;
    }
    
    while ((rc = sqlite3_step(stmt)) == SQLITE_ROW) {
        const unsigned char* name = sqlite3_column_text(stmt, 0);
        std::cout << "User: " << name << std::endl;
    }
}

// 使用ORM框架（推荐）
#include <odb/database.hxx>
#include <odb/transaction.hxx>

void safe_query_orm(odb::database& db, const std::string& username) {
    odb::transaction t(db.begin());
    
    auto user = db.find<User>(odb::query<User>::name == username);
    if (user) {
        std::cout << "Found user: " << user->name << std::endl;
    }
    
    t.commit();
}
```

### 3.4 命令注入防护

#### 错误示例

```cpp
void unsafe_ping(const std::string& host) {
    std::string cmd = "ping -c 4 " + host;
    system(cmd.c_str());
}
```

#### 正确示例

```cpp
#include <cstdlib>
#include <unistd.h>
#include <sys/wait.h>
#include <regex>

bool is_valid_hostname(const std::string& host) {
    static const std::regex pattern(
        R"(^([a-zA-Z0-9]([a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}$)"
    );
    std::smatch match;
    return std::regex_match(host, match, pattern);
}

int safe_ping(const std::string& host) {
    if (!is_valid_hostname(host)) {
        std::cerr << "Invalid hostname" << std::endl;
        return -1;
    }
    
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork failed");
        return -1;
    }
    
    if (pid == 0) {
        std::array<const char*, 5> args = {"ping", "-c", "4", host.c_str(), nullptr};
        execvp("ping", const_cast<char* const*>(args.data()));
        _exit(127);
    }
    
    int status;
    waitpid(pid, &status, 0);
    return WEXITSTATUS(status);
}

// C++17 使用std::filesystem和更安全的进程执行
#include <filesystem>
#include <boost/process.hpp>

namespace bp = boost::process;
namespace fs = std::filesystem;

int safe_ping_modern(const std::string& host) {
    if (!is_valid_hostname(host)) {
        return -1;
    }
    
    try {
        bp::child c("ping", "-c", "4", host);
        c.wait();
        return c.exit_code();
    } catch (const std::exception& e) {
        std::cerr << "Execution failed: " << e.what() << std::endl;
        return -1;
    }
}
```

### 3.5 文件路径安全

#### 错误示例

```cpp
void unsafe_read_file(const std::string& filename) {
    std::string path = "/var/data/" + filename;
    std::ifstream file(path);
    std::string content((std::istreambuf_iterator<char>(file)),
                        std::istreambuf_iterator<char>());
}
```

#### 正确示例

```cpp
#include <filesystem>
#include <fstream>

namespace fs = std::filesystem;

std::optional<std::string> safe_read_file(const std::string& filename) {
    const fs::path base_dir = "/var/data";
    
    if (filename.empty() || filename.find("..") != std::string::npos) {
        std::cerr << "Invalid filename" << std::endl;
        return std::nullopt;
    }
    
    fs::path user_path(filename);
    if (user_path.is_absolute()) {
        std::cerr << "Absolute path not allowed" << std::endl;
        return std::nullopt;
    }
    
    fs::path full_path = base_dir / user_path;
    
    std::error_code ec;
    fs::path canonical_path = fs::canonical(full_path, ec);
    if (ec) {
        std::cerr << "Path resolution failed" << std::endl;
        return std::nullopt;
    }
    
    auto [it, end] = std::mismatch(canonical_path.begin(), canonical_path.end(),
                                    base_dir.begin(), base_dir.end());
    if (it != canonical_path.end()) {
        std::cerr << "Path escape detected" << std::endl;
        return std::nullopt;
    }
    
    std::ifstream file(canonical_path);
    if (!file) {
        return std::nullopt;
    }
    
    return std::string((std::istreambuf_iterator<char>(file)),
                       std::istreambuf_iterator<char>());
}

bool is_path_within_directory(const fs::path& path, const fs::path& dir) {
    std::error_code ec;
    auto canonical_path = fs::canonical(path, ec);
    if (ec) return false;
    
    auto canonical_dir = fs::canonical(dir, ec);
    if (ec) return false;
    
    auto path_str = canonical_path.string();
    auto dir_str = canonical_dir.string();
    
    return path_str.rfind(dir_str, 0) == 0;
}
```

### 3.6 格式化字符串安全

#### 错误示例

```cpp
void unsafe_log(const char* user_message) {
    printf(user_message);
}

void unsafe_syslog(int priority, const std::string& msg) {
    syslog(priority, msg.c_str());
}
```

#### 正确示例

```cpp
#include <iostream>
#include <format>
#include <syslog.h>

void safe_log(const std::string& user_message) {
    std::cout << user_message << std::endl;
}

void safe_syslog(int priority, const std::string& msg) {
    syslog(priority, "%s", msg.c_str());
}

template<typename... Args>
void safe_format_log(std::format_string<Args...> fmt, Args&&... args) {
    std::string message = std::format(fmt, std::forward<Args>(args)...);
    std::cout << message << std::endl;
}
```

### 3.7 整数溢出检测

#### 错误示例

```cpp
void unsafe_allocate(size_t count, size_t item_size) {
    size_t total = count * item_size;
    int* buf = new int[total];
    delete[] buf;
}
```

#### 正确示例

```cpp
#include <limits>
#include <stdexcept>
#include <memory>

namespace safe_math {
    template<typename T>
    std::optional<T> safe_add(T a, T b) {
        if (a > std::numeric_limits<T>::max() - b) {
            return std::nullopt;
        }
        return a + b;
    }
    
    template<typename T>
    std::optional<T> safe_mul(T a, T b) {
        if (a == 0 || b == 0) return T{0};
        if (a > std::numeric_limits<T>::max() / b) {
            return std::nullopt;
        }
        return a * b;
    }
}

std::unique_ptr<int[]> safe_allocate(size_t count, size_t item_size) {
    auto total = safe_math::safe_mul(count, item_size);
    if (!total) {
        throw std::overflow_error("Integer overflow in allocation");
    }
    
    return std::make_unique<int[]>(count);
}

// C++26 将提供 std::checked_arithmetic
```

### 3.8 使用std::string替代C字符串

#### 错误示例

```cpp
void unsafe_string_ops(const char* input) {
    char* copy = strdup(input);
    char* token = strtok(copy, " ");
    while (token != nullptr) {
        std::cout << token << std::endl;
        token = strtok(nullptr, " ");
    }
    free(copy);
}
```

#### 正确示例

```cpp
#include <string>
#include <string_view>
#include <sstream>
#include <algorithm>

void safe_string_ops(std::string_view input) {
    std::string copy(input);
    std::istringstream iss(copy);
    std::string token;
    while (iss >> token) {
        std::cout << token << std::endl;
    }
}

std::vector<std::string> safe_split(const std::string& str, char delimiter) {
    std::vector<std::string> tokens;
    std::string token;
    std::istringstream tokenStream(str);
    while (std::getline(tokenStream, token, delimiter)) {
        tokens.push_back(token);
    }
    return tokens;
}

std::string safe_trim(const std::string& str) {
    auto start = std::find_if_not(str.begin(), str.end(), ::isspace);
    auto end = std::find_if_not(str.rbegin(), str.rend(), ::isspace).base();
    return (start < end) ? std::string(start, end) : std::string();
}
```

### 3.9 输入验证

#### 错误示例

```cpp
void unsafe_process_input(const std::string& input) {
    int value = std::stoi(input);
    std::cout << "Value: " << value << std::endl;
}
```

#### 正确示例

```cpp
#include <string>
#include <regex>
#include <optional>
#include <algorithm>

class InputValidator {
public:
    static std::optional<int> parse_int(const std::string& input,
                                        int min = std::numeric_limits<int>::min(),
                                        int max = std::numeric_limits<int>::max()) {
        try {
            size_t pos;
            int value = std::stoi(input, &pos);
            if (pos != input.length()) return std::nullopt;
            if (value < min || value > max) return std::nullopt;
            return value;
        } catch (const std::exception&) {
            return std::nullopt;
        }
    }
    
    static bool validate_email(const std::string& email) {
        static const std::regex pattern(R"(^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$)");
        return std::regex_match(email, pattern);
    }
    
    static bool validate_length(const std::string& input, size_t min_len, size_t max_len) {
        return input.length() >= min_len && input.length() <= max_len;
    }
    
    static bool validate_alphanumeric(const std::string& input) {
        return std::all_of(input.begin(), input.end(), [](char c) {
            return std::isalnum(static_cast<unsigned char>(c));
        });
    }
    
    static std::string sanitize_html(const std::string& input) {
        std::string result;
        result.reserve(input.size());
        
        for (char c : input) {
            switch (c) {
                case '&': result += "&amp;"; break;
                case '<': result += "&lt;"; break;
                case '>': result += "&gt;"; break;
                case '"': result += "&quot;"; break;
                case '\'': result += "&#x27;"; break;
                default: result += c; break;
            }
        }
        return result;
    }
    
    static bool validate_with_whitelist(const std::string& input,
                                        const std::vector<std::string>& allowed) {
        return std::find(allowed.begin(), allowed.end(), input) != allowed.end();
    }
};

void safe_process_input(const std::string& input) {
    auto value = InputValidator::parse_int(input, 0, 100);
    if (!value) {
        std::cerr << "Invalid input: must be integer between 0 and 100" << std::endl;
        return;
    }
    std::cout << "Value: " << *value << std::endl;
}
```

---

## 四、完整检查清单

### 代码审查检查清单

#### 字符串操作

- [ ] 使用std::string替代char[]
- [ ] 使用std::string_view进行只读字符串传递
- [ ] 使用std::format或fmt库进行格式化
- [ ] 避免使用c_str()后的指针长期持有

#### 容器与数组

- [ ] 使用std::vector/std::array替代C数组
- [ ] 使用at()进行边界检查访问
- [ ] 使用迭代器进行安全遍历

#### 资源管理

- [ ] 使用RAII模式管理所有资源
- [ ] 使用std::unique_ptr/std::shared_ptr
- [ ] 无裸指针delete/new

#### 数据库操作

- [ ] 使用参数化查询/预编译语句
- [ ] 禁止字符串拼接SQL
- [ ] 使用ORM框架

#### 命令执行

- [ ] 禁止system()执行用户输入
- [ ] 使用execvp等带参数形式
- [ ] 使用白名单验证命令参数

#### 文件路径

- [ ] 使用std::filesystem处理路径
- [ ] 检查路径遍历攻击
- [ ] 验证路径在允许目录内

#### 整数运算

- [ ] 关键计算检查溢出
- [ ] 使用安全数学运算库
- [ ] 使用无符号类型处理大小/索引

#### 输入验证

- [ ] 验证输入类型
- [ ] 验证输入长度
- [ ] 验证输入范围
- [ ] 验证输入格式（正则）
- [ ] 使用白名单优于黑名单

---

## 五、安全编码最佳实践

### 1. 编译器安全选项

```cmake
# CMakeLists.txt
add_compile_options(
    -Wall -Wextra -Werror
    -fstack-protector-strong
    -D_FORTIFY_SOURCE=2
    -fPIE
)
add_link_options(
    -Wl,-z,relro,-z,now
    -pie
)
```

### 2. 使用静态分析工具

```bash
cppcheck --enable=all --std=c++20 source.cpp
clang-tidy source.cpp -- -std=c++20
```

### 3. 启用地址消毒器

```bash
g++ -fsanitize=address -fsanitize=undefined -g source.cpp -o program
```

### 4. 输入验证原则

1. **拒绝默认**：默认拒绝所有输入，只接受明确的合法输入
2. **最小权限**：只授予完成功能所需的最小权限
3. **深度防御**：多层验证，不要依赖单一防护
4. **失败安全**：验证失败时进入安全状态

---

## 六、参考资料

- CWE-120: Buffer Copy without Checking Size
- CWE-134: Use of Externally-Controlled Format String
- CWE-78: OS Command Injection
- CWE-22: Path Traversal
- CWE-190: Integer Overflow
- SEI CERT C++ Coding Standard
- C++ Core Guidelines
- ISO/IEC 17961:2013 C Secure Coding Standard
