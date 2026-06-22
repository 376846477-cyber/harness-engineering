# C语言可靠性规范详细参考

## 一、系统输入可靠性

### 1.1 操作防呆设计

#### 规则1.1.1：输入参数有效性校验
所有公共接口必须对输入参数进行有效性校验，防止无效输入导致系统异常。

```c
// 错误示例：缺少参数校验
void process_data(char *data, size_t len) {
    memcpy(buffer, data, len);  // 若data为NULL将崩溃
}

// 正确示例：完整的参数校验
int process_data(const char *data, size_t len) {
    if (data == NULL) {
        log_error("process_data: data is NULL");
        return ERR_INVALID_PARAM;
    }
    if (len == 0 || len > MAX_DATA_LEN) {
        log_error("process_data: invalid len=%zu", len);
        return ERR_INVALID_PARAM;
    }
    if (len > buffer_size) {
        log_error("process_data: buffer overflow risk");
        return ERR_BUFFER_TOO_SMALL;
    }
    
    // 实际处理逻辑
    memcpy(buffer, data, len);
    return SUCCESS;
}
```

#### 规则1.1.2：缓冲区边界保护
所有内存操作必须进行边界检查，防止缓冲区溢出。

```c
// 错误示例：strcpy可能溢出
void copy_name(char *dest, const char *src) {
    strcpy(dest, src);  // 危险！
}

// 正确示例：使用安全的字符串操作
int copy_name(char *dest, size_t dest_size, const char *src) {
    if (dest == NULL || src == NULL || dest_size == 0) {
        return ERR_INVALID_PARAM;
    }
    
    size_t src_len = strlen(src);
    if (src_len >= dest_size) {
        log_error("copy_name: src too long (%zu >= %zu)", src_len, dest_size);
        return ERR_BUFFER_TOO_SMALL;
    }
    
    strncpy(dest, src, dest_size - 1);
    dest[dest_size - 1] = '\0';  // 确保null结尾
    return SUCCESS;
}

// 或使用snprintf
int copy_name_safe(char *dest, size_t dest_size, const char *src) {
    if (dest == NULL || src == NULL || dest_size == 0) {
        return ERR_INVALID_PARAM;
    }
    
    int ret = snprintf(dest, dest_size, "%s", src);
    if (ret < 0 || (size_t)ret >= dest_size) {
        return ERR_BUFFER_TOO_SMALL;
    }
    return SUCCESS;
}
```

#### 规则1.1.3：整数溢出防护
对整数运算进行溢出检查，防止算术错误。

```c
// 错误示例：未检查整数溢出
int calculate_size(int count, int unit_size) {
    return count * unit_size;  // 可能溢出
}

// 正确示例：完整的溢出检查
int calculate_size_safe(int count, int unit_size, size_t *result) {
    if (count < 0 || unit_size < 0) {
        return ERR_INVALID_PARAM;
    }
    
    // 检查乘法溢出
    if (count > 0 && unit_size > INT_MAX / count) {
        log_error("calculate_size: integer overflow");
        return ERR_OVERFLOW;
    }
    
    *result = (size_t)count * (size_t)unit_size;
    return SUCCESS;
}

// 使用安全的内存分配
void *safe_malloc_array(size_t count, size_t elem_size) {
    if (count == 0 || elem_size == 0) {
        return NULL;
    }
    
    // 检查size_t溢出
    if (count > SIZE_MAX / elem_size) {
        log_error("safe_malloc_array: size overflow");
        errno = ENOMEM;
        return NULL;
    }
    
    void *ptr = malloc(count * elem_size);
    if (ptr == NULL) {
        log_error("safe_malloc_array: malloc failed for %zu bytes", 
                  count * elem_size);
    }
    return ptr;
}
```

### 1.2 系统过载保护

#### 规则1.2.1：资源配额限制
对系统资源使用进行限制，防止资源耗尽。

```c
#define MAX_CONNECTIONS 1000
#define MAX_MEMORY_MB 512

static atomic_int g_connection_count = 0;
static atomic_size_t g_memory_used = 0;

typedef struct {
    int max_connections;
    int max_memory_mb;
    atomic_int *current_connections;
    atomic_size_t *current_memory;
} resource_quota_t;

int acquire_connection(resource_quota_t *quota) {
    if (quota == NULL) {
        return ERR_INVALID_PARAM;
    }
    
    int current = atomic_load(quota->current_connections);
    if (current >= quota->max_connections) {
        log_warn("connection limit reached: %d/%d", 
                 current, quota->max_connections);
        return ERR_QUOTA_EXCEEDED;
    }
    
    if (!atomic_compare_exchange_strong(quota->current_connections, 
                                        &current, current + 1)) {
        return ERR_CONCURRENT_MODIFICATION;
    }
    
    return SUCCESS;
}

void release_connection(resource_quota_t *quota) {
    if (quota != NULL) {
        atomic_fetch_sub(quota->current_connections, 1);
    }
}
```

#### 规则1.2.2：请求队列限流
实现请求队列和限流机制，防止系统过载。

