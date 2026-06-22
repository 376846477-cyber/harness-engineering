# C语言高效性规范详细参考

## 1. CPU优化

### 1.1 算法选择优化

**规则1.1.1：选择时间复杂度最优的算法**

✓ 正确示例：
```c
// 使用哈希表实现O(1)查找
typedef struct {
    int key;
    int value;
} HashEntry;

typedef struct {
    HashEntry* entries;
    size_t size;
    size_t capacity;
} HashTable;

int hash_find(HashTable* table, int key) {
    size_t index = key % table->capacity;
    if (table->entries[index].key == key) {
        return table->entries[index].value;  // O(1)
    }
    return -1;
}
```

✗ 错误示例：
```c
// 线性查找O(n)
int linear_find(int* arr, size_t n, int key) {
    for (size_t i = 0; i < n; i++) {
        if (arr[i] == key) {
            return i;  // O(n)
        }
    }
    return -1;
}
```

**规则1.1.2：根据数据规模选择合适算法**

✓ 正确示例：
```c
// 小数据用简单算法，大数据用高效算法
void sort_adaptive(int* arr, size_t n) {
    if (n < 32) {
        insertion_sort(arr, n);  // 小数据：简单算法
    } else {
        quicksort(arr, n);       // 大数据：高效算法
    }
}
```

### 1.2 空间换时间

**规则1.2.1：使用查找表替代重复计算**

✓ 正确示例：
```c
// 预计算查找表
static const int popcount_table[256] = {
    0, 1, 1, 2, 1, 2, 2, 3, 1, 2, 2, 3, 2, 3, 3, 4,
    // ... 256个值
};

int count_bits(uint8_t byte) {
    return popcount_table[byte];  // O(1)查表
}
```

✗ 错误示例：
```c
// 每次都重新计算
int count_bits_slow(uint8_t byte) {
    int count = 0;
    while (byte) {
        count += byte & 1;
        byte >>= 1;
    }
    return count;  // O(8)
}
```

**规则1.2.2：缓存计算结果**

✓ 正确示例：
```c
typedef struct {
    int input;
    int result;
    bool valid;
} CacheEntry;

int cached_fibonacci(int n, CacheEntry* cache) {
    if (n < 2) return n;
    if (cache[n].valid) {
        return cache[n].result;  // 缓存命中
    }
    cache[n].result = cached_fibonacci(n-1, cache) + 
                       cached_fibonacci(n-2, cache);
    cache[n].valid = true;
    return cache[n].result;
}
```

### 1.3 缓存友好代码

**规则1.3.1：数据结构缓存对齐**

✓ 正确示例：
```c
// 缓存行对齐（通常64字节）
typedef struct {
    int data[15];
    char padding[4];  // 填充到64字节
} __attribute__((aligned(64))) CacheAlignedStruct;

// 避免伪共享
typedef struct {
    volatile int counter;
    char padding[60];  // 隔离到不同缓存行
} __attribute__((aligned(64))) PerThreadCounter;
```

✗ 错误示例：
```c
// 未对齐，可能导致伪共享
typedef struct {
    volatile int counter;  // 4字节
} Counter;  // 多线程访问同一缓存行，性能下降
```

**规则1.3.2：顺序访问优于随机访问**

✓ 正确示例：
```c
// 按行优先顺序访问（缓存友好）
void matrix_sum_rows(int** matrix, int rows, int cols) {
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            matrix[i][j] *= 2;  // 连续内存访问
        }
    }
}
```

✗ 错误示例：
```c
// 按列访问（缓存不友好）
void matrix_sum_cols(int** matrix, int rows, int cols) {
    for (int j = 0; j < cols; j++) {
        for (int i = 0; i < rows; i++) {
            matrix[i][j] *= 2;  // 跳跃访问，缓存miss
        }
    }
}
```

### 1.4 冗余计算消除

**规则1.4.1：提取循环不变代码**

