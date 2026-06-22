---
name: c-portability
description: C语言编码规范 - 可移植性规范。适用于所有C语言代码开发场景：(1) 代码编写时考虑可移植性；(2) 代码审查时检查可移植性；(3) 跨平台实现。触发关键词：C语言、可移植、portability、平台、跨平台、编译器、POSIX
---

# C语言可移植性规范

基于Clean Code指导书

## 核心原则

C语言代码应尽量使用标准库和POSIX接口，避免引入平台相关的依赖。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 使用标准库替代平台API |
| 1.2 | 明确字符编码 |
| 1.3 | 注意数据类型大小 |
| 1.4 | 使用条件编译处理平台差异 |
| 2.1 | 使用跨平台构建系统 |
| 3.1 | 字节序处理 |

## 数据类型

```c
#include <stdint.h>

/* 使用固定宽度类型 */
int32_t count;          /* 明确32位 */
int64_t id;             /* 明确64位 */
size_t length;          /* 对象大小 */
uint8_t buffer[1024];   /* 明确8位 */

/* 避免使用平台相关类型 */
/* int, long 大小不确定 */
```

## 条件编译

```c
#ifdef _WIN32
    #include <windows.h>
    #define PATH_SEPARATOR "\\"
    #define snprintf _snprintf
#else
    #include <unistd.h>
    #include <pthread.h>
    #define PATH_SEPARATOR "/"
#endif
```

## 字节序

```c
#include <arpa/inet.h>

/* 网络字节序转换 */
uint32_t host_order = 0x12345678;
uint32_t net_order = htonl(host_order);
uint32_t back = ntohl(net_order);
```

## 详细规范

见 [references/portability.md](references/portability.md)

## 检查清单

- [ ] 是否使用标准库函数？
- [ ] 数据类型大小是否明确？
- [ ] 是否使用条件编译处理差异？
- [ ] 字节序是否处理？
- [ ] 构建系统是否跨平台？
