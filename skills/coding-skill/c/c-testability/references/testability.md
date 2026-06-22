# C语言可测试性规范详细参考

## 1. 函数单一职责

### 原则
每个函数只做一件事，便于独立测试和验证。

### 规范
- 函数长度控制在50行以内
- 函数参数不超过4个
- 嵌套层级不超过3层
- 单一入口、单一出口（避免多处return）

### 反例
```c
void process_user_data(User* user, Database* db, Logger* log) {
    if (user == NULL) return;
    
    // 验证用户数据
    if (strlen(user->name) == 0) {
        log->error("Invalid user name");
        return;
    }
    
    // 计算用户得分
    int score = 0;
    for (int i = 0; i < user->activity_count; i++) {
        score += user->activities[i].points;
    }
    
    // 保存到数据库
    db->save(user->id, score);
    
    // 发送通知
    send_notification(user->email, score);
}
```

### 正例
```c
bool validate_user(const User* user) {
    return user != NULL && strlen(user->name) > 0;
}

int calculate_user_score(const User* user) {
    int score = 0;
    for (int i = 0; i < user->activity_count; i++) {
        score += user->activities[i].points;
    }
    return score;
}

bool save_user_score(Database* db, int user_id, int score) {
    return db->save(user_id, score);
}

bool process_user_data(User* user, Database* db, Logger* log) {
    if (!validate_user(user)) {
        log->error("Invalid user");
        return false;
    }
    
    int score = calculate_user_score(user);
    
    if (!save_user_score(db, user->id, score)) {
        log->error("Failed to save score");
        return false;
    }
    
    return true;
}
```

---

## 2. 依赖注入

### 原则
通过参数传入依赖，而非在函数内部创建或使用全局变量。

### 规范
- 所有外部依赖通过参数传入
- 使用结构体封装相关依赖
- 避免在函数内调用malloc/fopen等资源获取函数

### 反例
```c
int read_config_value(const char* key) {
    FILE* file = fopen("config.txt", "r");  // 硬编码依赖
    if (file == NULL) return -1;
    
    int value;
    // 解析配置...
    fclose(file);
    return value;
}
```

### 正例
```c
typedef struct {
    FILE* (*open)(const char* path, const char* mode);
    int (*read)(FILE* file, char* buffer, int size);
    int (*close)(FILE* file);
} FileOps;

int read_config_value(const char* key, const char* path, FileOps* ops) {
    FILE* file = ops->open(path, "r");
    if (file == NULL) return -1;
    
    int value;
    // 解析配置...
    ops->close(file);
    return value;
}

// 生产环境
FileOps real_file_ops = { fopen, fread, fclose };

// 测试环境
FILE* mock_fopen(const char* path, const char* mode) { return (FILE*)0x1234; }
int mock_fread(FILE* file, char* buffer, int size) { 
    strcpy(buffer, "key=value");
    return 9;
}
int mock_fclose(FILE* file) { return 0; }

FileOps mock_file_ops = { mock_fopen, mock_fread, mock_fclose };
```

---

## 3. 避免全局状态

### 原则
全局状态导致测试相互影响，难以隔离和重现。

### 规范
- 禁止使用全局变量存储状态
- 使用上下文结构体传递状态
- 必须使用全局变量时，提供重置函数

### 反例
```c
static int counter = 0;  // 全局状态

int increment_counter(void) {
    return ++counter;
}

int get_counter(void) {
    return counter;
}
```

### 正例
```c
typedef struct {
    int counter;
} CounterContext;

void counter_init(CounterContext* ctx) {
    ctx->counter = 0;
}

int increment_counter(CounterContext* ctx) {
    return ++ctx->counter;
}

int get_counter(const CounterContext* ctx) {
    return ctx->counter;
}

// 测试代码
void test_counter(void) {
    CounterContext ctx;
    counter_init(&ctx);
    
    TEST_ASSERT_EQUAL_INT(1, increment_counter(&ctx));
    TEST_ASSERT_EQUAL_INT(2, increment_counter(&ctx));
}
```

---

## 4. 使用函数指针实现Mock

### 原则
函数指针允许在运行时替换实现，便于测试。

### 规范
- 对外部依赖定义函数指针类型
- 提供默认实现和测试替身
- 结构体中组织相关函数指针

