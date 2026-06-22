---
name: c-formatting
description: C语言编码规范 - 排版格式规范。适用于所有C语言代码开发场景：(1) 编写C代码时的格式检查；(2) 代码审查时检查缩进、空行；(3) 格式化代码；(4) include排序。触发关键词：C语言、格式、缩进、空行、include、格式化、format、indent
---

# C语言排版格式规范

基于MISRA C和GNU Coding Standards

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 缩进4空格，不使用制表符 |
| 规则1.2 | 行宽不超过120字符 |
| 规则1.3 | include按标准库/系统/本项目排序 |
| 规则1.4 | 函数间2空行 |
| 规则1.5 | 大括号换行（Allman风格） |

## include排序

```c
/* 对应头文件 */
#include "user_service.h"

/* C标准库 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* POSIX/系统头文件 */
#include <pthread.h>
#include <unistd.h>

/* 本项目其他头文件 */
#include "common/error_codes.h"
#include "storage/repository.h"
```

## 缩进示例

```c
int user_svc_create(user_service_t **service, const config_t *config)
{
    if (service == NULL || config == NULL) {
        return ERROR_INVALID_PARAM;
    }

    user_service_t *svc = (user_service_t *)malloc(sizeof(user_service_t));
    if (svc == NULL) {
        return ERROR_NO_MEMORY;
    }

    int ret = init_internal(svc, config);
    if (ret != SUCCESS) {
        free(svc);
        return ret;
    }

    *service = svc;
    return SUCCESS;
}
```

## 详细规范

见 [references/formatting.md](references/formatting.md)