✓ 正确示例：
```c
void process_data(float* data, size_t n, float factor) {
    float scaled_factor = factor * 0.5f;  // 提取到循环外
    for (size_t i = 0; i < n; i++) {
        data[i] = data[i] * scaled_factor + 1.0f;
    }
}
```

✗ 错误示例：
```c
void process_data_slow(float* data, size_t n, float factor) {
    for (size_t i = 0; i < n; i++) {
        data[i] = data[i] * (factor * 0.5f) + 1.0f;  // 重复计算
    }
}
```

**规则1.4.2：强度削减优化**

✓ 正确示例：
```c
// 用加法替代乘法
void compute_offsets(int* offsets, int stride, size_t n) {
    offsets[0] = 0;
    for (size_t i = 1; i < n; i++) {
        offsets[i] = offsets[i-1] + stride;  // 加法
    }
}
```

✗ 错误示例：
```c
void compute_offsets_slow(int* offsets, int stride, size_t n) {
    for (size_t i = 0; i < n; i++) {
        offsets[i] = i * stride;  // 乘法
    }
}
```

### 1.5 内联函数使用

**规则1.5.1：合理使用inline关键字**

✓ 正确示例：
```c
// 小型频繁调用的函数使用内联
static inline int max(int a, int b) {
    return (a > b) ? a : b;
}

static inline void swap_int(int* a, int* b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}
```

✗ 错误示例：
```c
// 大型函数不应内联
inline void large_function(int* data, size_t n) {
    // 100+行代码
    // 内联会导致代码膨胀
}

// 递归函数不应内联
inline int recursive_func(int n) {
    if (n <= 1) return 1;
    return n * recursive_func(n - 1);
}
```

## 2. 内存优化

### 2.1 数据结构选择

**规则2.1.1：根据访问模式选择数据结构**

✓ 正确示例：
```c
// 随机访问：使用数组
typedef struct {
    int* data;
    size_t size;
} Array;

int array_get(Array* arr, size_t index) {
    return arr->data[index];  // O(1)，缓存友好
}

// 频繁插入删除：使用链表
typedef struct Node {
    int data;
    struct Node* next;
} Node;

void list_insert(Node** head, int data) {
    Node* new_node = malloc(sizeof(Node));
    new_node->data = data;
    new_node->next = *head;
    *head = new_node;  // O(1)插入
}

// 固定大小缓冲区：使用环形缓冲区
typedef struct {
    int* buffer;
    size_t capacity;
    size_t head;
    size_t tail;
} RingBuffer;
```

**规则2.1.2：减少内存碎片**

✓ 正确示例：
```c
// 使用数组而非分散的小结构
typedef struct {
    int id;
    char name[32];
} Entity;

typedef struct {
    Entity entities[1000];  // 连续内存
    size_t count;
} EntityManager;
```

✗ 错误示例：
```c
// 分散的内存分配
Entity* entities[1000];
for (int i = 0; i < 1000; i++) {
    entities[i] = malloc(sizeof(Entity));  // 内存碎片
}
```

### 2.2 内存池管理

**规则2.2.1：使用内存池替代频繁malloc/free**

✓ 正确示例：
```c
typedef struct {
    void* memory;
    size_t capacity;
    size_t used;
} MemoryPool;

MemoryPool* pool_create(size_t capacity) {
    MemoryPool* pool = malloc(sizeof(MemoryPool));
    pool->memory = malloc(capacity);
    pool->capacity = capacity;
    pool->used = 0;
    return pool;
}

void* pool_alloc(MemoryPool* pool, size_t size) {
    if (pool->used + size > pool->capacity) {
        return NULL;  // 内存不足
    }
    void* ptr = (char*)pool->memory + pool->used;
    pool->used += size;
    return ptr;  // O(1)分配
}

void pool_reset(MemoryPool* pool) {
    pool->used = 0;  // 批量释放
}

void pool_destroy(MemoryPool* pool) {
    free(pool->memory);
    free(pool);
}
```

