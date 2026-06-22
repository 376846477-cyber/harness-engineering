# C语言并发规范详细参考

## 一、线程管理

### 1.1 线程创建与销毁

```c
#include <pthread.h>

void* thread_func(void* arg) {
    int* data = (int*)arg;
    // 线程工作
    free(data);
    return NULL;
}

int create_thread(pthread_t* tid, int value) {
    int* arg = malloc(sizeof(int));
    if (!arg) return -1;
    *arg = value;
    
    int ret = pthread_create(tid, NULL, thread_func, arg);
    if (ret != 0) {
        free(arg);
        return -1;
    }
    return 0;
}
```

### 1.2 线程生命周期规则

- **必须join或detach**：创建的线程必须被join(pthread_join)或detach(pthread_detach)
- **资源清理**：join会回收线程资源，detach让线程自动回收
- **参数传递**：通过arg传递的数据必须保证生命周期足够长，通常需要动态分配

## 二、互斥锁(Mutex)规范

### 2.1 Mutex初始化与销毁

```c
// 静态初始化（推荐）
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// 动态初始化
pthread_mutex_t mutex;
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);  // 可重入锁
pthread_mutex_init(&mutex, &attr);
pthread_mutexattr_destroy(&attr);

// 销毁
pthread_mutex_destroy(&mutex);
```

### 2.2 Mutex使用规范

```c
// 规范示例：使用RAII风格的锁管理
typedef struct {
    pthread_mutex_t mutex;
    int counter;
} safe_counter_t;

void counter_increment(safe_counter_t* sc) {
    pthread_mutex_lock(&sc->mutex);
    sc->counter++;
    pthread_mutex_unlock(&sc->mutex);
}

// 推荐使用trylock避免死锁
int counter_try_increment(safe_counter_t* sc) {
    if (pthread_mutex_trylock(&sc->mutex) != 0) {
        return -1;  // 获取锁失败
    }
    sc->counter++;
    pthread_mutex_unlock(&sc->mutex);
    return 0;
}
```

### 2.3 Mutex使用规则

- **配对使用**：lock和unlock必须成对出现
- **避免重复加锁**：普通mutex不可递归加锁，需要时使用PTHREAD_MUTEX_RECURSIVE
- **锁粒度最小化**：只保护真正需要同步的代码段
- **避免在持锁时调用未知函数**：可能导致死锁
- **错误处理**：检查返回值，特别是trylock

## 三、条件变量(Condition Variable)

### 3.1 基本使用模式

```c
typedef struct {
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    int ready;
    int data;
} producer_consumer_t;

// 等待条件
void wait_for_data(producer_consumer_t* pc) {
    pthread_mutex_lock(&pc->mutex);
    while (pc->ready == 0) {  // 必须用while，避免虚假唤醒
        pthread_cond_wait(&pc->cond, &pc->mutex);
    }
    int data = pc->data;
    pc->ready = 0;
    pthread_mutex_unlock(&pc->mutex);
}

// 通知条件
void produce_data(producer_consumer_t* pc, int value) {
    pthread_mutex_lock(&pc->mutex);
    pc->data = value;
    pc->ready = 1;
    pthread_cond_signal(&pc->cond);  // 或pthread_cond_broadcast
    pthread_mutex_unlock(&pc->mutex);
}
```

### 3.2 条件变量规则

- **必须关联mutex**：条件变量必须在持锁状态下使用
- **使用while循环**：避免虚假唤醒(spurious wakeup)
- **先修改状态后通知**：在改变条件后再signal/broadcast
- **broadcast vs signal**：需要唤醒所有等待者时用broadcast

## 四、读写锁(ReadWrite Lock)

### 4.1 使用场景

```c
typedef struct {
    pthread_rwlock_t rwlock;
    int cache[100];
    int valid;
} cache_t;

int cache_read(cache_t* c, int index) {
    pthread_rwlock_rdlock(&c->rwlock);  // 读锁，允许多线程并发读
    int value = c->cache[index];
    pthread_rwlock_unlock(&c->rwlock);
    return value;
}

void cache_update(cache_t* c, int index, int value) {
    pthread_rwlock_wrlock(&c->rwlock);  // 写锁，独占访问
    c->cache[index] = value;
    c->valid = 1;
    pthread_rwlock_unlock(&c->rwlock);
}
```

### 4.2 读写锁规则

- **适用于读多写少场景**：读操作可并发，写操作独占
- **避免写者饥饿**：考虑使用PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE
- **不要升级锁**：不要在持读锁时尝试获取写锁

## 五、死锁预防策略

### 5.1 固定加锁顺序

```c
// 错误示例：可能导致死锁
void transfer(account_t* from, account_t* to, int amount) {
    pthread_mutex_lock(&from->mutex);
    pthread_mutex_lock(&to->mutex);  // 顺序不一致会导致死锁
    // ...
}

// 正确示例：按地址顺序加锁
void transfer_safe(account_t* from, account_t* to, int amount) {
    account_t* first = (from < to) ? from : to;
    account_t* second = (from < to) ? to : from;
    
    pthread_mutex_lock(&first->mutex);
    pthread_mutex_lock(&second->mutex);
    // 转账操作
    pthread_mutex_unlock(&second->mutex);
    pthread_mutex_unlock(&first->mutex);
}
```