```c
typedef struct {
    pthread_mutex_t mutex;
    pthread_cond_t not_full;
    pthread_cond_t not_empty;
    void *buffer[QUEUE_SIZE];
    int head;
    int tail;
    int count;
    int max_size;
} bounded_queue_t;

int queue_init(bounded_queue_t *q, int max_size) {
    if (q == NULL || max_size <= 0 || max_size > QUEUE_SIZE) {
        return ERR_INVALID_PARAM;
    }
    
    memset(q, 0, sizeof(*q));
    q->max_size = max_size;
    
    if (pthread_mutex_init(&q->mutex, NULL) != 0) {
        return ERR_SYSTEM_CALL;
    }
    if (pthread_cond_init(&q->not_full, NULL) != 0) {
        pthread_mutex_destroy(&q->mutex);
        return ERR_SYSTEM_CALL;
    }
    if (pthread_cond_init(&q->not_empty, NULL) != 0) {
        pthread_cond_destroy(&q->not_full);
        pthread_mutex_destroy(&q->mutex);
        return ERR_SYSTEM_CALL;
    }
    
    return SUCCESS;
}

int queue_push(bounded_queue_t *q, void *item, int timeout_ms) {
    if (q == NULL || item == NULL) {
        return ERR_INVALID_PARAM;
    }
    
    pthread_mutex_lock(&q->mutex);
    
    // 等待队列不满
    while (q->count >= q->max_size) {
        if (timeout_ms == 0) {
            pthread_mutex_unlock(&q->mutex);
            return ERR_QUEUE_FULL;
        }
        
        struct timespec ts;
        clock_gettime(CLOCK_REALTIME, &ts);
        ts.tv_nsec += (timeout_ms % 1000) * 1000000;
        ts.tv_sec += timeout_ms / 1000 + ts.tv_nsec / 1000000000;
        ts.tv_nsec %= 1000000000;
        
        if (pthread_cond_timedwait(&q->not_full, &q->mutex, &ts) != 0) {
            pthread_mutex_unlock(&q->mutex);
            return ERR_TIMEOUT;
        }
    }
    
    q->buffer[q->tail] = item;
    q->tail = (q->tail + 1) % QUEUE_SIZE;
    q->count++;
    
    pthread_cond_signal(&q->not_empty);
    pthread_mutex_unlock(&q->mutex);
    
    return SUCCESS;
}
```

## 二、系统间通信可靠性

### 2.1 消息可靠发送

#### 规则2.1.1：消息重发机制
实现消息发送的重试机制，确保消息到达。

```c
typedef struct {
    int max_retries;
    int retry_interval_ms;
    int timeout_ms;
    int (*send_func)(void *ctx, const void *data, size_t len);
    void *send_ctx;
} retry_sender_t;

int send_with_retry(retry_sender_t *sender, const void *data, size_t len) {
    if (sender == NULL || data == NULL || len == 0) {
        return ERR_INVALID_PARAM;
    }
    if (sender->send_func == NULL) {
        return ERR_INVALID_PARAM;
    }
    
    int retry_count = 0;
    int last_error = ERR_UNKNOWN;
    
    while (retry_count <= sender->max_retries) {
        int ret = sender->send_func(sender->send_ctx, data, len);
        if (ret == SUCCESS) {
            if (retry_count > 0) {
                log_info("send succeeded after %d retries", retry_count);
            }
            return SUCCESS;
        }
        
        last_error = ret;
        retry_count++;
        
        if (retry_count <= sender->max_retries) {
            log_warn("send failed (attempt %d/%d), retrying in %dms: error=%d",
                     retry_count, sender->max_retries + 1, 
                     sender->retry_interval_ms, ret);
            
            // 指数退避
            int delay = sender->retry_interval_ms * (1 << (retry_count - 1));
            if (delay > 5000) delay = 5000;  // 最大5秒
            
            struct timespec ts = {
                .tv_sec = delay / 1000,
                .tv_nsec = (delay % 1000) * 1000000
            };
            nanosleep(&ts, NULL);
        }
    }
    
    log_error("send failed after %d retries: error=%d", 
              sender->max_retries, last_error);
    return last_error;
}
```

#### 规则2.1.2：消息幂等性设计
确保消息处理的幂等性，防止重复处理。

