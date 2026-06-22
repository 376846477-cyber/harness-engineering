# C语言错误处理规范详细参考

## 一、错误处理策略概览

### 1.1 核心原则

| 原则 | 说明 |
|-----|------|
| 显式检查 | 所有可能失败的调用必须检查返回值 |
| 及时处理 | 错误发生时立即处理，不可忽略 |
| 资源安全 | 确保任何错误路径都释放已分配资源 |
| 信息完整 | 错误信息包含位置、原因、上下文 |
| 可追溯性 | 日志记录错误发生点和传播路径 |

### 1.2 错误处理模式对比

| 模式 | 适用场景 | 优点 | 缺点 |
|-----|---------|-----|------|
| 返回值 | 常规函数调用 | 简单直观 | 只能返回单一值 |
| 错误码参数 | 需要返回结果值 | 分离结果与错误 | 调用稍复杂 |
| errno | 系统调用 | 标准化 | 线程安全性需注意 |
| 全局错误变量 | 复杂错误上下文 | 携带丰富信息 | 非线程安全 |

### 1.3 错误分类

```c
typedef enum {
    ERR_NONE = 0,
    
    ERR_INVALID_PARAM = -1,
    ERR_NULL_POINTER = -2,
    ERR_OUT_OF_RANGE = -3,
    
    ERR_NO_MEMORY = -10,
    ERR_BUFFER_OVERFLOW = -11,
    ERR_BUFFER_UNDERFLOW = -12,
    
    ERR_FILE_NOT_FOUND = -20,
    ERR_FILE_PERMISSION = -21,
    ERR_FILE_IO = -22,
    
    ERR_NETWORK_TIMEOUT = -30,
    ERR_NETWORK_CONNECTION = -31,
    
    ERR_SYSTEM = -100,
    ERR_UNKNOWN = -999
} error_code_t;
```

## 二、goto统一清理模式

### 2.1 标准模式

```c
int process_file(const char *path, data_t **result) {
    int ret = ERR_NONE;
    FILE *fp = NULL;
    char *buffer = NULL;
    data_t *data = NULL;
    size_t size = 0;
    
    if (!path || !result) {
        return ERR_NULL_POINTER;
    }
    
    fp = fopen(path, "r");
    if (!fp) {
        ret = ERR_FILE_NOT_FOUND;
        goto cleanup;
    }
    
    if (fseek(fp, 0, SEEK_END) != 0) {
        ret = ERR_FILE_IO;
        goto cleanup;
    }
    size = ftell(fp);
    rewind(fp);
    
    buffer = malloc(size + 1);
    if (!buffer) {
        ret = ERR_NO_MEMORY;
        goto cleanup;
    }
    
    if (fread(buffer, 1, size, fp) != size) {
        ret = ERR_FILE_IO;
        goto cleanup;
    }
    buffer[size] = '\0';
    
    data = parse_data(buffer);
    if (!data) {
        ret = ERR_INVALID_PARAM;
        goto cleanup;
    }
    
    *result = data;
    data = NULL;
    
cleanup:
    if (data) free_data(data);
    if (buffer) free(buffer);
    if (fp) fclose(fp);
    return ret;
}
```

### 2.2 多级清理

```c
int complex_operation(context_t *ctx) {
    int ret = ERR_NONE;
    resource_a_t *ra = NULL;
    resource_b_t *rb = NULL;
    resource_c_t *rc = NULL;
    
    ra = create_resource_a(ctx);
    if (!ra) { ret = ERR_NO_MEMORY; goto cleanup_a; }
    
    rb = create_resource_b(ra);
    if (!rb) { ret = ERR_NO_MEMORY; goto cleanup_b; }
    
    rc = create_resource_c(rb);
    if (!rc) { ret = ERR_NO_MEMORY; goto cleanup_c; }
    
    ret = do_work(rc);
    
cleanup_c:
    if (rc) destroy_resource_c(rc);
cleanup_b:
    if (rb) destroy_resource_b(rb);
cleanup_a:
    if (ra) destroy_resource_a(ra);
    return ret;
}
```

### 2.3 goto使用规范

| 规范 | 说明 |
|-----|------|
| 仅用于清理 | goto只能跳转到函数末尾的清理代码 |
| 向下跳转 | 只允许向前（代码顺序向后）跳转 |
| 标签命名 | 使用cleanup、done、error等语义化名称 |
| 指针初始化 | 所有资源指针声明时初始化为NULL |
| 逆序释放 | 清理顺序与分配顺序相反 |
| NULL检查 | 释放前检查指针是否非NULL |

## 三、错误码定义规范

### 3.1 枚举定义

