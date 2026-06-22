---
name: c-comments
description: C语言编码规范 - 注释规范。适用于所有C语言代码开发场景：(1) 编写函数头注释；(2) 代码审查时检查注释规范；(3) 添加文件注释。触发关键词：C语言、注释、comment、doc、文档、函数头注释
---

# C语言注释规范

基于MISRA C规范

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 公开头文件中的函数使用函数头注释 |
| 规则1.2 | 文件开头使用文件头注释 |
| 规则1.3 | 注释说明"为什么"而非"是什么" |
| 规则1.4 | 保持注释与代码同步 |

## 函数头注释

```c
/**
 * @brief 根据用户ID获取用户信息
 *
 * @param service 用户服务实例
 * @param id      用户唯一标识符
 * @param user    输出参数，存储用户信息
 *
 * @return 成功返回SUCCESS，失败返回错误码
 * @retval ERROR_INVALID_PARAM 参数无效
 * @retval ERROR_NOT_FOUND     用户不存在
 */
int user_svc_find_by_id(const user_service_t *service, int64_t id, user_t *user);
```

## 文件头注释

```c
/**
 * @file user_service.c
 * @brief 用户管理服务实现
 *
 * 提供用户CRUD操作和业务逻辑处理。
 */
```

## 行注释

```c
/* 使用snprintf替代sprintf防止缓冲区溢出 */
snprintf(buffer, sizeof(buffer), "%s", input);
```

## 详细规范

见 [references/comments.md](references/comments.md)
