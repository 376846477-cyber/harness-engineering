# C语言可移植性规范详细参考

## 一、标准C库使用

### 1.1 优先使用标准C库

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <stdbool.h>
#include <inttypes.h>

int main(void) {
    printf("标准C库函数具有最佳移植性\n");
    return 0;
}
```

### 1.2 标准库函数对照表

| 功能 | 推荐标准函数 | 避免使用 |
|------|-------------|----------|
| 字符串复制 | strcpy, strncpy | strcpy_s (MSVC特有) |
| 内存分配 | malloc, free | _alloca (Windows特有) |
| 文件操作 | fopen, fclose | _open (POSIX) |
| 时间获取 | time, strftime | GetTickCount (Windows) |

### 1.3 C标准版本选择

```c
#if defined(__STDC_VERSION__)
    #if __STDC_VERSION__ >= 201112L
        #define C_STANDARD "C11"
    #elif __STDC_VERSION__ >= 199901L
        #define C_STANDARD "C99"
    #else
        #define C_STANDARD "C89"
    #endif
#else
    #define C_STANDARD "Unknown"
#endif
```

## 二、避免平台特定API

### 2.1 平台抽象层设计

```c
#if defined(_WIN32) || defined(_WIN64)
    #include <windows.h>
    #define PLATFORM_WINDOWS
#elif defined(__linux__)
    #include <unistd.h>
    #define PLATFORM_LINUX
#elif defined(__APPLE__)
    #include <unistd.h>
    #define PLATFORM_MACOS
#else
    #define PLATFORM_UNKNOWN
#endif

#if defined(PLATFORM_WINDOWS)
    #define SLEEP_MS(ms) Sleep(ms)
#else
    #include <time.h>
    static inline void sleep_ms(unsigned int ms) {
        struct timespec ts = {
            .tv_sec = ms / 1000,
            .tv_nsec = (ms % 1000) * 1000000L
        };
        nanosleep(&ts, NULL);
    }
    #define SLEEP_MS(ms) sleep_ms(ms)
#endif
```

### 2.2 跨平台线程示例

```c
#if defined(PLATFORM_WINDOWS)
    #include <process.h>
    typedef HANDLE thread_t;
    #define thread_create(t, f, a) ((t) = _beginthreadex(NULL, 0, f, a, 0, NULL))
    #define thread_join(t) WaitForSingleObject((t), INFINITE)
    #define thread_close(t) CloseHandle(t)
#else
    #include <pthread.h>
    typedef pthread_t thread_t;
    #define thread_create(t, f, a) pthread_create(&(t), NULL, f, a)
    #define thread_join(t) pthread_join(t, NULL)
#endif
```

## 三、字节序处理

### 3.1 字节序检测

```c
static inline int is_little_endian(void) {
    const uint16_t test = 0x0001;
    return *(const uint8_t*)&test == 0x01;
}

static inline int is_big_endian(void) {
    return !is_little_endian();
}
```

### 3.2 字节序转换

```c
#if defined(PLATFORM_WINDOWS)
    #include <winsock2.h>
    #pragma comment(lib, "ws2_32.lib")
#else
    #include <arpa/inet.h>
#endif

static inline uint16_t swap_uint16(uint16_t val) {
    return (val << 8) | (val >> 8);
}

static inline uint32_t swap_uint32(uint32_t val) {
    return ((val << 24) & 0xFF000000) |
           ((val << 8)  & 0x00FF0000) |
           ((val >> 8)  & 0x0000FF00) |
           ((val >> 24) & 0x000000FF);
}

static inline uint64_t swap_uint64(uint64_t val) {
    return ((val << 56) & 0xFF00000000000000ULL) |
           ((val << 40) & 0x00FF000000000000ULL) |
           ((val << 24) & 0x0000FF0000000000ULL) |
           ((val << 8)  & 0x000000FF00000000ULL) |
           ((val >> 8)  & 0x00000000FF000000ULL) |
           ((val >> 24) & 0x0000000000FF0000ULL) |
           ((val >> 40) & 0x000000000000FF00ULL) |
           ((val >> 56) & 0x00000000000000FFULL);
}
```

### 3.3 网络字节序最佳实践

```c
void send_data(int socket, const void* data, size_t len) {
    uint32_t net_len = htonl((uint32_t)len);
    send(socket, (const char*)&net_len, sizeof(net_len), 0);
    send(socket, (const char*)data, len, 0);
}

size_t receive_data(int socket, void* buffer, size_t max_len) {
    uint32_t net_len;
    recv(socket, (char*)&net_len, sizeof(net_len), 0);
    size_t len = ntohl(net_len);
    if (len > max_len) len = max_len;
    recv(socket, (char*)buffer, len, 0);
    return len;
}
```

## 四、数据类型大小

### 4.1 使用固定宽度类型

```c
#include <stdint.h>

int8_t    i8;    int16_t   i16;   int32_t   i32;   int64_t   i64;
uint8_t   u8;    uint16_t  u16;   uint32_t  u32;   uint64_t  u64;

intptr_t  iptr;
uintptr_t uptr;
size_t    size;
ptrdiff_t diff;
```

### 4.2 类型大小对比

| 类型 | 32位系统 | 64位系统 |
|------|---------|---------|
| int | 4字节 | 4字节 |
| long | 4字节 (Windows) / 8字节 (Linux) | 4字节 (Windows) / 8字节 (Linux) |
| pointer | 4字节 | 8字节 |
| size_t | 4字节 | 8字节 |
| int64_t | 8字节 | 8字节 |

### 4.3 格式化字符串

```c
#include <inttypes.h>