```c
typedef enum {
    SUCCESS = 0,
    
    ERR_INVALID_ARGUMENT = 1,
    ERR_NULL_PARAM = 2,
    ERR_OUT_OF_BOUNDS = 3,
    
    ERR_ALLOC_FAILED = 100,
    ERR_REALLOC_FAILED = 101,
    
    ERR_FILE_OPEN = 200,
    ERR_FILE_READ = 201,
    ERR_FILE_WRITE = 202,
    
    ERR_SOCKET_CREATE = 300,
    ERR_SOCKET_CONNECT = 301,
    ERR_SOCKET_TIMEOUT = 302,
} error_t;
```

### 3.2 错误描述函数

```c
const char *error_strerror(error_t err) {
    static const char *messages[] = {
        [SUCCESS] = "Success",
        [ERR_INVALID_ARGUMENT] = "Invalid argument",
        [ERR_NULL_PARAM] = "Null parameter",
        [ERR_OUT_OF_BOUNDS] = "Index out of bounds",
        [ERR_ALLOC_FAILED] = "Memory allocation failed",
        [ERR_FILE_OPEN] = "Failed to open file",
    };
    
    if (err >= 0 && err < sizeof(messages)/sizeof(messages[0])) {
        return messages[err] ? messages[err] : "Unknown error";
    }
    return "Invalid error code";
}
```

### 3.3 错误码设计原则

| 原则 | 说明 |
|-----|------|
| 0表示成功 | 遵循POSIX惯例，if (!ret)表示成功 |
| 分组编号 | 按模块/类型分组，预留扩展空间 |
| 单一含义 | 每个错误码只表示一种错误 |
| 避免重复 | 检查是否已有相同含义的错误码 |
| 文档化 | 每个错误码必须有注释说明 |

## 四、errno使用规范

### 4.1 使用模式

```c
errno = 0;
int result = some_syscall();
if (result == -1) {
    int saved_errno = errno;
    log_error("syscall failed: %s", strerror(saved_errno));
    return map_errno_to_error(saved_errno);
}
```

### 4.2 errno规范

| 规范 | 说明 |
|-----|------|
| 先清零 | 调用前设置errno = 0 |
| 立即保存 | 失败后立即保存errno值 |
| 结合返回值 | 必须先检查返回值确定失败 |
| 线程安全 | errno是线程局部变量 |
| 勿用于非系统调用 | 自定义函数不应使用errno |

### 4.3 errno映射

```c
error_t map_errno_to_error(int e) {
    switch (e) {
        case 0: return SUCCESS;
        case ENOMEM: return ERR_ALLOC_FAILED;
        case ENOENT: return ERR_FILE_NOT_FOUND;
        case EACCES: return ERR_PERMISSION_DENIED;
        case EINVAL: return ERR_INVALID_ARGUMENT;
        default: return ERR_SYSTEM;
    }
}
```

## 五、NULL指针检查

### 5.1 函数入口检查

```c
int process_data(const data_t *data, size_t len) {
    if (data == NULL) {
        log_error("process_data: NULL data pointer");
        return ERR_NULL_PARAM;
    }
    if (len == 0) {
        log_error("process_data: zero length");
        return ERR_INVALID_ARGUMENT;
    }
    return do_process(data, len);
}
```

### 5.2 返回值检查

```c
buffer_t *buf = create_buffer(1024);
if (buf == NULL) {
    log_error("Failed to create buffer");
    return ERR_ALLOC_FAILED;
}

char *str = malloc(size);
if (str == NULL) {
    log_error("malloc failed for size %zu", size);
    return ERR_ALLOC_FAILED;
}
```

### 5.3 检查规范

| 场景 | 规范 |
|-----|------|
| 函数入口 | 检查所有指针参数 |
| 动态分配 | malloc/calloc/realloc后立即检查 |
| 返回指针 | 调用返回指针的函数后检查 |
| 结构体成员 | 解引用前检查指针成员 |

## 六、错误日志记录

### 6.1 日志级别

```c
typedef enum {
    LOG_DEBUG,
    LOG_INFO,
    LOG_WARN,
    LOG_ERROR,
    LOG_FATAL
} log_level_t;
```

### 6.2 日志宏定义

```c
#define LOG_ERROR(fmt, ...) \
    log_write(LOG_ERROR, __FILE__, __LINE__, __func__, fmt, ##__VA_ARGS__)

#define LOG_WARN(fmt, ...) \
    log_write(LOG_WARN, __FILE__, __LINE__, __func__, fmt, ##__VA_ARGS__)

#define LOG_INFO(fmt, ...) \
    log_write(LOG_INFO, __FILE__, __LINE__, __func__, fmt, ##__VA_ARGS__)
```

