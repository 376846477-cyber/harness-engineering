# C语言输入校验与安全规范详细参考

## 一、安全红线规则

### 绝对禁止

1. **禁止使用危险函数**：gets、sprintf、strcpy、strcat、vsprintf、scanf("%s")、strtok
2. **禁止用户输入作为格式化字符串**：printf(user_input)、syslog(LOG_INFO, user_input)
3. **禁止直接拼接命令执行**：system(cmd)、popen(cmd, "r")
4. **禁止未验证的路径操作**：直接使用用户提供的文件路径
5. **禁止忽略返回值**：malloc、fopen、scanf等返回值必须检查
6. **禁止整数溢出后继续操作**：加法、乘法前必须检查溢出

### 必须遵守

1. 所有缓冲区操作必须指定长度限制
2. 所有指针解引用前必须检查NULL
3. 所有动态内存必须匹配释放（防止内存泄漏）
4. 所有外部输入必须验证（类型、长度、范围、格式）
5. 所有错误路径必须正确清理资源

---

## 二、危险函数→安全函数对照表

### 字符串处理函数

| 危险函数 | 风险 | 安全替代 | 说明 |
|---------|------|---------|------|
| `gets(buf)` | 缓冲区溢出 | `fgets(buf, size, stdin)` | 必须指定大小 |
| `sprintf(buf, fmt, ...)` | 缓冲区溢出 | `snprintf(buf, size, fmt, ...)` | 返回值检查 |
| `strcpy(dst, src)` | 缓冲区溢出 | `strncpy(dst, src, size)` 或 `strlcpy` | 需手动添加'\0' |
| `strcat(dst, src)` | 缓冲区溢出 | `strncat(dst, src, size)` 或 `strlcat` | 注意剩余空间 |
| `vsprintf(buf, fmt, ap)` | 缓冲区溢出 | `vsnprintf(buf, size, fmt, ap)` | 可变参数安全 |
| `strtok(str, delim)` | 非线程安全 | `strtok_r(str, delim, &saveptr)` | POSIX安全版本 |

### 格式化函数

| 危险函数 | 风险 | 安全替代 | 说明 |
|---------|------|---------|------|
| `printf(user_input)` | 格式化字符串漏洞 | `printf("%s", user_input)` | 固定格式串 |
| `fprintf(fp, user_input)` | 格式化字符串漏洞 | `fprintf(fp, "%s", user_input)` | 固定格式串 |
| `syslog(pri, user_input)` | 格式化字符串漏洞 | `syslog(pri, "%s", user_input)` | 固定格式串 |

### 输入函数

| 危险函数 | 风险 | 安全替代 | 说明 |
|---------|------|---------|------|
| `scanf("%s", buf)` | 缓冲区溢出 | `scanf("%255s", buf)` | 指定宽度 |
| `scanf("%[^\n]", buf)` | 缓冲区溢出 | `fgets()` + `sscanf()` | 更安全可控 |
| `getchar()` 循环 | 无边界检查 | `fgets()` 或带计数循环 | 必须限制长度 |

---

## 三、详细代码示例

### 3.1 缓冲区溢出防护

#### 错误示例

```c
void unsafe_copy(const char *input) {
    char buffer[256];
    strcpy(buffer, input);  // 危险：无长度检查
    printf("Result: %s\n", buffer);
}

void unsafe_format(const char *name, int age) {
    char buffer[100];
    sprintf(buffer, "Name: %s, Age: %d", name, age);  // 危险：可能溢出
    puts(buffer);
}
```

#### 正确示例

```c
void safe_copy(const char *input) {
    char buffer[256];
    size_t input_len = strlen(input);
    
    if (input_len >= sizeof(buffer)) {
        fprintf(stderr, "Input too long\n");
        return;
    }
    
    strncpy(buffer, input, sizeof(buffer) - 1);
    buffer[sizeof(buffer) - 1] = '\0';  // 确保终止
    printf("Result: %s\n", buffer);
}

void safe_format(const char *name, int age) {
    char buffer[100];
    int written = snprintf(buffer, sizeof(buffer), 
                           "Name: %s, Age: %d", name, age);
    
    if (written < 0 || written >= (int)sizeof(buffer)) {
        fprintf(stderr, "Output truncated or error\n");
        return;
    }
    puts(buffer);
}
```

### 3.2 格式化字符串漏洞防护

#### 错误示例

```c
void unsafe_log(const char *user_message) {
    printf(user_message);  // 危险：用户输入作为格式化字符串
}

void unsafe_syslog(int priority, const char *msg) {
    syslog(priority, msg);  // 危险：msg可能包含%格式符
}
```

#### 正确示例

```c
void safe_log(const char *user_message) {
    printf("%s", user_message);  // 正确：固定格式字符串
}

void safe_syslog(int priority, const char *msg) {
    syslog(priority, "%s", msg);  // 正确：固定格式字符串
}
```

### 3.3 命令注入防护

#### 错误示例

