---
name: c-testability
description: C语言编码规范 - 可测试性规范。适用于所有C语言代码开发场景：(1) 代码编写时考虑可测试性；(2) 代码审查时检查可测试性；(3) 单元测试框架。触发关键词：C语言、可测试、testability、Unity、CMock、mock、stub、隔离、单元测试
---

# C语言可测试性规范

基于Clean Code指导书

## 核心原则

可测试性好的代码要满足：**可隔离、可控制、可观测、可定位**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 高内聚低耦合 |
| 2.1 | 减少全局状态 |
| 2.2 | 使用函数指针实现依赖注入 |
| 3.1 | 结果可观测 |
| 3.2 | 过程可观测 |
| 4.1 | 错误输出适当日志 |

## 函数指针依赖注入

```c
typedef struct {
    int (*read_data)(void *ctx, char *buffer, size_t size);
    int (*write_data)(void *ctx, const char *buffer, size_t size);
    void *ctx;
} io_ops_t;

typedef struct {
    io_ops_t *io;
    /* 其他成员... */
} processor_t;

int processor_run(processor_t *proc)
{
    char buffer[1024];
    int ret = proc->io->read_data(proc->io->ctx, buffer, sizeof(buffer));
    if (ret < 0) {
        return ERROR_IO;
    }
    /* 处理逻辑... */
    return SUCCESS;
}
```

## 单元测试

```c
/* test_user_service.c */
#include "unity.h"
#include "user_service.h"

void setUp(void) { /* 测试前初始化 */ }
void tearDown(void) { /* 测试后清理 */ }

void test_find_by_id_should_return_user_when_exists(void)
{
    user_service_t *svc = NULL;
    user_svc_create(&svc, &test_config);

    user_t user;
    int ret = user_svc_find_by_id(svc, 1, &user);

    TEST_ASSERT_EQUAL(SUCCESS, ret);
    TEST_ASSERT_EQUAL_STRING("Alice", user.name);

    user_svc_destroy(svc);
}

int main(void)
{
    UNITY_BEGIN();
    RUN_TEST(test_find_by_id_should_return_user_when_exists);
    return UNITY_END();
}
```

## 详细规范

见 [references/testability.md](references/testability.md)

## 检查清单

- [ ] 模块是否单一职责？
- [ ] 是否使用函数指针实现依赖注入？
- [ ] 外部依赖是否可替换？
- [ ] 结果是否方便验证？