### 5.2 Trylock超时机制

```c
int lock_two_mutexes(pthread_mutex_t* m1, pthread_mutex_t* m2) {
    if (pthread_mutex_trylock(m1) != 0) return -1;
    if (pthread_mutex_trylock(m2) != 0) {
        pthread_mutex_unlock(m1);
        return -1;
    }
    return 0;
}
```

### 5.3 死锁预防规则

- **固定顺序**：多个锁必须按固定全局顺序获取
- **避免嵌套**：尽量减少持锁时获取另一个锁
- **超时机制**：使用trylock + 超时或回退
- **锁粒度**：缩小临界区，减少持锁时间
- **文档化**：明确注释锁的获取顺序

## 六、线程局部存储(Thread-Local Storage)

### 6.1 __thread关键字

```c
// 每个线程独立的变量
static __thread int thread_local_counter = 0;

void increment_thread_counter(void) {
    thread_local_counter++;  // 每个线程独立的计数
}
```

### 6.2 pthread_key方式

```c
static pthread_key_t key;
static pthread_once_t key_once = PTHREAD_ONCE_INIT;

static void make_key(void) {
    pthread_key_create(&key, free);  // 线程退出时自动free
}

void* get_thread_buffer(void) {
    pthread_once(&key_once, make_key);
    void* buffer = pthread_getspecific(key);
    if (buffer == NULL) {
        buffer = malloc(BUFFER_SIZE);
        pthread_setspecific(key, buffer);
    }
    return buffer;
}
```

## 七、原子操作(C11 stdatomic.h)

### 7.1 基本使用

```c
#include <stdatomic.h>

atomic_int counter = ATOMIC_VAR_INIT(0);
atomic_flag lock = ATOMIC_FLAG_INIT;

// 原子操作
void atomic_increment(void) {
    atomic_fetch_add(&counter, 1);
}

int atomic_get_value(void) {
    return atomic_load(&counter);
}

// 自旋锁实现
void spin_lock(atomic_flag* flag) {
    while (atomic_flag_test_and_set(flag)) {
        // 自旋等待
    }
}

void spin_unlock(atomic_flag* flag) {
    atomic_flag_clear(flag);
}
```

### 7.2 内存序(Memory Order)

```c
// 宽松序：只保证原子性
atomic_load_explicit(&var, memory_order_relaxed);

// 获取-释放序：保证可见性
atomic_store_explicit(&flag, 1, memory_order_release);
while (atomic_load_explicit(&flag, memory_order_acquire) == 0);

// 顺序一致性：默认，最严格
atomic_load(&var);  // 等价于memory_order_seq_cst
```

## 八、线程安全数据结构

### 8.1 线程安全队列示例

```c
typedef struct node {
    int data;
    struct node* next;
} node_t;

typedef struct {
    node_t* head;
    node_t* tail;
    pthread_mutex_t mutex;
    pthread_cond_t cond;
} thread_queue_t;

void queue_push(thread_queue_t* q, int data) {
    node_t* n = malloc(sizeof(node_t));
    n->data = data;
    n->next = NULL;
    
    pthread_mutex_lock(&q->mutex);
    if (q->tail == NULL) {
        q->head = q->tail = n;
    } else {
        q->tail->next = n;
        q->tail = n;
    }
    pthread_cond_signal(&q->cond);
    pthread_mutex_unlock(&q->mutex);
}

int queue_pop(thread_queue_t* q) {
    pthread_mutex_lock(&q->mutex);
    while (q->head == NULL) {
        pthread_cond_wait(&q->cond, &q->mutex);
    }
    node_t* n = q->head;
    q->head = q->head->next;
    if (q->head == NULL) q->tail = NULL;
    pthread_mutex_unlock(&q->mutex);
    
    int data = n->data;
    free(n);
    return data;
}
```

## 九、检查清单

### 线程管理

- [ ] 所有创建的线程都已join或detach
- [ ] 线程参数的生命周期管理正确
- [ ] 线程函数有明确的退出路径

### Mutex使用

- [ ] lock/unlock配对，无遗漏
- [ ] 没有对普通mutex重复加锁
- [ ] 持锁时间足够短
- [ ] 没有在持锁时调用可能阻塞的函数

### 条件变量

- [ ] 使用while循环检查条件
- [ ] signal/broadcast在持锁状态调用
- [ ] 条件变量与正确的mutex关联

### 死锁预防

- [ ] 多个锁的获取顺序全局一致
- [ ] 没有循环等待
- [ ] 考虑使用trylock作为防护

### 资源管理

- [ ] 动态创建的mutex/cond/rwlock正确销毁
- [ ] thread-local资源有线程退出时的清理机制
- [ ] 没有内存泄漏

### 原子操作

- [ ] 正确选择内存序
- [ ] 原子操作用于简单变量，复杂操作使用mutex
- [ ] 避免ABA问题（必要时使用版本号）
