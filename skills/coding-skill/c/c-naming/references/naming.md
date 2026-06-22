# C语言命名规范详细参考

## 1. 源文件编码格式

- **编码**：UTF-8（无BOM）
- **换行符**：LF（Unix风格）
- **文件末尾**：保留一个空行
- **最大行宽**：建议不超过120字符

## 2. 标识符通用规则

### 基本约束
- **长度限制**：2-64字符（循环变量i/j/k除外）
- **允许字符**：字母、数字、下划线（必须字母或下划线开头）
- **禁止使用**：保留关键字、标准库函数名、双下划线开头、单下划线+大写字母开头

### 可读性原则
- 名称应自描述，避免缩写（常见缩写除外）
- 常见缩写示例：`buf`（buffer）、`ptr`（pointer）、`cnt`（count）、`cfg`（config）、`msg`（message）

## 3. 文件命名规则

| 规则 | 说明 | 示例 |
|------|------|------|
| 格式 | 全小写+下划线 | `user_service.c` |
| 成对 | .c/.h必须成对出现 | `user_service.c` + `user_service.h` |
| 前缀 | 使用模块名作为前缀 | `network_tcp_client.c` |
| 禁止 | 禁止空格、中文、特殊字符 | ❌ `User Manager.c` |

**目录结构示例**：
```
src/
├── user/
│   ├── user_service.c
│   ├── user_service.h
│   ├── user_dao.c
│   └── user_dao.h
└── network/
    ├── network_tcp.c
    └── network_tcp.h
```

## 4. 函数命名规则

### 基本格式
```
<模块前缀>_<动词>_<名词/形容词>
```

### 动词前缀约定表

| 前缀 | 用途 | 示例 |
|------|------|------|
| `create_` / `init_` | 创建/初始化 | `user_create()`, `buffer_init()` |
| `destroy_` / `deinit_` | 销毁/反初始化 | `user_destroy()`, `buffer_deinit()` |
| `get_` | 获取属性 | `user_get_name()` |
| `set_` | 设置属性 | `user_set_name()` |
| `is_` | 布尔查询（是） | `user_is_valid()` |
| `has_` | 布尔查询（有） | `user_has_permission()` |
| `can_` | 布尔查询（能） | `buffer_can_write()` |
| `should_` | 布尔查询（应该） | `task_should_retry()` |
| `add_` / `remove_` | 添加/移除 | `list_add_node()` |
| `find_` / `search_` | 查找 | `user_find_by_id()` |
| `parse_` / `format_` | 解析/格式化 | `json_parse_string()` |
| `encode_` / `decode_` | 编码/解码 | `base64_encode()` |
| `start_` / `stop_` | 启动/停止 | `server_start()` |
| `open_` / `close_` | 打开/关闭 | `file_open()` |
| `read_` / `write_` | 读/写 | `file_read_line()` |
| `connect_` / `disconnect_` | 连接/断开 | `tcp_connect()` |
| `enable_` / `disable_` | 使能/禁用 | `timer_enable()` |
| `calculate_` / `compute_` | 计算 | `math_calculate_distance()` |
| `convert_` / `transform_` | 转换 | `string_convert_to_int()` |
| `validate_` / `check_` | 验证/检查 | `input_validate_email()` |
| `handle_` / `process_` | 处理 | `event_handle_click()` |
| `dump_` / `print_` | 调试输出 | `config_dump()` |

### 函数命名示例
```c
/* 创建/销毁 */
user_t* user_create(const char* name);
void user_destroy(user_t* user);

/* 属性访问 */
const char* user_get_name(const user_t* user);
void user_set_name(user_t* user, const char* name);
int user_get_age(const user_t* user);

/* 布尔查询 */
bool user_is_valid(const user_t* user);
bool user_has_permission(const user_t* user, perm_t perm);
bool user_can_delete(const user_t* user);

/* 操作 */
void user_add_friend(user_t* user, const user_t* friend);
void user_remove_friend(user_t* user, user_id_t friend_id);
user_t* user_find_by_id(user_id_t id);
```

## 5. 变量命名规则

### 局部变量
- **格式**：`snake_case`
- **示例**：`user_name`, `buffer_size`, `retry_count`
- **循环变量例外**：`i`, `j`, `k` 可用于简单循环