```c
typedef struct {
    uint64_t msg_id;
    time_t timestamp;
    int processed;
    char data_hash[32];
} message_record_t;

#define MAX_MSG_RECORDS 10000

typedef struct {
    message_record_t records[MAX_MSG_RECORDS];
    int count;
    pthread_mutex_t mutex;
} message_dedup_t;

int is_duplicate_message(message_dedup_t *dedup, uint64_t msg_id, 
                         const char *data_hash) {
    if (dedup == NULL) {
        return ERR_INVALID_PARAM;
    }
    
    pthread_mutex_lock(&dedup->mutex);
    
    for (int i = 0; i < dedup->count; i++) {
        if (dedup->records[i].msg_id == msg_id) {
            // 找到记录，检查hash是否匹配
            if (strcmp(dedup->records[i].data_hash, data_hash) == 0) {
                pthread_mutex_unlock(&dedup->mutex);
                return 1;  // 重复消息
            }
            // 相同ID但不同hash，可能是伪造
            log_warn("message ID collision or tampering: id=%llu", 
                     (unsigned long long)msg_id);
            pthread_mutex_unlock(&dedup->mutex);
            return -1;  // 可疑消息
        }
    }
    
    // 新消息，添加到记录
    if (dedup->count < MAX_MSG_RECORDS) {
        dedup->records[dedup->count].msg_id = msg_id;
        dedup->records[dedup->count].timestamp = time(NULL);
        dedup->records[dedup->count].processed = 0;
        strncpy(dedup->records[dedup->count].data_hash, data_hash, 31);
        dedup->records[dedup->count].data_hash[31] = '\0';
        dedup->count++;
    }
    
    pthread_mutex_unlock(&dedup->mutex);
    return 0;  // 新消息
}

int process_message_idempotent(message_dedup_t *dedup, uint64_t msg_id,
                               const void *data, size_t len) {
    if (data == NULL || len == 0) {
        return ERR_INVALID_PARAM;
    }
    
    // 计算数据hash
    char hash[32];
    calculate_hash(data, len, hash);
    
    // 检查是否重复
    int ret = is_duplicate_message(dedup, msg_id, hash);
    if (ret == 1) {
        log_info("duplicate message ignored: id=%llu", 
                 (unsigned long long)msg_id);
        return SUCCESS;  // 已处理过，返回成功
    }
    if (ret < 0) {
        return ERR_INVALID_MESSAGE;
    }
    
    // 处理新消息
    return process_data(data, len);
}
```

### 2.2 状态定时同步

#### 规则2.2.1：心跳机制
实现心跳检测，及时发现通信故障。

```c
typedef struct {
    int fd;
    time_t last_heartbeat_sent;
    time_t last_heartbeat_received;
    int heartbeat_interval_sec;
    int heartbeat_timeout_sec;
    pthread_mutex_t mutex;
    volatile int running;
    pthread_t heartbeat_thread;
} heartbeat_conn_t;

void *heartbeat_thread_func(void *arg) {
    heartbeat_conn_t *conn = (heartbeat_conn_t *)arg;
    
    while (conn->running) {
        sleep(1);
        
        pthread_mutex_lock(&conn->mutex);
        
        time_t now = time(NULL);
        
        // 检查是否超时
        if (now - conn->last_heartbeat_received > conn->heartbeat_timeout_sec) {
            log_error("heartbeat timeout, connection lost");
            // 触发重连逻辑
            handle_connection_lost(conn);
            pthread_mutex_unlock(&conn->mutex);
            break;
        }
        
        // 发送心跳
        if (now - conn->last_heartbeat_sent >= conn->heartbeat_interval_sec) {
            int ret = send_heartbeat(conn);
            if (ret == SUCCESS) {
                conn->last_heartbeat_sent = now;
            } else {
                log_warn("send heartbeat failed: %d", ret);
            }
        }
        
        pthread_mutex_unlock(&conn->mutex);
    }
    
    return NULL;
}

int heartbeat_conn_init(heartbeat_conn_t *conn, int fd) {
    if (conn == NULL || fd < 0) {
        return ERR_INVALID_PARAM;
    }
    
    memset(conn, 0, sizeof(*conn));
    conn->fd = fd;
    conn->heartbeat_interval_sec = 5;
    conn->heartbeat_timeout_sec = 30;
    conn->last_heartbeat_received = time(NULL);
    conn->running = 1;
    
    if (pthread_mutex_init(&conn->mutex, NULL) != 0) {
        return ERR_SYSTEM_CALL;
    }
    
    if (pthread_create(&conn->heartbeat_thread, NULL, 
                       heartbeat_thread_func, conn) != 0) {
        pthread_mutex_destroy(&conn->mutex);
        return ERR_SYSTEM_CALL;
    }
    
    return SUCCESS;
}

void heartbeat_conn_destroy(heartbeat_conn_t *conn) {
    if (conn == NULL) {
        return;
    }
    
    conn->running = 0;
    pthread_join(conn->heartbeat_thread, NULL);
    pthread_mutex_destroy(&conn->mutex);
}
```

## 三、子系统内部可靠性

### 3.1 防御式编程

#### 规则3.1.1：NULL指针检查
所有指针使用前必须进行NULL检查。

```c
// 错误示例：未检查NULL
void print_user(user_t *user) {
    printf("Name: %s\n", user->name);  // 若user为NULL将崩溃
}

// 正确示例：完整的NULL检查
int print_user_safe(const user_t *user) {
    if (user == NULL) {
        log_error("print_user: user is NULL");
        return ERR_INVALID_PARAM;
    }
    
    printf("Name: %s\n", user->name ? user->name : "(null)");
    printf("Age: %d\n", user->age);
    return SUCCESS;
}

// 函数指针也要检查
typedef int (*callback_t)(void *ctx, int event);

int invoke_callback(callback_t cb, void *ctx, int event) {
    if (cb == NULL) {
        log_warn("callback is NULL, event=%d ignored", event);
        return ERR_INVALID_PARAM;
    }
    
    return cb(ctx, event);
}
```

