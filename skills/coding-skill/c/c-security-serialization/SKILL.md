---
name: c-security-serialization
description: C语言编码规范 - 序列化安全规范。适用于所有C语言代码开发场景：(1) 数据校验；(2) 格式安全；(3) 敏感数据保护。触发关键词：C语言、序列化、反序列化、JSON、安全、数据校验
---

# C语言序列化安全规范

基于安全编程规范

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 反序列化前必须校验数据 |
| 规则1.2 | 序列化数据必须包含版本信息 |
| 规则1.3 | 禁止序列化未加密的敏感数据 |
| 规则1.4 | 检查数据长度防止溢出 |

## 数据校验

```c
int parse_user(const char *json_str, user_t *user)
{
    cJSON *root = cJSON_Parse(json_str);
    if (root == NULL) {
        return ERROR_PARSE;
    }

    cJSON *name = cJSON_GetObjectItem(root, "name");
    if (!cJSON_IsString(name)) {
        cJSON_Delete(root);
        return ERROR_INVALID_FIELD;
    }

    if (strlen(name->valuestring) > MAX_NAME_LENGTH) {
        cJSON_Delete(root);
        return ERROR_FIELD_TOO_LONG;
    }

    strncpy(user->name, name->valuestring, MAX_NAME_LENGTH - 1);
    user->name[MAX_NAME_LENGTH - 1] = '\0';

    cJSON_Delete(root);
    return SUCCESS;
}
```

## 详细规范

见 [references/security-serialization.md](references/security-serialization.md)