**规则2.2.2：对象池模式**

✓ 正确示例：
```c
typedef struct Object {
    int data;
    struct Object* next;
} Object;

typedef struct {
    Object* free_list;
    Object* objects;
    size_t capacity;
} ObjectPool;

ObjectPool* objpool_create(size_t capacity) {
    ObjectPool* pool = malloc(sizeof(ObjectPool));
    pool->objects = malloc(sizeof(Object) * capacity);
    pool->capacity = capacity;
    pool->free_list = NULL;
    
    // 初始化空闲链表
    for (size_t i = 0; i < capacity; i++) {
        pool->objects[i].next = pool->free_list;
        pool->free_list = &pool->objects[i];
    }
    return pool;
}

Object* objpool_alloc(ObjectPool* pool) {
    if (pool->free_list == NULL) {
        return NULL;  // 池已满
    }
    Object* obj = pool->free_list;
    pool->free_list = obj->next;
    return obj;  // O(1)
}

void objpool_free(ObjectPool* pool, Object* obj) {
    obj->next = pool->free_list;
    pool->free_list = obj;  // O(1)
}
```

### 2.3 内存生命周期管理

**规则2.3.1：栈分配优于堆分配**

✓ 正确示例：
```c
void process_small_data(void) {
    int buffer[256];  // 栈上分配，快
    // 使用buffer
}  // 自动释放

typedef struct {
    int x, y, z;
} Vector3;

Vector3 vector_add(Vector3 a, Vector3 b) {
    Vector3 result = {a.x + b.x, a.y + b.y, a.z + b.z};
    return result;  // 返回值优化
}
```

✗ 错误示例：
```c
void process_small_data_slow(void) {
    int* buffer = malloc(256 * sizeof(int));  // 不必要的堆分配
    // 使用buffer
    free(buffer);  // 需要手动释放
}
```

**规则2.3.2：避免内存泄漏**

✓ 正确示例：
```c
int read_file(const char* filename, char** content, size_t* size) {
    FILE* file = fopen(filename, "r");
    if (!file) return -1;
    
    fseek(file, 0, SEEK_END);
    *size = ftell(file);
    fseek(file, 0, SEEK_SET);
    
    *content = malloc(*size + 1);
    if (!*content) {
        fclose(file);
        return -1;
    }
    
    if (fread(*content, 1, *size, file) != *size) {
        fclose(file);
        free(*content);
        *content = NULL;
        return -1;
    }
    
    fclose(file);
    (*content)[*size] = '\0';
    return 0;
}
```

### 2.4 避免不必要的内存拷贝

**规则2.4.1：使用指针替代结构体拷贝**

✓ 正确示例：
```c
typedef struct {
    int data[1000];
} LargeStruct;

void process_large_struct(const LargeStruct* s) {
    // 使用指针，避免拷贝
    int sum = 0;
    for (int i = 0; i < 1000; i++) {
        sum += s->data[i];
    }
}
```

✗ 错误示例：
```c
void process_large_struct_slow(LargeStruct s) {
    // 拷贝整个结构体（4000字节）
    int sum = 0;
    for (int i = 0; i < 1000; i++) {
        sum += s.data[i];
    }
}
```

**规则2.4.2：使用restrict关键字优化指针别名**

✓ 正确示例：
```c
// restrict告知编译器指针不重叠
void vector_add(float* restrict a, 
                const float* restrict b, 
                const float* restrict c, 
                size_t n) {
    for (size_t i = 0; i < n; i++) {
        a[i] = b[i] + c[i];  // 编译器可以向量化
    }
}
```

✗ 错误示例：
```c
void vector_add_slow(float* a, const float* b, const float* c, size_t n) {
    for (size_t i = 0; i < n; i++) {
        a[i] = b[i] + c[i];  // 编译器保守处理，无法优化
    }
}
```

