---
name: c-performance
description: C语言编码规范 - 高效性规范。适用于所有C语言代码开发场景：(1) 代码编写时考虑性能；(2) 代码审查时检查性能；(3) 算法、内存管理优化。触发关键词：C语言、高效、performance、性能、优化、算法、内存、缓存
---

# C语言高效性规范

基于Clean Code指导书

## 核心原则

在不影响其他代码质量属性的前提下，养成编写高效代码的习惯，**不要过早做性能优化**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 选择高效的数据结构 |
| 1.2 | 减少冗余或无效计算 |
| 1.3 | 合理使用缓存 |
| 2.1 | 减少内存分配 |
| 3.1 | 减少IO次数和系统调用 |
| 4.1 | 利用编译器优化 |
| 6.1 | 使用SIMD向量化加速数值计算 |

## 数据结构选择

```c
/* O(1)查找 - 哈希表 */
#include "uthash.h"
typedef struct {
    int key;
    char value[64];
    UT_hash_handle hh;
} hash_entry_t;

/* O(log n)查找 - 二分查找 */
int binary_search(const int *arr, size_t len, int target);

/* O(1)栈操作 */
typedef struct {
    int *data;
    size_t top;
    size_t capacity;
} stack_t;
```

## 内存池

```c
typedef struct {
    char *buffer;
    size_t offset;
    size_t capacity;
} memory_pool_t;

void *pool_alloc(memory_pool_t *pool, size_t size)
{
    if (pool->offset + size > pool->capacity) {
        return NULL;
    }
    void *ptr = pool->buffer + pool->offset;
    pool->offset += size;
    return ptr;
}
```

## SIMD向量化

```c
#include <immintrin.h>

// AVX向量加法：一次处理8个float
void avx_add(float* a, const float* b, const float* c, size_t n) {
    for (size_t i = 0; i + 8 <= n; i += 8) {
        __m256 vb = _mm256_loadu_ps(&b[i]);
        __m256 vc = _mm256_loadu_ps(&c[i]);
        __m256 va = _mm256_add_ps(vb, vc);
        _mm256_storeu_ps(&a[i], va);
    }
}

// 编译器自动向量化
#pragma GCC ivdep  // 忽略向量依赖
for (int i = 0; i < n; i++) {
    a[i] = b[i] + c[i];  // 编译器自动生成SIMD指令
}
```

## 详细规范

见 [references/performance.md](references/performance.md)

## 检查清单

- [ ] 是否使用了合适的数据结构？
- [ ] 是否有重复计算可以缓存？
- [ ] 是否有不必要的内存分配？
- [ ] 是否减少了系统调用？
- [ ] 是否利用了编译器优化？
- [ ] 数值计算是否可以利用SIMD向量化？