### 示例
```c
// 定义依赖接口
typedef struct {
    int (*send)(const char* data, int len);
    int (*receive)(char* buffer, int max_len);
} NetworkInterface;

// 生产实现
int real_send(const char* data, int len) {
    // 实际网络发送...
    return len;
}

int real_receive(char* buffer, int max_len) {
    // 实际网络接收...
    return 0;
}

NetworkInterface real_network = { real_send, real_receive };

// Mock实现
static char mock_send_buffer[256];
static int mock_send_count = 0;

int mock_send(const char* data, int len) {
    strncpy(mock_send_buffer, data, len);
    mock_send_count++;
    return len;
}

int mock_receive(char* buffer, int max_len) {
    const char* response = "OK";
    strcpy(buffer, response);
    return strlen(response);
}

NetworkInterface mock_network = { mock_send, mock_receive };

// 使用依赖的业务代码
typedef struct {
    NetworkInterface* net;
} HttpClient;

void http_client_init(HttpClient* client, NetworkInterface* net) {
    client->net = net;
}

int http_post(HttpClient* client, const char* data) {
    return client->net->send(data, strlen(data));
}

// 测试代码
void test_http_post(void) {
    HttpClient client;
    http_client_init(&client, &mock_network);
    
    int result = http_post(&client, "test data");
    
    TEST_ASSERT_EQUAL_INT(9, result);
    TEST_ASSERT_EQUAL_STRING("test data", mock_send_buffer);
    TEST_ASSERT_EQUAL_INT(1, mock_send_count);
}
```

---

## 5. 分离纯函数和IO函数

### 原则
纯函数无副作用，易于测试；IO函数需要隔离和mock。

### 规范
- 纯函数：相同输入总是产生相同输出，无副作用
- IO函数：涉及文件、网络、数据库、时间等外部资源
- 业务逻辑放入纯函数，IO操作单独封装

### 纯函数示例
```c
int calculate_discount(int price, int discount_percent) {
    if (discount_percent < 0 || discount_percent > 100) {
        return price;
    }
    return price * (100 - discount_percent) / 100;
}

// 测试简单直接
void test_calculate_discount(void) {
    TEST_ASSERT_EQUAL_INT(90, calculate_discount(100, 10));
    TEST_ASSERT_EQUAL_INT(50, calculate_discount(100, 50));
    TEST_ASSERT_EQUAL_INT(100, calculate_discount(100, -1));  // 边界
    TEST_ASSERT_EQUAL_INT(100, calculate_discount(100, 101)); // 边界
}
```

### IO函数封装
```c
typedef struct {
    int (*read_price)(int product_id);
    void (*write_price)(int product_id, int price);
} PriceStorage;

int update_price_with_discount(PriceStorage* storage, int product_id, int discount) {
    int original_price = storage->read_price(product_id);
    int new_price = calculate_discount(original_price, discount);
    storage->write_price(product_id, new_price);
    return new_price;
}
```

---

## 6. 单元测试框架

### Unity框架
```c
#include "unity.h"

int add(int a, int b) {
    return a + b;
}

void setUp(void) {
    // 每个测试前执行
}

void tearDown(void) {
    // 每个测试后执行
}

void test_add_positive_numbers(void) {
    TEST_ASSERT_EQUAL_INT(5, add(2, 3));
}

void test_add_negative_numbers(void) {
    TEST_ASSERT_EQUAL_INT(-5, add(-2, -3));
}

void test_add_zero(void) {
    TEST_ASSERT_EQUAL_INT(3, add(3, 0));
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_add_positive_numbers);
    RUN_TEST(test_add_negative_numbers);
    RUN_TEST(test_add_zero);
    return UNITY_END();
}
```

### CMock使用
```c
// header.h
int external_dependency(int value);

// source.c
#include "header.h"
int process(int input) {
    return external_dependency(input) * 2;
}

// test.c
#include "unity.h"
#include "cmock.h"
#include "header.h"

void test_process_calls_dependency(void) {
    external_dependency_ExpectAndReturn(5, 10);
    TEST_ASSERT_EQUAL_INT(20, process(5));
}
```

---

## 7. 测试替身模式

### Stub（桩）
返回固定值，用于隔离依赖。
```c
int stub_database_query(const char* sql) {
    return 42;  // 固定返回值
}
```

### Mock（模拟对象）
验证调用行为。
```c
typedef struct {
    const char* last_sql;
    int call_count;
} MockDatabase;

int mock_database_query(MockDatabase* mock, const char* sql) {
    mock->last_sql = sql;
    mock->call_count++;
    return 42;
}

void test_query_called(void) {
    MockDatabase mock = {0};
    
    mock_database_query(&mock, "SELECT * FROM users");
    
    TEST_ASSERT_EQUAL_STRING("SELECT * FROM users", mock.last_sql);
    TEST_ASSERT_EQUAL_INT(1, mock.call_count);
}
```

### Fake（伪实现）
简化的功能实现。
```c
#define FAKE_DB_SIZE 10
typedef struct {
    int keys[FAKE_DB_SIZE];
    int values[FAKE_DB_SIZE];
    int count;
} FakeDatabase;

void fake_db_init(FakeDatabase* db) {
    db->count = 0;
}

void fake_db_insert(FakeDatabase* db, int key, int value) {
    if (db->count < FAKE_DB_SIZE) {
        db->keys[db->count] = key;
        db->values[db->count] = value;
        db->count++;
    }
}

int fake_db_get(FakeDatabase* db, int key) {
    for (int i = 0; i < db->count; i++) {
        if (db->keys[i] == key) {
            return db->values[i];
        }
    }
    return -1;
}
```