## 3. 位操作优化

### 3.1 使用位操作替代算术运算

**规则3.1.1：用位操作替代乘除法**

✓ 正确示例：
```c
int multiply_by_8(int x) {
    return x << 3;  // 乘以8
}

int divide_by_4(int x) {
    return x >> 2;  // 除以4
}

int modulo_8(int x) {
    return x & 7;   // 模8
}

bool is_power_of_2(int x) {
    return x > 0 && (x & (x - 1)) == 0;
}

int next_power_of_2(int x) {
    x--;
    x |= x >> 1;
    x |= x >> 2;
    x |= x >> 4;
    x |= x >> 8;
    x |= x >> 16;
    return x + 1;
}
```

✗ 错误示例：
```c
int multiply_by_8_slow(int x) {
    return x * 8;  // 可能产生乘法指令
}

int divide_by_4_slow(int x) {
    return x / 4;  // 可能产生除法指令
}
```

### 3.2 位域优化

**规则3.2.1：合理使用位域节省内存**

✓ 正确示例：
```c
typedef struct {
    unsigned int flag1 : 1;
    unsigned int flag2 : 1;
    unsigned int flag3 : 1;
    unsigned int type  : 2;
    unsigned int value : 12;
} CompactFlags;  // 2字节
```

✗ 错误示例：
```c
typedef struct {
    bool flag1;     // 1字节
    bool flag2;     // 1字节
    bool flag3;     // 1字节
    int type;       // 4字节
    int value;      // 4字节
} WastefulFlags;    // 共12字节
```

**规则3.2.2：位操作技巧**

✓ 正确示例：
```c
// 交换两个数（无临时变量）
void swap_xor(int* a, int* b) {
    if (a != b) {
        *a ^= *b;
        *b ^= *a;
        *a ^= *b;
    }
}

// 快速计数置位个数
int popcount(uint32_t x) {
    x = x - ((x >> 1) & 0x55555555);
    x = (x & 0x33333333) + ((x >> 2) & 0x33333333);
    x = (x + (x >> 4)) & 0x0F0F0F0F;
    x = x + (x >> 8);
    x = x + (x >> 16);
    return x & 0x7F;
}

// 找最低置位
int find_first_set(uint32_t x) {
    return __builtin_ffs(x);  // GCC内置函数
}

// 找最高置位
int find_last_set(uint32_t x) {
    return 31 - __builtin_clz(x);  // GCC内置函数
}
```

## 4. IO优化

### 4.1 批量IO操作

**规则4.1.1：使用缓冲IO**

✓ 正确示例：
```c
// 设置大缓冲区
void buffered_io_example(const char* filename) {
    FILE* file = fopen(filename, "w");
    char buffer[64 * 1024];  // 64KB缓冲区
    setvbuf(file, buffer, _IOFBF, sizeof(buffer));
    
    for (int i = 0; i < 100000; i++) {
        fprintf(file, "%d\n", i);  // 缓冲写入
    }
    fclose(file);
}
```

✗ 错误示例：
```c
void unbuffered_io_example(const char* filename) {
    FILE* file = fopen(filename, "w");
    for (int i = 0; i < 100000; i++) {
        fprintf(file, "%d\n", i);  // 频繁系统调用
    }
    fclose(file);
}
```

**规则4.1.2：批量读写**

✓ 正确示例：
```c
// 批量写入
void batch_write(int* data, size_t n, const char* filename) {
    FILE* file = fopen(filename, "wb");
    fwrite(data, sizeof(int), n, file);  // 一次写入
    fclose(file);
}

// 批量读取
void batch_read(int* data, size_t n, const char* filename) {
    FILE* file = fopen(filename, "rb");
    fread(data, sizeof(int), n, file);  // 一次读取
    fclose(file);
}
```

### 4.2 内存映射文件

**规则4.2.1：使用mmap处理大文件**