uint64_t value = 12345678901234ULL;
printf("Value: %" PRIu64 "\n", value);

int32_t signed_val = -12345;
printf("Signed: %" PRId32 "\n", signed_val);

uintptr_t ptr_val = (uintptr_t)&value;
printf("Pointer: 0x%" PRIxPTR "\n", ptr_val);
```

## 五、路径分隔符处理

### 5.1 路径分隔符宏

```c
#if defined(PLATFORM_WINDOWS)
    #define PATH_SEPARATOR '\\'
    #define PATH_SEPARATOR_STR "\\"
#else
    #define PATH_SEPARATOR '/'
    #define PATH_SEPARATOR_STR "/"
#endif
```

### 5.2 路径处理函数

```c
#include <string.h>

char* path_normalize(char* path) {
    if (path == NULL) return NULL;
    
    char* src = path;
    char* dst = path;
    
    while (*src) {
        if (*src == '/' || *src == '\\') {
            *dst++ = PATH_SEPARATOR;
            while (*src == '/' || *src == '\\') src++;
        } else {
            *dst++ = *src++;
        }
    }
    *dst = '\0';
    return path;
}

void path_join(char* dest, size_t size, const char* dir, const char* file) {
    snprintf(dest, size, "%s%s%s", dir, PATH_SEPARATOR_STR, file);
}
```

### 5.3 路径示例

```c
char full_path[256];
path_join(full_path, sizeof(full_path), "/home/user", "data.txt");
printf("Full path: %s\n", full_path);
```

## 六、编译器差异处理

### 6.1 编译器检测

```c
#if defined(__GNUC__)
    #define COMPILER_GCC
    #define COMPILER_VERSION (__GNUC__ * 10000 + __GNUC_MINOR__ * 100)
#elif defined(_MSC_VER)
    #define COMPILER_MSVC
    #define COMPILER_VERSION _MSC_VER
#elif defined(__clang__)
    #define COMPILER_CLANG
    #define COMPILER_VERSION (__clang_major__ * 10000 + __clang_minor__ * 100)
#endif
```

### 6.2 编译器特定属性

```c
#if defined(COMPILER_GCC) || defined(COMPILER_CLANG)
    #define ATTRIBUTE_UNUSED __attribute__((unused))
    #define ATTRIBUTE_PRINTF(fmt, args) __attribute__((format(printf, fmt, args)))
    #define ATTRIBUTE_NORETURN __attribute__((noreturn))
    #define LIKELY(x) __builtin_expect(!!(x), 1)
    #define UNLIKELY(x) __builtin_expect(!!(x), 0)
#elif defined(COMPILER_MSVC)
    #define ATTRIBUTE_UNUSED
    #define ATTRIBUTE_PRINTF(fmt, args)
    #define ATTRIBUTE_NORETURN __declspec(noreturn)
    #define LIKELY(x) (x)
    #define UNLIKELY(x) (x)
#endif
```

### 6.3 内联函数差异

```c
#if defined(COMPILER_GCC) || defined(COMPILER_CLANG)
    #define FORCE_INLINE __attribute__((always_inline)) inline
#elif defined(COMPILER_MSVC)
    #define FORCE_INLINE __forceinline
#else
    #define FORCE_INLINE inline
#endif

FORCE_INLINE int fast_abs(int x) {
    return x < 0 ? -x : x;
}
```

## 七、条件编译最佳实践

### 7.1 条件编译范围最小化

```c
#if defined(PLATFORM_WINDOWS)
    #include <windows.h>
#else
    #include <unistd.h>
    #include <sys/time.h>
#endif

double get_time_seconds(void) {
#if defined(PLATFORM_WINDOWS)
    LARGE_INTEGER freq, counter;
    QueryPerformanceFrequency(&freq);
    QueryPerformanceCounter(&counter);
    return (double)counter.QuadPart / (double)freq.QuadPart;
#else
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return (double)tv.tv_sec + (double)tv.tv_usec / 1000000.0;
#endif
}
```

### 7.2 功能特性检测

```c
#if defined(__STDC_VERSION__) && __STDC_VERSION__ >= 199901L
    #define HAS_VLA 1
    #define HAS_INLINE 1
#endif

#if defined(__STDC_VERSION__) && __STDC_VERSION__ >= 201112L
    #define HAS_GENERIC_SELECTION 1
    #define HAS_STATIC_ASSERT 1
#endif
```

## 八、可移植性检查清单

### 8.1 类型与字节序

- [ ] 使用 `<stdint.h>` 的固定宽度类型
- [ ] 使用 `<inttypes.h>` 的格式化宏
- [ ] 网络传输使用 `htonl/ntohl`
- [ ] 文件格式明确定义字节序
- [ ] 避免假设指针大小

### 8.2 平台API

- [ ] 优先使用标准C库函数
- [ ] 平台特定API封装到抽象层
- [ ] 使用条件编译隔离平台差异
- [ ] 提供跨平台替代实现

### 8.3 路径与文件

- [ ] 使用 `PATH_SEPARATOR` 宏
- [ ] 统一路径分隔符处理
- [ ] 注意换行符差异（`\r\n` vs `\n`）
- [ ] 文件权限使用跨平台方式

### 8.4 编译器兼容

- [ ] 检测并适配主流编译器
- [ ] 使用标准语法，避免编译器扩展
- [ ] 警告级别设置一致（`-Wall -Wextra`）
- [ ] 测试多个编译器编译

### 8.5 构建系统

- [ ] 使用CMake等跨平台构建工具
- [ ] 避免硬编码路径和编译器
- [ ] 提供配置选项检测平台特性
- [ ] 文档说明平台依赖
