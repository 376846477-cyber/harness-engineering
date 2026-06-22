---
name: c-maintainability
description: C语言编码规范 - 可维护性规范。适用于所有C语言代码开发场景：(1) 代码设计时考虑可维护性；(2) 代码审查时检查可维护性；(3) 重构代码提升可维护性。触发关键词：C语言、可维护、maintainability、模块化、接口、依赖、耦合、复用
---

# C语言可维护性规范

基于Clean Code指导书

## 核心原则

可维护是指软件可修改、可扩展和可复用的能力。修改/扩展代码要影响局部化，应对的方法是**抽象**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 使用头文件定义接口 |
| 1.2 | 隐藏实现细节（不透明指针） |
| 1.3 | 模块化设计 |
| 2.1 | 接口最小化 |
| 3.1 | 优先组合而非继承（通过包含结构体） |
| 4.1 | 提取公共代码（DRY） |

## 不透明指针（Opaque Pointer）

```c
/* user_service.h - 公开接口 */
typedef struct user_service_s user_service_t;

int user_svc_create(user_service_t **service, const config_t *config);
void user_svc_destroy(user_service_t *service);
int user_svc_find_by_id(const user_service_t *service, int64_t id, user_t *user);

/* user_service.c - 私有实现 */
struct user_service_s {
    repository_t *repo;
    cache_t *cache;
    pthread_mutex_t mutex;
};
```

## 函数指针实现多态

```c
typedef struct {
    int (*open)(void *self, const char *path);
    int (*read)(void *self, char *buffer, size_t size);
    int (*close)(void *self);
} io_interface_t;
```

## 详细规范

见 [references/maintainability.md](references/maintainability.md)

## 检查清单

- [ ] 是否通过头文件接口隔离了变化？
- [ ] 是否使用不透明指针隐藏实现？
- [ ] 是否有重复代码需要提取？
- [ ] 模块间依赖是否最小化？
