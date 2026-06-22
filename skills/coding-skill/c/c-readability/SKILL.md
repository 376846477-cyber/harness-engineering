---
name: c-readability
description: C语言编码规范 - 可读性规范。适用于所有C语言代码开发场景：(1) 代码编写时提升可读性；(2) 代码审查时检查可读性；(3) 重构代码提升可读性。触发关键词：C语言、可读、readability、易读、理解、阅读
---

# C语言可读性规范

基于Clean Code指导书

## 核心原则

提升可读性就是要：**传递足够多的有用信息、减少信息失真、减少信息干扰、降低信息理解难度**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 利用名称传递信息 |
| 1.2 | 利用格式传递信息 |
| 1.3 | 注释作为代码的补充 |
| 2.1 | 无废弃代码 |
| 3.1 | 功能单一 |
| 3.2 | 简化语句逻辑，使用卫语句 |
| 3.3 | 变量作用域尽量小 |

## 函数单一职责

```c
/* 错误：一个函数做太多事 */
int process_user(int id, char *name, int *result);

/* 正确：每个函数单一职责 */
int user_find_by_id(int id, user_t *user);
int user_update_name(int id, const char *name);
int user_calculate_score(const user_t *user, int *score);
```

## 减少嵌套

```c
/* 卫语句 */
int user_svc_validate(const user_t *user)
{
    if (user == NULL) {
        return ERROR_INVALID_PARAM;
    }
    if (user->id <= 0) {
        return ERROR_INVALID_ID;
    }
    if (user->name[0] == '\0') {
        return ERROR_INVALID_NAME;
    }
    return SUCCESS;
}
```

## 详细规范

见 [references/readability.md](references/readability.md)

## 检查清单

- [ ] 名称是否传递了足够的信息？
- [ ] 代码格式是否一致？
- [ ] 是否有废弃代码？
- [ ] 函数是否功能单一？
- [ ] 嵌套层级是否过深？
