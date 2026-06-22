---
name: c-naming
description: C语言编码规范 - 命名规范。适用于所有C语言代码开发场景：(1) 编写任何文件、函数时的命名检查；(2) 代码审查时检查命名规范；(3) 重构时确保命名符合规范。触发关键词：C语言、命名、snake_case、常量、变量、函数、宏、naming
---

# C语言命名规范

基于MISRA C和SEI CERT C规范

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 文件名小写+下划线 |
| 规则1.2 | 函数名模块前缀_snake_case |
| 规则1.3 | 变量名snake_case |
| 规则1.4 | 宏UPPER_SNAKE_CASE |
| 规则1.5 | 类型定义使用_typedef后缀或前缀 |
| 规则1.6 | 避免名称冲突 |

## 命名风格对照

| 类型 | 风格 | 示例 |
|-----|------|-----|
| 文件名 | 小写+下划线 | `user_service.c`, `user_service.h` |
| 函数 | 模块_snake_case | `user_svc_create()`, `user_svc_find_by_id()` |
| 变量 | snake_case | `user_name`, `record_count` |
| 宏/常量 | UPPER_SNAKE_CASE | `MAX_BUFFER_SIZE`, `DEFAULT_TIMEOUT` |
| 类型定义 | PascalCase或前缀_t | `UserService`, `user_id_t` |
| 结构体 | PascalCase或前缀_s | `UserService`, `user_node_s` |
| 枚举 | UPPER_SNAKE_CASE或前缀_e | `COLOR_RED`, `status_e` |

## 常见错误

**错误示例**:
```c
int UserID;              /* 变量应snake_case */
void GetUser();          /* 函数应snake_case+模块前缀 */
int maxconnections;      /* 常量应UPPER_SNAKE_CASE */
typedef struct user_service;  /* 类型应有命名规范 */
```

**正确示例**:
```c
int user_id;
void user_svc_get_user(void);
#define MAX_CONNECTIONS 100
typedef struct user_service_s user_service_t;
```

## 详细规范

见 [references/naming.md](references/naming.md)