```c
void unsafe_ping(const char *host) {
    char cmd[256];
    sprintf(cmd, "ping -c 4 %s", host);  // 危险：拼接命令
    system(cmd);  // 危险：可能执行恶意命令
}
```

#### 正确示例

```c
#include <unistd.h>
#include <sys/wait.h>

int safe_ping(const char *host) {
    if (!is_valid_hostname(host)) {
        fprintf(stderr, "Invalid hostname\n");
        return -1;
    }
    
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork failed");
        return -1;
    }
    
    if (pid == 0) {
        char *args[] = {"ping", "-c", "4", (char *)host, NULL};
        execvp("ping", args);
        _exit(127);
    }
    
    int status;
    waitpid(pid, &status, 0);
    return WEXITSTATUS(status);
}

int is_valid_hostname(const char *host) {
    if (!host || !*host) return 0;
    size_t len = strlen(host);
    if (len > 253) return 0;
    
    for (size_t i = 0; i < len; i++) {
        char c = host[i];
        if (!isalnum(c) && c != '-' && c != '.') {
            return 0;
        }
    }
    return 1;
}
```

### 3.4 文件路径安全

#### 错误示例

```c
void unsafe_read_file(const char *filename) {
    char path[512];
    sprintf(path, "/var/data/%s", filename);  // 危险：路径遍历
    FILE *fp = fopen(path, "r");  // 可能打开任意文件
    if (fp) {
        // 读取文件...
        fclose(fp);
    }
}
```

#### 正确示例

```c
#include <limits.h>
#include <stdlib.h>
#include <string.h>
#include <libgen.h>

int is_path_safe(const char *path) {
    if (!path) return 0;
    if (strstr(path, "..") != NULL) return 0;
    if (path[0] == '/') return 0;
    if (strstr(path, "/.") != NULL) return 0;
    return 1;
}

void safe_read_file(const char *filename) {
    if (!is_path_safe(filename)) {
        fprintf(stderr, "Invalid filename\n");
        return;
    }
    
    char base_dir[] = "/var/data";
    char path[PATH_MAX];
    
    int written = snprintf(path, sizeof(path), "%s/%s", base_dir, filename);
    if (written < 0 || written >= PATH_MAX) {
        fprintf(stderr, "Path too long\n");
        return;
    }
    
    char resolved[PATH_MAX];
    if (realpath(path, resolved) == NULL) {
        perror("realpath failed");
        return;
    }
    
    if (strncmp(resolved, base_dir, strlen(base_dir)) != 0) {
        fprintf(stderr, "Path escape detected\n");
        return;
    }
    
    FILE *fp = fopen(resolved, "r");
    if (fp) {
        // 读取文件...
        fclose(fp);
    }
}
```

### 3.5 整数溢出检测

#### 错误示例

```c
void unsafe_allocate(size_t count, size_t item_size) {
    size_t total = count * item_size;  // 危险：可能溢出
    void *buf = malloc(total);
    // ...
}
```

#### 正确示例

```c
#include <stddef.h>
#include <limits.h>
#include <stdint.h>

int safe_multiply(size_t a, size_t b, size_t *result) {
    if (a == 0 || b == 0) {
        *result = 0;
        return 1;
    }
    if (a > SIZE_MAX / b) {
        return 0;  // 溢出
    }
    *result = a * b;
    return 1;
}

void safe_allocate(size_t count, size_t item_size) {
    size_t total;
    
    if (!safe_multiply(count, item_size, &total)) {
        fprintf(stderr, "Integer overflow\n");
        return;
    }
    
    void *buf = malloc(total);
    if (buf == NULL) {
        fprintf(stderr, "Memory allocation failed\n");
        return;
    }
    // ...
    free(buf);
}
```

### 3.6 空指针解引用防护

#### 错误示例

```c
void unsafe_strlen(const char *str) {
    int len = strlen(str);  // 危险：str可能是NULL
    printf("Length: %d\n", len);
}
```

#### 正确示例

```c
void safe_strlen(const char *str) {
    if (str == NULL) {
        fprintf(stderr, "NULL pointer\n");
        return;
    }
    size_t len = strlen(str);
    printf("Length: %zu\n", len);
}

void safe_strdup(const char *str) {
    if (str == NULL) {
        fprintf(stderr, "NULL pointer\n");
        return;
    }
    
    char *copy = strdup(str);
    if (copy == NULL) {
        perror("strdup failed");
        return;
    }
    // 使用copy...
    free(copy);
}
```

### 3.7 Use-After-Free防护

#### 错误示例

```c
void unsafe_uaf(void) {
    char *buf = malloc(100);
    free(buf);
    strcpy(buf, "test");  // 危险：use-after-free
}
```

#### 正确示例

```c
void safe_uaf(void) {
    char *buf = malloc(100);
    if (buf == NULL) return;
    
    strncpy(buf, "test", 99);
    buf[99] = '\0';
    // 使用buf...
    
    free(buf);
    buf = NULL;  // 防止悬挂指针
}
```

### 3.8 SQL注入防护（使用SQLite示例）

#### 错误示例