### 全局变量
- **格式**：`g_<snake_case>`
- **示例**：`g_server_config`, `g_connection_pool`, `g_error_message`

### 静态变量
- **文件作用域静态**：`s_<snake_case>`
- **示例**：`s_instance_count`, `s_cached_value`

### 成员变量（结构体内部）
- **格式**：`snake_case`（不加前缀）
- **示例**：`struct user_s { char* name; int age; }`

### 变量命名示例
```c
/* 局部变量 */
int buffer_size = 1024;
char* file_path = "/etc/config";

/* 全局变量 */
config_t g_app_config;
logger_t g_logger;

/* 静态变量 */
static int s_connection_count = 0;
static cache_t s_response_cache;

/* 循环变量例外 */
for (int i = 0; i < count; i++) {
    for (int j = 0; j < width; j++) {
        /* ... */
    }
}

/* 有意义的循环变量 */
for (int row = 0; row < height; row++) {
    for (int col = 0; col < width; col++) {
        /* ... */
    }
}
```

## 6. 宏和常量命名规则

### 宏常量
- **格式**：`UPPER_SNAKE_CASE`
- **模块前缀**：`MODULE_UPPER_NAME`
- **示例**：`MAX_BUFFER_SIZE`, `USER_MAX_NAME_LENGTH`, `NETWORK_TIMEOUT_MS`

### 宏函数
- **格式**：`UPPER_SNAKE_CASE`
- **示例**：`MIN(a, b)`, `MAX(a, b)`, `ARRAY_SIZE(arr)`

### 枚举常量
- **格式**：`UPPER_SNAKE_CASE` 或 `<枚举名>_<值>`
- **示例**：`COLOR_RED`, `STATUS_OK`, `ERROR_INVALID_PARAM`

### 宏定义示例
```c
/* 常量宏 */
#define MAX_BUFFER_SIZE         4096
#define DEFAULT_TIMEOUT_MS      5000
#define USER_MAX_NAME_LENGTH    64

/* 宏函数 */
#define MIN(a, b)               ((a) < (b) ? (a) : (b))
#define MAX(a, b)               ((a) > (b) ? (a) : (b))
#define ARRAY_SIZE(arr)         (sizeof(arr) / sizeof((arr)[0]))
#define STRINGIFY(x)            #x
#define CONCAT(a, b)            a##b

/* 条件编译宏 */
#ifdef DEBUG
#define LOG_DEBUG(fmt, ...)     printf("[DEBUG] " fmt "\n", ##__VA_ARGS__)
#else
#define LOG_DEBUG(fmt, ...)     ((void)0)
#endif
```

## 7. 类型定义命名规则

### typedef 类型
- **格式**：`<name>_t`（POSIX风格）或 `PascalCase`
- **示例**：`user_t`, `buffer_t`, `Config`, `Logger`

### 结构体类型
- **格式**：`<name>_s` 或 `PascalCase`
- **示例**：`struct user_s`, `struct config_s`

### 联合体类型
- **格式**：`<name>_u` 或 `PascalCase`
- **示例**：`union data_u`, `union packet_data_u`

### 枚举类型
- **格式**：`<name>_e` 或 `PascalCase`
- **示例**：`enum status_e`, `enum color_e`

### 类型定义示例
```c
/* 方式一：POSIX风格（推荐） */
typedef struct user_s {
    char* name;
    int age;
} user_t;

typedef enum status_e {
    STATUS_OK = 0,
    STATUS_ERROR = -1,
    STATUS_INVALID_PARAM = -2
} status_t;

/* 方式二：PascalCase */
typedef struct User {
    char* name;
    int age;
} User;

typedef enum Status {
    Status_Ok = 0,
    Status_Error = -1
} Status;

/* 不透明类型（前向声明） */
typedef struct user_impl_s user_t;  /* 实现在.c文件中 */
```

## 8. 结构体命名规则

### 命名格式
- **结构体标签**：`struct <name>_s`
- **typedef别名**：`<name>_t` 或 `PascalCase`

