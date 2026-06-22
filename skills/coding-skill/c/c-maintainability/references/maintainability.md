# C语言可维护性规范详细参考

## 1. 模块化设计

### 1.1 文件组织原则

每个模块由一对 `.c/.h` 文件组成，头文件定义接口，源文件实现细节。

**目录结构示例：**
```
project/
├── include/
│   ├── module_a.h
│   └── module_b.h
├── src/
│   ├── module_a.c
│   └── module_b.c
└── main.c
```

**头文件示例（module_a.h）：**
```c
#ifndef MODULE_A_H
#define MODULE_A_H

#include <stdint.h>

typedef struct module_a module_a_t;

module_a_t* module_a_create(int init_value);
void module_a_destroy(module_a_t* handle);
int module_a_process(module_a_t* handle, int input);

#endif
```

**源文件示例（module_a.c）：**
```c
#include "module_a.h"
#include <stdlib.h>

struct module_a {
    int value;
    int count;
};

module_a_t* module_a_create(int init_value) {
    module_a_t* handle = malloc(sizeof(module_a_t));
    if (handle) {
        handle->value = init_value;
        handle->count = 0;
    }
    return handle;
}

void module_a_destroy(module_a_t* handle) {
    if (handle) {
        free(handle);
    }
}

int module_a_process(module_a_t* handle, int input) {
    if (!handle) return -1;
    handle->count++;
    return handle->value + input;
}
```

### 1.2 命名空间规范

使用模块前缀避免命名冲突：
- 函数：`模块名_动词名词`（如 `uart_send_data`）
- 类型：`模块名_t`（如 `uart_handle_t`）
- 宏：`模块名_宏名`（如 `UART_BUFFER_SIZE`）
- 静态变量：`s_模块名_变量名`

## 2. 接口与实现分离

### 2.1 不透明指针（Opaque Pointer）

头文件仅声明类型别名，隐藏实现细节：

```c
/* logger.h - 公共接口 */
#ifndef LOGGER_H
#define LOGGER_H

typedef struct logger logger_t;

typedef enum {
    LOG_LEVEL_DEBUG = 0,
    LOG_LEVEL_INFO,
    LOG_LEVEL_ERROR
} log_level_t;

logger_t* logger_create(const char* filename);
void logger_destroy(logger_t* handle);
void logger_write(logger_t* handle, log_level_t level, const char* msg);

#endif
```

```c
/* logger.c - 私有实现 */
#include "logger.h"
#include <stdio.h>
#include <string.h>

#define MAX_PATH_LEN 256

struct logger {
    FILE* fp;
    char filepath[MAX_PATH_LEN];
    log_level_t min_level;
};

logger_t* logger_create(const char* filename) {
    logger_t* handle = malloc(sizeof(logger_t));
    if (handle && filename) {
        handle->fp = fopen(filename, "a");
        strncpy(handle->filepath, filename, MAX_PATH_LEN - 1);
        handle->min_level = LOG_LEVEL_INFO;
    }
    return handle;
}
```

**优点：**
- 二进制兼容性：修改结构体不影响调用方
- 编译隔离：修改实现只需重编一个文件
- 信息隐藏：用户无法直接访问内部成员

## 3. 头文件最佳实践

### 3.1 只暴露必要接口

```c
/* 糟糕示例：暴露了内部函数 */
#ifndef DATABASE_H
#define DATABASE_H

void db_connect(void);
void db_disconnect(void);
void db_query(const char* sql);
void db_internal_parse(const char* sql);  /* 内部函数不应暴露 */
int db_validate_connection(void);          /* 内部函数不应暴露 */

#endif

/* 良好示例：只暴露公共接口 */
#ifndef DATABASE_H
#define DATABASE_H

void db_connect(void);
void db_disconnect(void);
void db_query(const char* sql);

#endif
```

### 3.2 最小化头文件依赖

```c
/* 糟糕示例：头文件包含过多 */
#ifndef APP_H
#define APP_H

#include "logger.h"
#include "database.h"
#include "network.h"
#include "config.h"
#include <stdio.h>
#include <stdlib.h>

void app_init(void);

#endif

/* 良好示例：使用前向声明 */
#ifndef APP_H
#define APP_H

/* 仅声明指针时使用前向声明 */
struct logger;
struct database;

void app_init(struct logger* log, struct database* db);

#endif
```

### 3.3 头文件包含顺序

```c
/* 1. 对应头文件 */
#include "module.h"

/* 2. 项目内头文件 */
#include "config.h"
#include "logger.h"

/* 3. 第三方库头文件 */
#include <sqlite3.h>

/* 4. 标准库头文件 */
#include <stdio.h>
#include <stdlib.h>
```

## 4. 避免全局变量

### 4.1 全局变量的危害

```c
/* 糟糕示例：全局状态 */
static int g_counter = 0;
static char g_buffer[1024];

void increment(void) {
    g_counter++;
}

void process(const char* data) {
    strcpy(g_buffer, data);
    /* 多线程不安全，难以测试 */
}
```

### 4.2 使用上下文结构体

```c
/* 良好示例：上下文封装 */
typedef struct {
    int counter;
    char buffer[1024];
    int error_code;
} processor_t;

processor_t* processor_create(void) {
    processor_t* ctx = malloc(sizeof(processor_t));
    if (ctx) {
        ctx->counter = 0;
        ctx->error_code = 0;
    }
    return ctx;
}

void processor_increment(processor_t* ctx) {
    if (ctx) ctx->counter++;
}

void processor_destroy(processor_t* ctx) {
    free(ctx);
}
```

## 5. 使用 static 限制作用域

### 5.1 静态函数

限制函数作用域在当前文件：

