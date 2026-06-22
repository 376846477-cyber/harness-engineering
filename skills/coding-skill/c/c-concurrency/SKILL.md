---
name: c-concurrency
description: C语言编码规范 - 并发规范。适用于所有C语言代码开发场景：(1) 编写多线程代码；(2) 共享变量访问控制；(3) 死锁预防。触发关键词：C语言、线程、pthread、mutex、并发、concurrent、锁、lock、信号量
---

# C语言并发规范

基于POSIX线程规范

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 共享变量必须同步访问 |
| 规则1.2 | 使用pthread_mutex保护共享状态 |
| 规则1.3 | 避免死锁：固定加锁顺序 |
| 规则1.4 | 避免竞态条件 |
| 规则1.5 | 使用条件变量进行线程间通知 |

## 线程安全示例

```c
typedef struct {
    int64_t count;
    pthread_mutex_t mutex;
} safe_counter_t;

int safe_counter_init(safe_counter_t *counter)
{
    counter->count = 0;
    return pthread_mutex_init(&counter->mutex, NULL);
}

void safe_counter_increment(safe_counter_t *counter)
{
    pthread_mutex_lock(&counter->mutex);
    counter->count++;
    pthread_mutex_unlock(&counter->mutex);
}

int64_t safe_counter_get(safe_counter_t *counter)
{
    int64_t value;
    pthread_mutex_lock(&counter->mutex);
    value = counter->count;
    pthread_mutex_unlock(&counter->mutex);
    return value;
}

void safe_counter_destroy(safe_counter_t *counter)
{
    pthread_mutex_destroy(&counter->mutex);
}
```

## 详细规范

见 [references/concurrency.md](references/concurrency.md)