### 结构体定义示例
```c
/* 完整定义 */
typedef struct user_config_s {
    char* server_host;
    int server_port;
    int timeout_ms;
    bool enable_ssl;
} user_config_t;

/* 嵌套结构体 */
typedef struct network_address_s {
    char ip[16];
    int port;
} network_address_t;

typedef struct network_endpoint_s {
    network_address_t local_addr;
    network_address_t remote_addr;
    int socket_fd;
} network_endpoint_t;

/* 不透明结构体（隐藏实现） */
/* header.h */
typedef struct user_impl_s user_t;
user_t* user_create(void);
void user_destroy(user_t* user);

/* impl.c */
struct user_impl_s {
    char* name;
    int age;
    void* private_data;
};
```

## 9. 枚举命名规则

### 枚举类型命名
- **格式**：`<name>_e` 或 `PascalCase`

### 枚举值命名
- **格式**：`UPPER_SNAKE_CASE` 或 `<枚举名>_<值>`

### 枚举定义示例
```c
/* 方式一：统一前缀（推荐） */
typedef enum status_e {
    STATUS_OK = 0,
    STATUS_ERROR = -1,
    STATUS_INVALID_PARAM = -2,
    STATUS_OUT_OF_MEMORY = -3,
    STATUS_TIMEOUT = -4
} status_t;

/* 方式二：枚举名作为前缀 */
typedef enum status_e {
    Status_Ok = 0,
    Status_Error = -1,
    Status_InvalidParam = -2
} Status;

/* 标志位枚举 */
typedef enum file_mode_e {
    FILE_MODE_READ      = 1 << 0,   /* 0x01 */
    FILE_MODE_WRITE     = 1 << 1,   /* 0x02 */
    FILE_MODE_APPEND    = 1 << 2,   /* 0x04 */
    FILE_MODE_BINARY    = 1 << 3    /* 0x08 */
} file_mode_t;

/* 使用标志位 */
int mode = FILE_MODE_READ | FILE_MODE_WRITE;
```

## 10. 头文件保护规则

### 方式一：#ifndef（传统方式）
```c
#ifndef MODULE_PATH_FILENAME_H_
#define MODULE_PATH_FILENAME_H_

/* 头文件内容 */

#endif /* MODULE_PATH_FILENAME_H_ */
```

### 方式二：#pragma once（现代方式）
```c
#pragma once

/* 头文件内容 */
```

### 头文件保护示例
```c
/* user_service.h */
#ifndef USER_SERVICE_USER_SERVICE_H_
#define USER_SERVICE_USER_SERVICE_H_

#include "user/user_dao.h"

typedef struct user_service_s user_service_t;

user_service_t* user_service_create(user_dao_t* dao);
void user_service_destroy(user_service_t* service);
status_t user_service_get_user(user_service_t* service, user_id_t id, user_t** user);

#endif /* USER_SERVICE_USER_SERVICE_H_ */
```

## 11. 命名风格对照表

| 类型 | 命名风格 | 前缀/后缀 | 示例 |
|------|----------|-----------|------|
| 文件名 | `lower_snake_case` | 模块前缀 | `user_service.c` |
| 函数名 | `lower_snake_case` | 模块前缀 | `user_create()` |
| 局部变量 | `lower_snake_case` | 无 | `user_name` |
| 全局变量 | `lower_snake_case` | `g_` | `g_app_config` |
| 静态变量 | `lower_snake_case` | `s_` | `s_instance_count` |
| 宏常量 | `UPPER_SNAKE_CASE` | 模块前缀 | `MAX_BUFFER_SIZE` |
| 宏函数 | `UPPER_SNAKE_CASE` | 无 | `MIN(a, b)` |
| typedef类型 | `lower_snake_case` | `_t` | `user_t` |
| 结构体标签 | `lower_snake_case` | `_s` | `struct user_s` |
| 联合体标签 | `lower_snake_case` | `_u` | `union data_u` |
| 枚举标签 | `lower_snake_case` | `_e` | `enum status_e` |
| 枚举值 | `UPPER_SNAKE_CASE` | 类型前缀 | `STATUS_OK` |
| 头文件保护 | `UPPER_SNAKE_CASE_H_` | 路径前缀 | `USER_SERVICE_H_` |

## 12. 常见错误与正确示例对比

### 文件命名

| 错误示例 | 正确示例 | 说明 |
|----------|----------|------|
| `UserManager.c` | `user_manager.c` | 应全小写 |
| `user-manager.c` | `user_manager.c` | 用下划线非连字符 |
| `userManager.c` | `user_manager.c` | 不用驼峰命名 |
| `usr_mgr.c` | `user_manager.c` | 避免过度缩写 |