#### 规则3.1.2：返回值检查
所有可能失败的函数调用都必须检查返回值。

```c
// 错误示例：忽略返回值
void read_config(const char *path) {
    FILE *fp = fopen(path, "r");
    char line[256];
    while (fgets(line, sizeof(line), fp)) {  // 若fp为NULL将崩溃
        // 处理配置
    }
    fclose(fp);  // 若fp为NULL将崩溃
}

// 正确示例：完整的错误处理
int read_config_safe(const char *path, config_t *config) {
    if (path == NULL || config == NULL) {
        return ERR_INVALID_PARAM;
    }
    
    FILE *fp = fopen(path, "r");
    if (fp == NULL) {
        log_error("failed to open config file: %s, error=%s", 
                  path, strerror(errno));
        return ERR_FILE_OPEN;
    }
    
    char line[256];
    int line_num = 0;
    int ret = SUCCESS;
    
    while (fgets(line, sizeof(line), fp) != NULL) {
        line_num++;
        
        // 解析配置行
        ret = parse_config_line(line, config);
        if (ret != SUCCESS) {
            log_error("parse config failed at line %d: %d", line_num, ret);
            break;
        }
    }
    
    if (ferror(fp)) {
        log_error("read config file error: %s", strerror(errno));
        ret = ERR_FILE_READ;
    }
    
    fclose(fp);
    return ret;
}
```

#### 规则3.1.3：errno错误处理
正确使用errno进行错误处理。

```c
// 错误示例：未保存errno
int write_data(int fd, const void *data, size_t len) {
    ssize_t ret = write(fd, data, len);
    if (ret < 0) {
        log_error("write failed: %s", strerror(errno));  // errno可能被log_error改变
        return -1;
    }
    return 0;
}

// 正确示例：立即保存errno
int write_data_safe(int fd, const void *data, size_t len) {
    if (data == NULL || len == 0) {
        errno = EINVAL;
        return ERR_INVALID_PARAM;
    }
    
    size_t written = 0;
    while (written < len) {
        ssize_t ret = write(fd, (const char *)data + written, len - written);
        if (ret < 0) {
            int saved_errno = errno;  // 立即保存
            if (saved_errno == EINTR) {
                continue;  // 被信号中断，重试
            }
            if (saved_errno == EAGAIN || saved_errno == EWOULDBLOCK) {
                // 非阻塞socket，稍后重试
                usleep(1000);
                continue;
            }
            log_error("write failed: fd=%d, error=%s", fd, strerror(saved_errno));
            return ERR_SYSTEM_CALL;
        }
        written += ret;
    }
    
    return SUCCESS;
}

// 使用错误码转换
int errno_to_error_code(int err) {
    switch (err) {
        case 0:         return SUCCESS;
        case EINVAL:    return ERR_INVALID_PARAM;
        case ENOMEM:    return ERR_NO_MEMORY;
        case ENOENT:    return ERR_NOT_FOUND;
        case EACCES:    return ERR_PERMISSION;
        case EEXIST:    return ERR_ALREADY_EXISTS;
        case ETIMEDOUT: return ERR_TIMEOUT;
        default:        return ERR_SYSTEM_CALL;
    }
}
```

### 3.2 资源管理

#### 规则3.2.1：malloc/free配对与goto清理
使用goto进行统一的资源清理，确保资源不泄漏。

```c
// 错误示例：多处返回导致资源泄漏
int process_file_bad(const char *path) {
    char *buffer = malloc(BUFFER_SIZE);
    FILE *fp = fopen(path, "r");
    
    if (fp == NULL) {
        return -1;  // buffer泄漏！
    }
    
    if (fread(buffer, 1, BUFFER_SIZE, fp) == 0) {
        fclose(fp);
        return -2;  // buffer泄漏！
    }
    
    // 处理数据...
    if (error_condition) {
        fclose(fp);
        return -3;  // buffer泄漏！
    }
    
    fclose(fp);
    free(buffer);
    return 0;
}

// 正确示例：使用goto统一清理
int process_file_good(const char *path) {
    int ret = SUCCESS;
    char *buffer = NULL;
    FILE *fp = NULL;
    data_t *data = NULL;
    
    if (path == NULL) {
        ret = ERR_INVALID_PARAM;
        goto cleanup;
    }
    
    buffer = malloc(BUFFER_SIZE);
    if (buffer == NULL) {
        log_error("malloc failed");
        ret = ERR_NO_MEMORY;
        goto cleanup;
    }
    
    fp = fopen(path, "r");
    if (fp == NULL) {
        log_error("fopen failed: %s", path);
        ret = ERR_FILE_OPEN;
        goto cleanup;
    }
    
    size_t bytes_read = fread(buffer, 1, BUFFER_SIZE, fp);
    if (bytes_read == 0) {
        if (ferror(fp)) {
            log_error("fread failed");
            ret = ERR_FILE_READ;
            goto cleanup;
        }
    }
    
    data = parse_data(buffer, bytes_read);
    if (data == NULL) {
        log_error("parse_data failed");
        ret = ERR_PARSE;
        goto cleanup;
    }
    
    ret = process_data(data);
    
cleanup:
    if (data != NULL) {
        free_data(data);
    }
    if (fp != NULL) {
        fclose(fp);
    }
    if (buffer != NULL) {
        free(buffer);
    }
    return ret;
}
```