```c
/* 内部辅助函数用static */
static int parse_header(const char* data) {
    return data[0] << 8 | data[1];
}

static bool validate_checksum(const uint8_t* data, size_t len) {
    uint8_t sum = 0;
    for (size_t i = 0; i < len; i++) {
        sum += data[i];
    }
    return sum == 0;
}

/* 公共接口 */
int packet_process(const uint8_t* data, size_t len) {
    if (!validate_checksum(data, len)) {
        return -1;
    }
    return parse_header((const char*)data);
}
```

### 5.2 静态全局变量

文件内使用的全局变量用static：

```c
/* 单文件内的缓存 */
static char s_error_msg[256] = {0};

void set_error(const char* msg) {
    strncpy(s_error_msg, msg, sizeof(s_error_msg) - 1);
}

const char* get_error(void) {
    return s_error_msg;
}
```

## 6. 依赖注入模式

### 6.1 函数指针注入

```c
/* 定义接口 */
typedef struct {
    int (*read)(void* ctx, uint8_t* buf, size_t len);
    int (*write)(void* ctx, const uint8_t* buf, size_t len);
    void* ctx;
} io_interface_t;

/* 使用注入的接口 */
int data_processor_process(io_interface_t* io, const uint8_t* data, size_t len) {
    return io->write(io->ctx, data, len);
}

/* 具体实现 */
int file_read(void* ctx, uint8_t* buf, size_t len) {
    FILE* fp = (FILE*)ctx;
    return fread(buf, 1, len, fp);
}

int file_write(void* ctx, const uint8_t* buf, size_t len) {
    FILE* fp = (FILE*)ctx;
    return fwrite(buf, 1, len, fp);
}

/* 使用示例 */
FILE* fp = fopen("data.bin", "rb");
io_interface_t io = {
    .read = file_read,
    .write = file_write,
    .ctx = fp
};
```

### 6.2 配置结构体注入

```c
typedef struct {
    const char* host;
    int port;
    int timeout_ms;
    int retry_count;
} network_config_t;

typedef struct {
    network_config_t config;
    int socket;
} network_client_t;

network_client_t* network_client_create(const network_config_t* config) {
    network_client_t* client = malloc(sizeof(network_client_t));
    if (client && config) {
        memcpy(&client->config, config, sizeof(network_config_t));
        client->socket = -1;
    }
    return client;
}
```

## 7. 配置与代码分离

### 7.1 配置文件方式

```c
/* config.ini */
[server]
host=192.168.1.100
port=8080
timeout=5000

/* config.h */
typedef struct {
    char host[64];
    int port;
    int timeout_ms;
} app_config_t;

int config_load(const char* filename, app_config_t* config);
```

### 7.2 编译时配置

```c
/* config.h */
#ifndef CONFIG_H
#define CONFIG_H

/* 可通过编译选项覆盖 */
#ifndef BUFFER_SIZE
#define BUFFER_SIZE 1024
#endif

#ifndef MAX_CONNECTIONS
#define MAX_CONNECTIONS 100
#endif

#ifndef DEBUG_MODE
#define DEBUG_MODE 0
#endif

#endif
```

```c
/* 编译时指定 */
/* gcc -DBUFFER_SIZE=4096 -DMAX_CONNECTIONS=200 app.c */
```

## 8. 避免重复代码（DRY原则）

### 8.1 提取公共函数

```c
/* 糟糕示例：重复代码 */
void process_int_array(int* arr, int len) {
    for (int i = 0; i < len; i++) {
        arr[i] = arr[i] * 2;
    }
}

void process_float_array(float* arr, int len) {
    for (int i = 0; i < len; i++) {
        arr[i] = arr[i] * 2;
    }
}

/* 良好示例：使用函数指针和void* */
typedef void (*transform_fn)(void* element);

void transform_array(void* arr, size_t len, size_t elem_size, transform_fn fn) {
    uint8_t* ptr = (uint8_t*)arr;
    for (size_t i = 0; i < len; i++) {
        fn(ptr + i * elem_size);
    }
}
```

### 8.2 宏和模板

```c
/* 使用宏生成类型安全的容器 */
#define DEFINE_VECTOR(type, name) \
    typedef struct { \
        type* data; \
        size_t size; \
        size_t capacity; \
    } name##_vector_t; \
    \
    name##_vector_t* name##_vector_create(size_t cap) { \
        name##_vector_t* v = malloc(sizeof(name##_vector_t)); \
        if (v) { \
            v->data = malloc(sizeof(type) * cap); \
            v->capacity = v->data ? cap : 0; \
            v->size = 0; \
        } \
        return v; \
    }

DEFINE_VECTOR(int, int)
DEFINE_VECTOR(float, float)
```

## 检查清单

### 模块化设计
- [ ] 每个模块有独立的 .c/.h 文件对
- [ ] 函数和类型使用统一的前缀命名
- [ ] 头文件使用 include guard
- [ ] 头文件只包含必要的接口声明

### 接口设计
- [ ] 使用不透明指针隐藏实现细节
- [ ] 公共接口有清晰的文档注释
- [ ] 内部函数使用 static 限制作用域
- [ ] 避免暴露内部结构体定义

### 依赖管理
- [ ] 使用前向声明减少头文件包含
- [ ] 没有循环依赖
- [ ] 头文件包含顺序正确
- [ ] 通过依赖注入降低耦合

### 代码质量
- [ ] 避免全局变量，使用上下文结构体
- [ ] 配置数据与代码逻辑分离
- [ ] 重复代码已提取为公共函数
- [ ] 每个函数职责单一

### 可测试性
- [ ] 外部依赖可通过接口注入
- [ ] 函数可独立测试
- [ ] 无硬编码的外部依赖
- [ ] 状态可重置