✓ 正确示例：
```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

void process_large_file(const char* filename) {
    int fd = open(filename, O_RDONLY);
    struct stat st;
    fstat(fd, &st);
    size_t size = st.st_size;
    
    // 映射到内存
    void* mapped = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);
    close(fd);
    
    if (mapped == MAP_FAILED) {
        return;
    }
    
    // 直接访问内存
    char* data = (char*)mapped;
    for (size_t i = 0; i < size; i++) {
        // 处理data[i]
    }
    
    munmap(mapped, size);
}
```

### 4.3 异步IO

**规则4.3.1：使用异步IO提升并发性能**

✓ 正确示例：
```c
#ifdef __linux__
#include <libaio.h>

void async_io_example(int fd, void* buffer, size_t size, off_t offset) {
    io_context_t ctx = 0;
    io_setup(128, &ctx);
    
    struct iocb cb;
    struct iocb* cbs[1] = {&cb};
    
    io_prep_pread(&cb, fd, buffer, size, offset);
    
    io_submit(ctx, 1, cbs);
    
    struct io_event events[1];
    io_getevents(ctx, 1, 1, events, NULL);
    
    io_destroy(ctx);
}
#endif
```

## 5. 编译器优化选项

### 5.1 优化级别选择

**规则5.1.1：选择合适的优化级别**

```bash
# 开发阶段：-O0或-O1（快速编译，便于调试）
gcc -O0 -g program.c

# 一般优化：-O2（平衡性能和代码大小）
gcc -O2 program.c

# 最高优化：-O3（可能增加代码大小）
gcc -O3 program.c

# 针对特定CPU优化
gcc -O3 -march=native program.c

# 链接时优化（LTO）
gcc -O3 -flto program.c
```

### 5.2 编译器内置函数

**规则5.2.1：使用编译器内置函数**

✓ 正确示例：
```c
// 分支预测提示
#define LIKELY(x)   __builtin_expect(!!(x), 1)
#define UNLIKELY(x) __builtin_expect(!!(x), 0)

void process_data(int* data, size_t n) {
    for (size_t i = 0; i < n; i++) {
        if (UNLIKELY(data[i] < 0)) {
            handle_error();
        } else {
            process_normal(data[i]);
        }
    }
}

// 快速内存操作
void fast_copy(void* dst, const void* src, size_t n) {
    __builtin_memcpy(dst, src, n);  // 可能内联
}

// 位操作内置函数
int count_bits_builtin(uint32_t x) {
    return __builtin_popcount(x);
}

int find_first_one(uint32_t x) {
    return __builtin_ffs(x);
}

int count_leading_zeros(uint32_t x) {
    return __builtin_clz(x);
}
```

### 5.3 编译器属性

**规则5.3.1：使用函数属性优化**

✓ 正确示例：
```c
// 纯函数（无副作用，可优化重复调用）
__attribute__((pure)) int compute_hash(const char* str);

// 常量函数（仅依赖参数）
__attribute__((const)) int factorial(int n);

// 不返回的函数
__attribute__((noreturn)) void fatal_error(const char* msg);

// 对齐属性
typedef struct {
    int value;
} __attribute__((aligned(16))) AlignedStruct;

// 打包属性（无填充）
typedef struct __attribute__((packed)) {
    uint8_t type;
    uint32_t length;
    uint8_t data[1];
} PackedHeader;

// 分支预测
__attribute__((hot)) void critical_path(void);
__attribute__((cold)) void error_handler(void);
```

## 6. SIMD向量化优化

### 6.1 SIMD基础概念

**规则6.1.1：理解SIMD性能优势**

SIMD（Single Instruction Multiple Data）单指令多数据流，一次指令处理多个数据，可显著提升数值计算性能。