#### 规则3.2.2：文件句柄配对管理
确保所有文件句柄都正确关闭。

```c
// 正确示例：多文件处理的资源管理
int merge_files(const char **paths, int count, const char *output_path) {
    int ret = SUCCESS;
    FILE **files = NULL;
    FILE *out_fp = NULL;
    char buffer[4096];
    
    if (paths == NULL || count <= 0 || output_path == NULL) {
        ret = ERR_INVALID_PARAM;
        goto cleanup;
    }
    
    files = calloc(count, sizeof(FILE *));
    if (files == NULL) {
        log_error("calloc failed");
        ret = ERR_NO_MEMORY;
        goto cleanup;
    }
    
    // 打开所有输入文件
    for (int i = 0; i < count; i++) {
        files[i] = fopen(paths[i], "r");
        if (files[i] == NULL) {
            log_error("fopen failed: %s", paths[i]);
            ret = ERR_FILE_OPEN;
            goto cleanup;
        }
    }
    
    // 打开输出文件
    out_fp = fopen(output_path, "w");
    if (out_fp == NULL) {
        log_error("fopen failed: %s", output_path);
        ret = ERR_FILE_OPEN;
        goto cleanup;
    }
    
    // 合并文件内容
    for (int i = 0; i < count; i++) {
        size_t bytes;
        while ((bytes = fread(buffer, 1, sizeof(buffer), files[i])) > 0) {
            if (fwrite(buffer, 1, bytes, out_fp) != bytes) {
                log_error("fwrite failed");
                ret = ERR_FILE_WRITE;
                goto cleanup;
            }
        }
        if (ferror(files[i])) {
            log_error("fread failed from file %d", i);
            ret = ERR_FILE_READ;
            goto cleanup;
        }
    }
    
cleanup:
    if (out_fp != NULL) {
        fclose(out_fp);
    }
    if (files != NULL) {
        for (int i = 0; i < count; i++) {
            if (files[i] != NULL) {
                fclose(files[i]);
            }
        }
        free(files);
    }
    return ret;
}
```

#### 规则3.2.3：锁的配对管理
确保所有锁都正确释放，避免死锁。

```c
// 正确示例：使用goto确保锁释放
typedef struct {
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    int data;
    int ready;
} shared_data_t;

int wait_for_data(shared_data_t *shared, int timeout_sec) {
    int ret = SUCCESS;
    int locked = 0;
    
    if (shared == NULL) {
        return ERR_INVALID_PARAM;
    }
    
    if (pthread_mutex_lock(&shared->mutex) != 0) {
        log_error("pthread_mutex_lock failed");
        return ERR_LOCK;
    }
    locked = 1;
    
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    ts.tv_sec += timeout_sec;
    
    while (!shared->ready) {
        int wait_ret = pthread_cond_timedwait(&shared->cond, &shared->mutex, &ts);
        if (wait_ret == ETIMEDOUT) {
            log_warn("wait for data timeout");
            ret = ERR_TIMEOUT;
            goto cleanup;
        }
        if (wait_ret != 0) {
            log_error("pthread_cond_timedwait failed: %d", wait_ret);
            ret = ERR_COND_WAIT;
            goto cleanup;
        }
    }
    
    ret = shared->data;
    
cleanup:
    if (locked) {
        pthread_mutex_unlock(&shared->mutex);
    }
    return ret;
}

// 使用RAII风格宏（仅适用于简单场景）
#define MUTEX_LOCK_GUARD(mutex) \
    pthread_mutex_t *__lock_ptr = mutex; \
    pthread_mutex_lock(__lock_ptr); \
    __attribute__((cleanup(mutex_cleanup))) int __lock_guard = 1

void mutex_cleanup(int *guard) {
    (void)guard;
}

// 但在C中，仍推荐使用显式的goto清理方式
```

### 3.3 核心流程依赖最小化

#### 规则3.3.1：模块解耦设计
核心功能应尽量减少外部依赖，确保基本功能可用。