### 6.3 日志规范

| 规范 | 说明 |
|-----|------|
| 位置信息 | 包含文件名、行号、函数名 |
| 错误上下文 | 记录参数值、状态等上下文 |
| 错误码 | 记录具体错误码 |
| 英文消息 | 避免编码问题 |
| 避免敏感信息 | 不记录密码、密钥等 |

## 七、断言使用

### 7.1 assert使用场景

```c
#include <assert.h>

int divide(int a, int b) {
    assert(b != 0);
    return a / b;
}

void process_array(int *arr, size_t len) {
    assert(arr != NULL);
    assert(len > 0);
    for (size_t i = 0; i < len; i++) {
        arr[i] *= 2;
    }
}
```

### 7.2 断言规范

| 规范 | 说明 |
|-----|------|
| 检查不变量 | 验证程序逻辑假设 |
| 仅用于开发 | NDEBUG定义时被移除 |
| 无副作用 | 断言表达式不应有副作用 |
| 快速失败 | 发现编程错误立即终止 |
| 生产代码 | 关键检查不应使用assert |

### 7.3 运行时检查

```c
#define CHECK_NULL(ptr, ret) \
    do { \
        if ((ptr) == NULL) { \
            LOG_ERROR("NULL pointer: %s", #ptr); \
            return (ret); \
        } \
    } while (0)

#define CHECK_ALLOC(ptr) \
    do { \
        if ((ptr) == NULL) { \
            LOG_ERROR("Allocation failed: %s", #ptr); \
            return ERR_ALLOC_FAILED; \
        } \
    } while (0)
```

## 八、防御式编程

### 8.1 边界检查

```c
int array_get(const array_t *arr, size_t index, int *value) {
    if (arr == NULL || value == NULL) {
        return ERR_NULL_PARAM;
    }
    if (index >= arr->size) {
        LOG_ERROR("Index %zu out of bounds (size: %zu)", index, arr->size);
        return ERR_OUT_OF_BOUNDS;
    }
    *value = arr->data[index];
    return SUCCESS;
}
```

### 8.2 输入验证

```c
int set_config(const char *name, const char *value) {
    if (name == NULL || value == NULL) {
        return ERR_NULL_PARAM;
    }
    if (strlen(name) > MAX_NAME_LEN) {
        return ERR_INVALID_ARGUMENT;
    }
    if (!is_valid_name(name)) {
        return ERR_INVALID_ARGUMENT;
    }
    return update_config(name, value);
}
```

### 8.3 资源限制

```c
#define MAX_ALLOC_SIZE (1024 * 1024 * 100)

void *safe_malloc(size_t size) {
    if (size == 0 || size > MAX_ALLOC_SIZE) {
        LOG_ERROR("Invalid allocation size: %zu", size);
        return NULL;
    }
    void *ptr = malloc(size);
    if (ptr == NULL) {
        LOG_ERROR("malloc failed for size %zu", size);
    }
    return ptr;
}
```

## 九、完整检查清单

### 9.1 编码阶段

- [ ] 所有函数入口检查NULL参数
- [ ] malloc/calloc/realloc返回值检查
- [ ] fopen/fread/fwrite等文件操作返回值检查
- [ ] 系统调用返回值检查
- [ ] 字符串操作长度检查
- [ ] 数组索引边界检查
- [ ] 除法运算除数检查
- [ ] 错误路径使用goto统一清理

### 9.2 资源管理

- [ ] 所有资源指针初始化为NULL
- [ ] 清理标签中释放所有资源
- [ ] 释放前检查指针非NULL
- [ ] 释放后将指针置NULL
- [ ] 避免重复释放
- [ ] 避免内存泄漏

### 9.3 错误传播

- [ ] 错误码定义清晰明确
- [ ] 错误信息包含上下文
- [ ] 日志记录错误位置
- [ ] 保留errno原始值
- [ ] 正确映射错误码

### 9.4 断言使用

- [ ] 不变量使用assert检查
- [ ] 断言无副作用
- [ ] 生产环境关键检查用运行时检查替代
- [ ] 前置条件使用assert
- [ ] 后置条件考虑使用assert

### 9.5 日志规范

- [ ] 错误使用ERROR级别
- [ ] 警告使用WARN级别
- [ ] 包含文件名和行号
- [ ] 包含错误码
- [ ] 不记录敏感信息

### 9.6 代码审查

- [ ] 所有返回值都已处理
- [ ] 无忽略的错误
- [ ] 资源在所有路径正确释放
- [ ] 错误信息有足够上下文
- [ ] 错误处理逻辑正确
- [ ] 无未初始化变量使用