```c
// 标量运算：每次处理1个数据
for (int i = 0; i < n; i++) {
    c[i] = a[i] + b[i];  // n次加法指令
}

// SIMD运算：每次处理4/8/16个数据（取决于指令集）
// AVX-256: 8个float或4个double
// AVX-512: 16个float或8个double
```

### 6.2 编译器自动向量化

**规则6.2.1：编写向量化友好代码**

✓ 正确示例：
```c
// 连续内存访问，无数据依赖
void vector_add(float* restrict a, 
                const float* restrict b, 
                const float* restrict c, 
                size_t n) {
    for (size_t i = 0; i < n; i++) {
        a[i] = b[i] + c[i];  // 编译器可自动向量化
    }
}

// 编译选项：gcc -O3 -march=native -ftree-vectorize
// 查看向量化报告：gcc -O3 -fopt-info-vec-missed
```

✗ 错误示例：
```c
// 数据依赖，无法向量化
void bad_example(float* a, size_t n) {
    for (size_t i = 1; i < n; i++) {
        a[i] = a[i-1] + 1;  // 依赖前一次结果
    }
}

// 指针别名，无法向量化
void bad_example2(float* a, float* b, size_t n) {
    for (size_t i = 0; i < n; i++) {
        a[i] = b[i] * 2;  // a和b可能重叠
    }
}
```

**规则6.2.2：使用编译器提示**

```c
// GCC/Clang 向量化提示
#pragma GCC ivdep  // 忽略向量依赖
for (int i = 0; i < n; i++) {
    a[i] = b[i] + c[i];
}

// 指定循环展开次数
#pragma GCC unroll 4
for (int i = 0; i < n; i++) {
    a[i] = b[i] * c[i];
}

// OpenMP向量化
#pragma omp simd aligned(a, b, c: 32)
for (int i = 0; i < n; i++) {
    a[i] = b[i] + c[i];
}
```

### 6.3 SSE/AVX Intrinsics

**规则6.3.1：SSE基础操作（128位）**

```c
#include <immintrin.h>  // 包含所有SIMD头文件

void sse_add(float* a, const float* b, const float* c, size_t n) {
    size_t i = 0;
    // 每次处理4个float（128位 / 32位 = 4）
    for (; i + 4 <= n; i += 4) {
        __m128 vb = _mm_loadu_ps(&b[i]);  // 加载4个float
        __m128 vc = _mm_loadu_ps(&c[i]);
        __m128 va = _mm_add_ps(vb, vc);   // 向量加法
        _mm_storeu_ps(&a[i], va);          // 存储4个float
    }
    // 处理剩余元素
    for (; i < n; i++) {
        a[i] = b[i] + c[i];
    }
}

// 对齐加载（更快）
void sse_add_aligned(float* a, const float* b, const float* c, size_t n) {
    // 确保内存16字节对齐
    // float* a = aligned_alloc(16, n * sizeof(float));
    
    for (size_t i = 0; i + 4 <= n; i += 4) {
        __m128 vb = _mm_load_ps(&b[i]);   // 假设已对齐
        __m128 vc = _mm_load_ps(&c[i]);
        __m128 va = _mm_add_ps(vb, vc);
        _mm_store_ps(&a[i], va);          // 对齐存储
    }
}
```

**规则6.3.2：AVX基础操作（256位）**