```c
// 正确示例：分级依赖管理
typedef enum {
    DEP_LEVEL_CRITICAL = 0,    // 核心功能必须
    DEP_LEVEL_IMPORTANT = 1,   // 重要但可降级
    DEP_LEVEL_OPTIONAL = 2     // 可选功能
} dependency_level_t;

typedef struct {
    const char *name;
    dependency_level_t level;
    int (*init_func)(void);
    void (*cleanup_func)(void);
    int (*health_check)(void);
    int is_available;
} module_dep_t;

int init_module_deps(module_dep_t *deps, int count) {
    int critical_failed = 0;
    
    // 先初始化所有依赖
    for (int i = 0; i < count; i++) {
        if (deps[i].init_func != NULL) {
            int ret = deps[i].init_func();
            deps[i].is_available = (ret == SUCCESS);
            
            if (!deps[i].is_available) {
                log_error("module %s init failed: %d", deps[i].name, ret);
                
                if (deps[i].level == DEP_LEVEL_CRITICAL) {
                    critical_failed = 1;
                }
            }
        } else {
            deps[i].is_available = 1;
        }
    }
    
    // 核心依赖失败则返回错误
    if (critical_failed) {
        return ERR_CRITICAL_DEP_FAILED;
    }
    
    return SUCCESS;
}

int call_module_safe(module_dep_t *dep, int (*func)(void *), void *arg) {
    if (dep == NULL || func == NULL) {
        return ERR_INVALID_PARAM;
    }
    
    if (!dep->is_available) {
        log_warn("module %s not available, using fallback", dep->name);
        
        if (dep->level == DEP_LEVEL_CRITICAL) {
            return ERR_MODULE_UNAVAILABLE;
        }
        // 降级处理
        return ERR_DEGRADED;
    }
    
    return func(arg);
}
```

### 3.4 流程异常降级处理

#### 规则3.4.1：多级降级策略
实现多级降级，确保基本功能可用。

```c
typedef enum {
    DEGRADE_LEVEL_FULL = 0,      // 完整功能
    DEGRADE_LEVEL_REDUCED = 1,   // 降级功能（缓存）
    DEGRADE_LEVEL_MINIMAL = 2,   // 最小功能（本地数据）
    DEGRADE_LEVEL_OFFLINE = 3    // 离线模式
} degrade_level_t;

typedef struct {
    degrade_level_t current_level;
    time_t last_success_time;
    int consecutive_failures;
    int threshold_for_degrade;
    int threshold_for_recover;
} degrade_controller_t;

int update_degrade_level(degrade_controller_t *ctrl, int success) {
    if (ctrl == NULL) {
        return ERR_INVALID_PARAM;
    }
    
    time_t now = time(NULL);
    
    if (success) {
        ctrl->last_success_time = now;
        ctrl->consecutive_failures = 0;
        
        // 尝试恢复到更高级别
        if (ctrl->current_level > DEGRADE_LEVEL_FULL) {
            ctrl->current_level--;
            log_info("service recovered to level %d", ctrl->current_level);
        }
    } else {
        ctrl->consecutive_failures++;
        
        // 连续失败达到阈值，降级
        if (ctrl->consecutive_failures >= ctrl->threshold_for_degrade) {
            if (ctrl->current_level < DEGRADE_LEVEL_OFFLINE) {
                ctrl->current_level++;
                log_warn("service degraded to level %d", ctrl->current_level);
            }
        }
    }
    
    return SUCCESS;
}

int execute_with_degrade(degrade_controller_t *ctrl, 
                         int (*full_func)(void *),
                         int (*reduced_func)(void *),
                         int (*minimal_func)(void *),
                         void *arg) {
    if (ctrl == NULL || arg == NULL) {
        return ERR_INVALID_PARAM;
    }
    
    int ret;
    
    switch (ctrl->current_level) {
        case DEGRADE_LEVEL_FULL:
            if (full_func != NULL) {
                ret = full_func(arg);
                update_degrade_level(ctrl, ret == SUCCESS);
                if (ret == SUCCESS) return ret;
            }
            // 失败后降级尝试
            // fall through
            
        case DEGRADE_LEVEL_REDUCED:
            if (reduced_func != NULL) {
                log_info("using reduced mode");
                return reduced_func(arg);
            }
            // fall through
            
        case DEGRADE_LEVEL_MINIMAL:
            if (minimal_func != NULL) {
                log_info("using minimal mode");
                return minimal_func(arg);
            }
            // fall through
            
        case DEGRADE_LEVEL_OFFLINE:
            log_warn("service in offline mode");
            return ERR_SERVICE_UNAVAILABLE;
            
        default:
            return ERR_INVALID_STATE;
    }
}
```

### 3.5 子系统崩溃处理

#### 规则3.5.1：子进程监控
监控子进程状态，自动恢复崩溃的进程。