### Spy（间谍）
记录调用信息。
```c
typedef struct {
    int send_call_count;
    const char* last_data;
} NetworkSpy;

int spy_send(NetworkSpy* spy, const char* data, int len) {
    spy->send_call_count++;
    spy->last_data = data;
    return len;
}
```

---

## 8. 可测试的代码结构

### 目录结构
```
project/
├── src/
│   ├── module.c
│   └── module.h
├── test/
│   ├── test_module.c
│   ├── mocks/
│   │   └── mock_dependency.c
│   └── fixtures/
│       └── test_data.txt
├── unity/
│   └── unity.c
└── Makefile
```

### 头文件设计
```c
// module.h
#ifndef MODULE_H
#define MODULE_H

// 公开接口
typedef struct Module Module;

Module* module_create(void);
void module_destroy(Module* m);
int module_process(Module* m, int input);

// 可测试接口（仅测试文件包含）
#ifdef TESTABLE
typedef struct {
    int (*process_internal)(int value);
} ModuleInternal;
ModuleInternal* module_get_internal(Module* m);
#endif

#endif
```

### 测试Makefile示例
```makefile
CC = gcc
CFLAGS = -Wall -Wextra -I./src -I./unity

SRC = src/module.c
TEST_SRC = test/test_module.c unity/unity.c
MOCK_SRC = test/mocks/mock_dependency.c

test: $(SRC) $(TEST_SRC) $(MOCK_SRC)
	$(CC) $(CFLAGS) -DTESTABLE -o test_runner $^
	./test_runner

coverage: $(SRC) $(TEST_SRC) $(MOCK_SRC)
	$(CC) $(CFLAGS) -DTESTABLE --coverage -o test_runner $^
	./test_runner
	gcov src/module.c
```

---

## 9. 可测试性检查清单

### 函数设计
- [ ] 函数长度是否小于50行？
- [ ] 函数参数是否不超过4个？
- [ ] 函数是否有明确的单一职责？
- [ ] 嵌套层级是否不超过3层？
- [ ] 函数是否有清晰的成功/失败返回值？

### 依赖管理
- [ ] 是否避免了在函数内部创建依赖？
- [ ] 外部依赖是否通过参数注入？
- [ ] 是否定义了清晰的接口类型（函数指针结构体）？
- [ ] 是否提供了默认实现和测试替身？

### 全局状态
- [ ] 是否避免了全局变量？
- [ ] 状态是否封装在上下文结构体中？
- [ ] 是否提供了初始化和重置函数？

### 代码分离
- [ ] 是否分离了纯函数和IO函数？
- [ ] 业务逻辑是否集中在纯函数中？
- [ ] IO操作是否有独立封装？

### 测试支持
- [ ] 是否提供了测试接口（TESTABLE宏）？
- [ ] 是否有完整的setUp/tearDown？
- [ ] 测试用例是否覆盖正常、边界、异常场景？
- [ ] 测试是否相互独立、可重复执行？

### 文档和命名
- [ ] 测试命名是否描述清楚（test_模块_函数_场景_预期）？
- [ ] 是否有测试覆盖率目标？
- [ ] 关键路径是否达到100%覆盖率？

---

## 10. 常见问题与解决方案

### 问题1：遗留代码难以测试
**解决方案**：创建包装层，逐步重构。
```c
// 包装层
typedef struct {
    int (*legacy_call)(int param);
} LegacyWrapper;

// 新代码通过wrapper调用
int new_feature(LegacyWrapper* wrapper, int value) {
    return wrapper->legacy_call(value);
}
```

### 问题2：静态函数无法测试
**解决方案**：使用条件编译暴露静态函数。
```c
#ifdef TEST
#define STATIC
#else
#define STATIC static
#endif

STATIC int internal_helper(int x) {
    return x * 2;
}
```

### 问题3：硬件相关代码难以测试
**解决方案**：抽象硬件接口。
```c
typedef struct {
    void (*write_pin)(int pin, int value);
    int (*read_pin)(int pin);
} HardwareInterface;

// 测试时使用mock实现
// 生产时使用真实硬件操作
```

### 问题4：时间相关代码难以测试
**解决方案**：注入时间函数。
```c
typedef uint32_t (*TimeProvider)(void);

uint32_t get_timestamp(TimeProvider get_time) {
    return get_time();
}

// 生产
uint32_t real_time(void) { return time(NULL); }

// 测试
static uint32_t mock_time_value = 0;
uint32_t mock_time(void) { return mock_time_value; }
```