```c
#include <immintrin.h>

void avx_add(float* a, const float* b, const float* c, size_t n) {
    size_t i = 0;
    // 每次处理8个float（256位 / 32位 = 8）
    for (; i + 8 <= n; i += 8) {
        __m256 vb = _mm256_loadu_ps(&b[i]);  // 加载8个float
        __m256 vc = _mm256_loadu_ps(&c[i]);
        __m256 va = _mm256_add_ps(vb, vc);   // 向量加法
        _mm256_storeu_ps(&a[i], va);          // 存储8个float
    }
    // 处理剩余元素
    for (; i < n; i++) {
        a[i] = b[i] + c[i];
    }
}

// AVX矩阵乘法优化
void avx_dot_product(float* result, const float* a, const float* b, size_t n) {
    __m256 sum = _mm256_setzero_ps();
    
    for (size_t i = 0; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 vb = _mm256_loadu_ps(&b[i]);
        sum = _mm256_add_ps(sum, _mm256_mul_ps(va, vb));
    }
    
    // 水平求和
    __m128 hi = _mm256_extractf128_ps(sum, 1);
    __m128 lo = _mm256_castps256_ps128(sum);
    __m128 sum128 = _mm_add_ps(lo, hi);
    sum128 = _mm_hadd_ps(sum128, sum128);
    sum128 = _mm_hadd_ps(sum128, sum128);
    
    *result = _mm_cvtss_f32(sum128);
    
    // 处理剩余元素
    for (size_t i = n - n % 8; i < n; i++) {
        *result += a[i] * b[i];
    }
}
```

### 6.4 ARM NEON（移动端/嵌入式）

**规则6.4.1：NEON基础操作**

```c
#ifdef __ARM_NEON
#include <arm_neon.h>

void neon_add(float* a, const float* b, const float* c, size_t n) {
    size_t i = 0;
    // 每次处理4个float（128位）
    for (; i + 4 <= n; i += 4) {
        float32x4_t vb = vld1q_f32(&b[i]);  // 加载4个float
        float32x4_t vc = vld1q_f32(&c[i]);
        float32x4_t va = vaddq_f32(vb, vc);  // 向量加法
        vst1q_f32(&a[i], va);                // 存储4个float
    }
    // 处理剩余元素
    for (; i < n; i++) {
        a[i] = b[i] + c[i];
    }
}
#endif
```

### 6.5 SIMD最佳实践

**规则6.5.1：选择正确的SIMD级别**

| 场景 | 推荐方案 |
|------|----------|
| 简单循环 | 编译器自动向量化 (-O3 -march=native) |
| 性能关键路径 | SSE/AVX Intrinsics |
| 跨平台代码 | OpenMP SIMD 或编译器提示 |
| ARM平台 | NEON Intrinsics |
| 便携性要求 | Intel ISPC 或库函数（ipp, mkl） |

**规则6.5.2：内存对齐要求**

```c
// SSE: 16字节对齐
// AVX: 32字节对齐
// AVX-512: 64字节对齐

#include <stdlib.h>

// C11 对齐分配
float* aligned_array_alloc(size_t n, size_t alignment) {
    float* ptr = NULL;
    if (posix_memalign((void**)&ptr, alignment, n * sizeof(float)) != 0) {
        return NULL;
    }
    return ptr;
}

// 使用示例
float* data = aligned_array_alloc(1024, 32);  // 32字节对齐（AVX）

// C17 aligned_alloc
float* data2 = aligned_alloc(32, 1024 * sizeof(float));
```

**规则6.5.3：避免SIMD陷阱**

✗ 错误示例：
```c
// 陷阱1：非对齐访问导致性能下降或崩溃
float data[100];  // 可能不对齐
__m256 v = _mm256_load_ps(data);  // 危险！使用loadu

// 陷阱2：混合VEX和SSE代码（AVX-SSE过渡惩罚）
// 解决：编译时使用 -mavx 或在SSE代码后加 _mm256_zeroupper()

// 陷阱3：假依赖
__m128 a = _mm_add_ps(b, c);
__m128 d = _mm_add_ps(a, e);  // 依赖a，无法并行
```

✓ 正确示例：
```c
// 使用非对齐加载
__m256 v = _mm256_loadu_ps(data);  // 安全

// 清除AVX高位，避免惩罚
void after_avx_sse_mix() {
    _mm256_zeroupper();  // 在AVX代码后、SSE代码前调用
}

// 减少依赖链
__m128 sum1 = _mm_add_ps(b, c);
__m128 sum2 = _mm_add_ps(d, e);  // 独立计算
__m128 result = _mm_add_ps(sum1, sum2);
```