```c
typedef struct {
    pid_t pid;
    char *path;
    char **args;
    int restart_count;
    int max_restarts;
    time_t start_time;
    time_t last_restart_time;
    int min_uptime_sec;  // 最小运行时间，防止频繁重启
} subprocess_t;

int subprocess_start(subprocess_t *proc) {
    if (proc == NULL || proc->path == NULL) {
        return ERR_INVALID_PARAM;
    }
    
    // 检查重启频率
    time_t now = time(NULL);
    if (proc->last_restart_time > 0 && 
        now - proc->last_restart_time < proc->min_uptime_sec) {
        log_error("subprocess restarting too frequently, stopping");
        return ERR_RESTART_LIMIT;
    }
    
    pid_t pid = fork();
    if (pid < 0) {
        log_error("fork failed: %s", strerror(errno));
        return ERR_SYSTEM_CALL;
    }
    
    if (pid == 0) {
        // 子进程
        execv(proc->path, proc->args);
        log_error("execv failed: %s", strerror(errno));
        exit(1);
    }
    
    // 父进程
    proc->pid = pid;
    proc->start_time = time(NULL);
    proc->last_restart_time = proc->start_time;
    
    log_info("subprocess started: pid=%d, path=%s", pid, proc->path);
    return SUCCESS;
}

int subprocess_monitor(subprocess_t *proc) {
    if (proc == NULL || proc->pid <= 0) {
        return ERR_INVALID_PARAM;
    }
    
    int status;
    pid_t result = waitpid(proc->pid, &status, WNOHANG);
    
    if (result == 0) {
        // 子进程仍在运行
        return SUCCESS;
    }
    
    if (result < 0) {
        log_error("waitpid failed: %s", strerror(errno));
        return ERR_SYSTEM_CALL;
    }
    
    // 子进程已退出
    if (WIFEXITED(status)) {
        log_warn("subprocess exited with code %d", WEXITSTATUS(status));
    } else if (WIFSIGNALED(status)) {
        log_warn("subprocess killed by signal %d", WTERMSIG(status));
    }
    
    proc->pid = 0;
    proc->restart_count++;
    
    // 检查重启次数
    if (proc->restart_count > proc->max_restarts) {
        log_error("subprocess restart limit reached: %d > %d",
                  proc->restart_count, proc->max_restarts);
        return ERR_RESTART_LIMIT;
    }
    
    // 自动重启
    log_info("restarting subprocess (attempt %d/%d)",
             proc->restart_count, proc->max_restarts);
    return subprocess_start(proc);
}
```

#### 规则3.5.2：信号处理
正确处理信号，确保资源清理。

```c
volatile sig_atomic_t g_terminated = 0;
volatile sig_atomic_t g_child_exited = 0;

void signal_handler(int sig) {
    switch (sig) {
        case SIGTERM:
        case SIGINT:
            g_terminated = 1;
            break;
        case SIGCHLD:
            g_child_exited = 1;
            break;
        default:
            break;
    }
}

int setup_signal_handlers(void) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = signal_handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    
    if (sigaction(SIGTERM, &sa, NULL) < 0) {
        log_error("sigaction(SIGTERM) failed");
        return ERR_SYSTEM_CALL;
    }
    if (sigaction(SIGINT, &sa, NULL) < 0) {
        log_error("sigaction(SIGINT) failed");
        return ERR_SYSTEM_CALL;
    }
    if (sigaction(SIGCHLD, &sa, NULL) < 0) {
        log_error("sigaction(SIGCHLD) failed");
        return ERR_SYSTEM_CALL;
    }
    
    // 忽略SIGPIPE
    sa.sa_handler = SIG_IGN;
    if (sigaction(SIGPIPE, &sa, NULL) < 0) {
        log_error("sigaction(SIGPIPE) failed");
        return ERR_SYSTEM_CALL;
    }
    
    return SUCCESS;
}

int main_loop(void) {
    int ret = setup_signal_handlers();
    if (ret != SUCCESS) {
        return ret;
    }
    
    while (!g_terminated) {
        // 处理子进程退出
        if (g_child_exited) {
            g_child_exited = 0;
            // 处理所有已退出的子进程
            while (waitpid(-1, NULL, WNOHANG) > 0);
        }
        
        // 主循环逻辑
        ret = process_events();
        if (ret != SUCCESS) {
            log_error("process_events failed: %d", ret);
        }
    }
    
    log_info("graceful shutdown initiated");
    return cleanup_resources();
}
```

## 四、工具使用指导

### 4.1 Valgrind内存检测

```bash
# 检测内存泄漏
valgrind --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         --verbose \
         ./program

# 检测未初始化值使用
valgrind --track-origins=yes ./program

# 检测线程错误
valgrind --tool=helgrind ./program

# 检测缓存性能
valgrind --tool=cachegrind ./program

# 抑制已知的误报
valgrind --suppressions=suppressions.supp ./program
```

常见问题检测：
- **内存泄漏**：确认所有malloc/free配对
- **非法读写**：检查数组越界、野指针
- **未初始化值**：检查变量初始化
- **重复释放**：检查free调用路径

### 4.2 AddressSanitizer

```bash
# 编译时启用ASan
gcc -fsanitize=address -fno-omit-frame-pointer -g -O1 -o program program.c

# 运行时配置
export ASAN_OPTIONS=detect_leaks=1:abort_on_error=1:log_path=asan.log
./program

# 检测栈/全局变量溢出
gcc -fsanitize=address -fsanitize-recover=address -o program program.c
```

ASan检测的问题类型：
- 堆缓冲区溢出
- 栈缓冲区溢出
- 全局缓冲区溢出
- Use after free
- Double free
- Memory leaks

### 4.3 静态分析工具

```bash
# 使用cppcheck
cppcheck --enable=all --inconclusive --std=c11 src/

# 使用clang-tidy
clang-tidy src/*.c -- -std=c11

# 使用splint
splint +posixlib src/*.c
```

### 4.4 自定义检测宏