```c
void unsafe_query(sqlite3 *db, const char *username) {
    char sql[512];
    sprintf(sql, "SELECT * FROM users WHERE name='%s'", username);  // 危险
    sqlite3_exec(db, sql, NULL, NULL, NULL);
}
```

#### 正确示例

```c
void safe_query(sqlite3 *db, const char *username) {
    sqlite3_stmt *stmt;
    const char *sql = "SELECT * FROM users WHERE name = ?";
    
    int rc = sqlite3_prepare_v2(db, sql, -1, &stmt, NULL);
    if (rc != SQLITE_OK) {
        fprintf(stderr, "Prepare failed: %s\n", sqlite3_errmsg(db));
        return;
    }
    
    rc = sqlite3_bind_text(stmt, 1, username, -1, SQLITE_TRANSIENT);
    if (rc != SQLITE_OK) {
        fprintf(stderr, "Bind failed\n");
        sqlite3_finalize(stmt);
        return;
    }
    
    while ((rc = sqlite3_step(stmt)) == SQLITE_ROW) {
        // 处理结果...
    }
    
    sqlite3_finalize(stmt);
}
```

---

## 四、完整检查清单

### 代码审查检查清单

#### 缓冲区操作

- [ ] 所有字符串拷贝使用strncpy/strlcpy并指定长度
- [ ] 所有字符串拼接使用strncat/strlcat并检查剩余空间
- [ ] 所有格式化输出使用snprintf/vsnprintf
- [ ] 所有数组访问都有边界检查
- [ ] 缓冲区末尾正确添加'\0'终止符

#### 格式化字符串

- [ ] printf/fprintf/sprintf格式串是字面量常量
- [ ] syslog使用固定格式串"%s"
- [ ] 没有将用户输入作为格式化字符串

#### 命令执行

- [ ] 不使用system()执行用户输入
- [ ] 使用execvp/execve等带参数数组形式
- [ ] 命令参数使用白名单验证

#### 文件操作

- [ ] 文件路径进行规范化处理
- [ ] 检查路径遍历攻击（../等）
- [ ] 使用realpath验证最终路径
- [ ] 检查路径是否在允许目录内

#### 内存管理

- [ ] malloc/calloc/realloc返回值检查NULL
- [ ] free后指针置NULL
- [ ] 没有重复释放（double-free）
- [ ] 没有内存泄漏
- [ ] 整数运算前检查溢出

#### 指针操作

- [ ] 所有指针解引用前检查NULL
- [ ] 没有使用未初始化的指针
- [ ] 没有野指针和悬挂指针

#### 输入验证

- [ ] 所有外部输入都进行验证
- [ ] 验证输入类型（数字、字符串等）
- [ ] 验证输入长度（最小/最大）
- [ ] 验证输入范围（数值范围）
- [ ] 验证输入格式（正则、模式）
- [ ] 使用白名单优于黑名单

---

## 五、安全编码最佳实践

### 1. 使用静态分析工具

```bash
# 使用静态分析工具检查代码
cppcheck --enable=all source.c
clang --analyze source.c
```

### 2. 编译器安全选项

```makefile
# Makefile中启用安全编译选项
CFLAGS += -Wall -Wextra -Werror
CFLAGS += -fstack-protector-strong
CFLAGS += -D_FORTIFY_SOURCE=2
CFLAGS += -fPIE -pie
LDFLAGS += -Wl,-z,relro,-z,now
```

### 3. 运行时保护

- 启用ASLR（地址空间布局随机化）
- 启用Stack Canaries（栈保护）
- 使用不可执行内存（NX/DEP）

### 4. 输入验证原则

1. **拒绝默认**：默认拒绝所有输入，只接受明确的合法输入
2. **最小权限**：只授予完成功能所需的最小权限
3. **深度防御**：多层验证，不要依赖单一防护
4. **失败安全**：验证失败时进入安全状态

---

## 六、常见漏洞CVE示例

### CVE-2012-2122: MySQL认证绕过

```c
// 错误示例：memcmp返回值处理不当
if (memcmp(hash, expected, 32) == 0) {
    // 认证成功
}

// 正确做法：使用常量时间比较
if (CRYPTO_memcmp(hash, expected, 32) == 0) {
    // 认证成功
}
```

### 格式化字符串漏洞示例

```c
// 漏洞代码
void log_message(const char *msg) {
    printf(msg);  // 如果msg包含%n，可写内存
}

// 修复后
void log_message(const char *msg) {
    printf("%s", msg);
}
```

---

## 七、参考资料

- CWE-120: Buffer Copy without Checking Size of Input
- CWE-134: Use of Externally-Controlled Format String
- CWE-78: Improper Neutralization of Special Elements used in an OS Command
- CWE-22: Improper Limitation of a Pathname to a Restricted Directory
- CWE-190: Integer Overflow or Wraparound
- CWE-416: Use After Free
- CWE-476: NULL Pointer Dereference
- SEI CERT C Coding Standard
- ISO/IEC 17961:2013 C Secure Coding Standard
