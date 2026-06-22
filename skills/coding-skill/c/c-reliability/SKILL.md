---
name: c-reliability
description: C语言编码规范 - 可靠性规范。适用于所有C语言代码开发场景：(1) 代码编写时考虑可靠性；(2) 代码审查时检查可靠性；(3) 防御式编程、容错、资源管理。触发关键词：C语言、可靠、reliability、容错、防御式编程、内存泄露、故障恢复
---

# C语言可靠性规范

基于Clean Code指导书

## 核心原则

系统必须预备**预防错误、容错、故障恢复（自愈）**的能力。C语言没有自动资源管理，必须格外注意资源释放。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 每个malloc对应free |
| 1.2 | 操作防呆设计 |
| 1.3 | 系统过载保护 |
| 2.1 | 防御式编程 |
| 2.2 | 核心流程依赖最小化 |
| 2.3 | 降级处理策略 |

## 资源管理

```c
int process_file(const char *path, result_t *result)
{
    int ret = SUCCESS;
    FILE *fp = NULL;
    char *buffer = NULL;

    fp = fopen(path, "r");
    if (fp == NULL) {
        return ERROR_FILE_OPEN;
    }

    buffer = (char *)malloc(BUFFER_SIZE);
    if (buffer == NULL) {
        ret = ERROR_NO_MEMORY;
        goto cleanup;
    }

    /* 处理逻辑... */

cleanup:
    if (buffer != NULL) {
        free(buffer);
    }
    if (fp != NULL) {
        fclose(fp);
    }
    return ret;
}
```

## 防御式编程

```c
int user_svc_find_by_id(const user_service_t *service, int64_t id, user_t *user)
{
    if (service == NULL || user == NULL) {
        return ERROR_INVALID_PARAM;
    }
    if (id <= 0) {
        return ERROR_INVALID_ID;
    }
    return repo_find(service->repo, id, user);
}
```

## 详细规范

见 [references/reliability.md](references/reliability.md)

## 检查清单

- [ ] 每个malloc是否都有对应free？
- [ ] 每个fopen是否都有对应fclose？
- [ ] 关键操作是否有防呆设计？
- [ ] 是否有降级处理策略？