```c
// 调试辅助宏
#ifdef DEBUG
#define CHECK_NULL(ptr) \
    do { \
        if ((ptr) == NULL) { \
            fprintf(stderr, "%s:%d: NULL pointer: %s\n", \
                    __FILE__, __LINE__, #ptr); \
            abort(); \
        } \
    } while(0)

#define CHECK_ALLOC(ptr) \
    do { \
        if ((ptr) == NULL) { \
            fprintf(stderr, "%s:%d: Allocation failed for %s\n", \
                    __FILE__, __LINE__, #ptr); \
            abort(); \
        } \
    } while(0)
#else
#define CHECK_NULL(ptr) ((void)0)
#define CHECK_ALLOC(ptr) ((void)0)
#endif

// 内存分配包装器
static int g_total_allocations = 0;

void *debug_malloc(size_t size, const char *file, int line) {
    void *ptr = malloc(size);
    if (ptr != NULL) {
        g_total_allocations++;
        fprintf(stderr, "ALLOC %p (%zu bytes) at %s:%d, total=%d\n",
                ptr, size, file, line, g_total_allocations);
    } else {
        fprintf(stderr, "ALLOC FAILED (%zu bytes) at %s:%d\n", size, file, line);
    }
    return ptr;
}

void debug_free(void *ptr, const char *file, int line) {
    if (ptr != NULL) {
        g_total_allocations--;
        fprintf(stderr, "FREE %p at %s:%d, remaining=%d\n",
                ptr, file, line, g_total_allocations);
    }
    free(ptr);
}

#ifdef DEBUG_MEMORY
#define malloc(size) debug_malloc(size, __FILE__, __LINE__)
#define free(ptr) debug_free(ptr, __FILE__, __LINE__)
#endif
```

## 五、可靠性检查清单

### 5.1 输入验证检查清单

- [ ] 所有公共函数都检查NULL指针参数
- [ ] 数值参数检查范围和有效性
- [ ] 字符串参数检查长度和格式
- [ ] 缓冲区操作前检查边界
- [ ] 文件路径参数检查合法性
- [ ] 数组索引检查边界

### 5.2 资源管理检查清单

- [ ] 所有malloc都有对应的free
- [ ] 所有fopen都有对应的fclose
- [ ] 所有socket都有对应的close
- [ ] 所有pthread_mutex_lock都有对应的unlock
- [ ] 使用goto实现统一清理路径
- [ ] 清理函数可以安全处理NULL
- [ ] 所有分配的内存都被正确初始化

### 5.3 错误处理检查清单

- [ ] 所有可能失败的系统调用都检查返回值
- [ ] errno在使用前立即保存
- [ ] 错误信息包含足够的上下文
- [ ] 错误码定义清晰，有文档说明
- [ ] 函数返回前释放所有临时资源
- [ ] 错误传播到调用者

### 5.4 并发安全检查清单

- [ ] 共享变量访问都有适当的锁保护
- [ ] 锁的获取顺序一致，避免死锁
- [ ] 锁持有时间最小化
- [ ] 使用原子操作处理简单计数器
- [ ] 条件变量配合while循环使用
- [ ] 线程清理函数正确注册

### 5.5 容错机制检查清单

- [ ] 关键操作有重试机制
- [ ] 重试有合理的退避策略
- [ ] 有超时控制机制
- [ ] 有资源配额限制
- [ ] 有降级处理策略
- [ ] 异常状态可恢复

### 5.6 代码质量检查清单

- [ ] 通过编译无警告（-Wall -Wextra）
- [ ] 通过静态分析（cppcheck/clang-tidy）
- [ ] 通过动态检测（valgrind/ASan）
- [ ] 无内存泄漏
- [ ] 无未定义行为
- [ ] 无竞态条件

## 六、常见错误模式

### 6.1 资源泄漏模式

```c
// 错误：提前返回导致泄漏
if (error) {
    return -1;  // buffer未释放
}

// 正确：goto统一清理
if (error) {
    ret = ERR_xxx;
    goto cleanup;
}
```

### 6.2 双重释放模式

```c
// 错误：可能导致双重释放
free(ptr);
if (condition) {
    free(ptr);  // 错误！
}

// 正确：释放后置NULL
free(ptr);
ptr = NULL;  // 防止双重释放

// 或在清理时检查
if (ptr != NULL) {
    free(ptr);
    ptr = NULL;
}
```

### 6.3 使用后释放模式

```c
// 错误：使用已释放的内存
free(ptr);
process(ptr->data);  // 错误！

// 正确：先使用，后释放
process(ptr->data);
free(ptr);
ptr = NULL;
```

### 6.4 返回栈地址模式

```c
// 错误：返回局部变量地址
char *get_name(void) {
    char name[100];
    strcpy(name, "test");
    return name;  // 错误！返回栈地址
}

// 正确：返回堆分配或静态存储
char *get_name(void) {
    char *name = malloc(100);
    if (name != NULL) {
        strcpy(name, "test");
    }
    return name;  // 调用者负责free
}

// 或使用静态存储
const char *get_name_static(void) {
    static char name[100];
    strcpy(name, "test");
    return name;  // 注意：非线程安全
}
```
