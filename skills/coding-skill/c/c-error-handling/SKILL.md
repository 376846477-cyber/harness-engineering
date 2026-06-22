---
name: c-error-handling
description: C语言编码规范 - 错误处理规范。适用于所有C语言代码开发场景：(1) 编写错误处理代码；(2) 代码审查时检查错误处理；(3) 使用goto做统一清理。触发关键词：C语言、错误处理、error、errno、goto、返回值、错误码
---

# C语言错误处理规范

基于SEI CERT C和MISRA C规范

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 检查所有函数返回值 |
| 规则1.2 | 使用goto做统一清理 |
| 规则1.3 | 定义统一的错误码 |
| 规则1.4 | 每个malloc对应free |
| 规则1.5 | 错误信息不含敏感数据 |

## goto清理模式

```c
int user_svc_process(const config_t *config, result_t *result)
{
    int ret = SUCCESS;
    char *buffer = NULL;
    file_handle_t *fh = NULL;

    buffer = (char *)malloc(BUFFER_SIZE);
    if (buffer == NULL) {
        ret = ERROR_NO_MEMORY;
        goto cleanup;
    }

    fh = file_open(config->path);
    if (fh == NULL) {
        ret = ERROR_FILE_OPEN;
        goto cleanup;
    }

    ret = do_process(fh, buffer, result);
    if (ret != SUCCESS) {
        goto cleanup;
    }

cleanup:
    if (fh != NULL) {
        file_close(fh);
    }
    if (buffer != NULL) {
        free(buffer);
    }
    return ret;
}
```

## 错误码定义

```c
typedef enum {
    SUCCESS = 0,
    ERROR_INVALID_PARAM = -1,
    ERROR_NO_MEMORY = -2,
    ERROR_NOT_FOUND = -3,
    ERROR_PERMISSION = -4,
    ERROR_IO = -5,
} error_code_t;
```

## 详细规范

见 [references/error-handling.md](references/error-handling.md)