### 函数命名

| 错误示例 | 正确示例 | 说明 |
|----------|----------|------|
| `createUser()` | `user_create()` | 使用snake_case和模块前缀 |
| `CreateUser()` | `user_create()` | 不用PascalCase |
| `userCreate()` | `user_create()` | 不用camelCase |
| `getUser()` | `user_get_by_id()` | 模块前缀+动词+名词 |
| `checkValid()` | `user_is_valid()` | 布尔返回用is_前缀 |
| `init()` | `user_init()` | 缺少模块前缀 |

### 变量命名

| 错误示例 | 正确示例 | 说明 |
|----------|----------|------|
| `userName` | `user_name` | 使用snake_case |
| `UserName` | `user_name` | 不用PascalCase |
| `gUserName` | `g_user_name` | 全局变量用g_前缀 |
| `sUserName` | `s_user_name` | 静态变量用s_前缀 |
| `n` | `user_count` | 避免无意义变量名 |
| `buff` | `buffer` | 拼写完整单词 |
| `temp` | `temp_file_path` | 避免泛指名称 |

### 宏和常量命名

| 错误示例 | 正确示例 | 说明 |
|----------|----------|------|
| `maxSize` | `MAX_SIZE` | 宏用大写 |
| `MaxSize` | `MAX_SIZE` | 宏用大写 |
| `max_size` | `MAX_SIZE` | 宏用大写 |
| `max` | `MAX_VALUE` | 避免太短 |
| `size` | `BUFFER_SIZE` | 缺少前缀/上下文 |

### 类型定义命名

| 错误示例 | 正确示例 | 说明 |
|----------|----------|------|
| `typedef int INT;` | `typedef int user_id_t;` | 避免无意义类型 |
| `typedef struct user User;` | `typedef struct user_s user_t;` | 使用_t后缀 |
| `enum status { ... };` | `typedef enum status_e { ... } status_t;` | 提供typedef别名 |

### 枚举值命名

| 错误示例 | 正确示例 | 说明 |
|----------|----------|------|
| `ok, error` | `STATUS_OK, STATUS_ERROR` | 需要类型前缀 |
| `Ok, Error` | `STATUS_OK, STATUS_ERROR` | 用大写 |
| `status_ok` | `STATUS_OK` | 用大写 |

## 13. 完整检查清单

### 文件检查
- [ ] 文件名使用 `lower_snake_case`
- [ ] 文件名使用模块前缀
- [ ] .c/.h 文件成对出现
- [ ] 头文件有保护宏

### 函数检查
- [ ] 函数名使用 `snake_case`
- [ ] 函数名包含模块前缀
- [ ] 函数名以动词开头
- [ ] 布尔返回函数使用 `is_`/`has_`/`can_` 前缀
- [ ] 函数名长度 2-64 字符

### 变量检查
- [ ] 局部变量使用 `snake_case`
- [ ] 全局变量使用 `g_` 前缀
- [ ] 静态变量使用 `s_` 前缀
- [ ] 变量名长度 2-64 字符（循环变量除外）
- [ ] 变量名自描述，避免过度缩写
- [ ] 避免使用标准库函数名

### 宏和常量检查
- [ ] 宏常量使用 `UPPER_SNAKE_CASE`
- [ ] 宏函数使用 `UPPER_SNAKE_CASE`
- [ ] 宏有模块前缀（避免冲突）
- [ ] 宏参数使用括号保护

### 类型定义检查
- [ ] typedef 使用 `_t` 后缀
- [ ] 结构体标签使用 `_s` 后缀
- [ ] 联合体标签使用 `_u` 后缀
- [ ] 枚举标签使用 `_e` 后缀
- [ ] 枚举值使用 `UPPER_SNAKE_CASE`

### 其他检查
- [ ] 无双下划线开头
- [ ] 无单下划线+大写字母开头
- [ ] 无保留关键字冲突
- [ ] 无标准库名称冲突
- [ ] 头文件保护宏唯一且符合规范

### 命名冲突检查
- [ ] 函数使用模块前缀避免冲突
- [ ] 宏使用模块前缀避免冲突
- [ ] 全局变量使用 `g_` 前缀避免冲突
- [ ] 枚举值使用类型前缀避免冲突
- [ ] 头文件保护使用完整路径避免冲突