## 7. 并发优化

### 7.1 多线程性能

**规则7.1.1：避免伪共享**

✓ 正确示例：
```c
// 使用缓存行对齐
#define CACHE_LINE_SIZE 64

typedef struct {
    volatile int counter;
    char padding[CACHE_LINE_SIZE - sizeof(int)];
} __attribute__((aligned(CACHE_LINE_SIZE))) ThreadCounter;

ThreadCounter counters[MAX_THREADS];  // 每个线程独立的缓存行
```

✗ 错误示例：
```c
int counters[MAX_THREADS];  // 所有计数器在同一缓存行
```

**规则7.1.2：使用原子操作优化**

✓ 正确示例：
```c
#include <stdatomic.h>

// 原子计数器
atomic_int global_counter = ATOMIC_VAR_INIT(0);

void increment_counter(void) {
    atomic_fetch_add_explicit(&global_counter, 1, 
                              memory_order_relaxed);
}

// 比较交换循环
int update_value(atomic_int* ptr, int new_value) {
    int old_value = atomic_load(ptr);
    while (!atomic_compare_exchange_weak(ptr, &old_value, new_value)) {
        // old_value已更新
    }
    return old_value;
}
```

## 8. 性能分析工具

### 8.1 性能分析工具使用

**规则8.1.1：使用profiler定位热点**

```bash
# gprof性能分析
gcc -pg program.c -o program
./program
gprof program gmon.out > analysis.txt

# perf性能分析
perf record -g ./program
perf report

# Valgrind/Callgrind
valgrind --tool=callgrind ./program
kcachegrind callgrind.out.*

# strace系统调用追踪
strace -c ./program

# time命令测量时间
time ./program
```

### 8.2 内存分析

**规则8.2.1：检测内存问题**

```bash
# Valgrind内存检查
valgrind --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         ./program

# AddressSanitizer（编译时检测）
gcc -fsanitize=address -g program.c

# MemorySanitizer
gcc -fsanitize=memory -g program.c

# UndefinedBehaviorSanitizer
gcc -fsanitize=undefined -g program.c
```

---

## 性能优化检查清单

### CPU优化
- [ ] 选择时间复杂度最优的算法
- [ ] 使用查找表替代重复计算
- [ ] 数据结构缓存对齐
- [ ] 顺序访问而非随机访问
- [ ] 提取循环不变代码
- [ ] 强度削减（乘法→加法）
- [ ] 合理使用inline函数

### 内存优化
- [ ] 栈分配优先于堆分配
- [ ] 使用内存池减少malloc/free
- [ ] 避免内存碎片
- [ ] 使用指针避免结构体拷贝
- [ ] 使用restrict关键字
- [ ] 避免内存泄漏

### 位操作优化
- [ ] 用位移替代乘除法
- [ ] 合理使用位域节省空间
- [ ] 使用内置位操作函数

### IO优化
- [ ] 使用缓冲IO
- [ ] 批量读写而非逐字节
- [ ] 大文件使用mmap
- [ ] 减少系统调用次数

### 编译器优化
- [ ] 选择合适的优化级别(-O2/-O3)
- [ ] 使用-march=native
- [ ] 启用LTO链接时优化
- [ ] 使用编译器内置函数
- [ ] 合理使用函数属性

### 并发优化
- [ ] 避免伪共享
- [ ] 使用适当的原子操作
- [ ] 选择合适的内存序

### SIMD向量化
- [ ] 优先使用编译器自动向量化
- [ ] 内存对齐满足SIMD要求
- [ ] 使用restrict避免指针别名
- [ ] 必要时使用SSE/AVX Intrinsics
- [ ] 处理剩余元素

### 性能分析
- [ ] 使用profiler定位热点
- [ ] 检测内存泄漏
- [ ] 分析缓存miss
- [ ] 追踪系统调用开销
