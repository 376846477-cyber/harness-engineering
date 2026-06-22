# C语言可读性规范详细参考

## 一、名称传递信息

### 1.1 核心原则
名称是代码可读性的第一要素。好的名称应该自解释，让读者无需查看注释或文档就能理解其用途。

### 1.2 函数命名规范
```c
// ❌ 不好的命名
void process(int data);
void handle(void);
void do_stuff(char *str);

// ✅ 好的命名
void parse_http_response(int socket_fd);
void handle_connection_timeout(void);
void validate_user_input(char *input_buffer);

// 动词+名词结构
void calculate_total_price(void);
int read_config_file(const char *path);
bool is_valid_email(const char *email);
```

### 1.3 变量命名规范
```c
// ❌ 不好的命名
int x, y, z;
char *p;
int temp;
int flag;

// ✅ 好的命名
int user_age;
int retry_count;
char *file_content_buffer;
bool is_connection_active;
int max_retry_attempts;

// 带单位的命名
int timeout_ms;           // 毫秒
size_t buffer_size_bytes; // 字节
int connection_timeout_sec; // 秒

// 布尔变量命名
bool is_valid;
bool has_permission;
bool can_write;
bool should_retry;
```

### 1.4 命名禁忌
```c
// ❌ 避免无意义名称
int data;           // 太泛化
int value;          // 无信息量
char *ptr;          // 仅说明类型
int temp;           // 临时什么？
int flag;           // 什么标志？

// ❌ 避免误导性名称
int account_list;   // 实际是单个账户，不是列表
char *user_name;    // 实际存储的是ID

// ❌ 避免缩写（除非是通用缩写）
int usr_cnt;        // 用 user_count
int calc_prc();     // 用 calculate_price
int buf_sz;         // 用 buffer_size

// ✅ 可接受的通用缩写
int http_status;
char *html_content;
int cpu_usage;
size_t strlen_result;
```

## 二、格式传递信息

### 2.1 缩进规范
```c
// 使用4空格缩进（或团队统一的Tab）
void process_user_data(user_t *user) {
    if (user == NULL) {
        return;
    }
    
    for (int i = 0; i < user->item_count; i++) {
        process_item(user->items[i]);
    }
}

// switch语句缩进
switch (status_code) {
    case HTTP_OK:
        handle_success();
        break;
    case HTTP_NOT_FOUND:
        handle_not_found();
        break;
    default:
        handle_unknown_status(status_code);
        break;
}
```

### 2.2 空行使用
```c
// ✅ 用空行分隔逻辑块
int calculate_order_total(order_t *order) {
    int subtotal = 0;
    
    // 计算商品小计
    for (int i = 0; i < order->item_count; i++) {
        subtotal += order->items[i].price * order->items[i].quantity;
    }
    
    // 计算折扣
    int discount = 0;
    if (order->has_coupon) {
        discount = calculate_discount(subtotal, order->coupon_code);
    }
    
    // 计算税费
    int tax = subtotal * TAX_RATE / 100;
    
    return subtotal - discount + tax;
}

// ❌ 空行过多或过少
int bad_example(void) {
    int x = 10;
    
    
    
    
    int y = 20;
    return x + y;
}
```

### 2.3 对齐与分组
```c
// ✅ 相关声明分组
static const int MAX_RETRY_COUNT = 3;
static const int RETRY_DELAY_MS = 1000;
static const int TIMEOUT_SEC = 30;

// ✅ 相关变量声明分组
int socket_fd;
int connection_status;
struct sockaddr_in server_addr;

// 结构体成员对齐（可选，看团队规范）
struct user_info {
    int    user_id;
    char   user_name[64];
    char   email[128];
    time_t created_at;
};

// ✅ 初始化对齐
config_t config = {
    .server_port    = 8080,
    .max_connection = 100,
    .timeout_sec    = 30,
    .debug_mode     = false
};
```

### 2.4 长行处理
```c
// ❌ 过长的行
int result = some_function(arg1, arg2, arg3, arg4, arg5, arg6, arg7, arg8);

// ✅ 合理换行
int result = some_function(
    arg1,
    arg2,
    arg3,
    arg4,
    arg5,
    arg6,
    arg7,
    arg8
);

// ✅ 链式调用换行
int status = connect_to_server(server_addr)
              .set_timeout(timeout_sec)
              .enable_ssl()
              .send_request(request_data)
              .get_response();
```

## 三、注释作为补充

### 3.1 注释原则
注释应该解释"为什么"而不是"是什么"。代码本身应该清晰表达做什么。

```c
// ❌ 无价值的注释
i = i + 1;  // i加1

// ❌ 重复代码的注释
// 检查指针是否为空
if (ptr == NULL) {
    return -1;
}

// ✅ 解释意图的注释
// 使用二分查找而非线性查找，因为列表已排序且数据量大
int index = binary_search(sorted_list, target_value);

// ✅ 解释原因的注释
// 重试3次而不是1次，因为网络环境不稳定可能导致临时失败
for (int retry = 0; retry < MAX_RETRY_COUNT; retry++) {
    if (send_packet(packet) == SUCCESS) {
        break;
    }
}

// ✅ TODO和FIXME注释
// TODO: 添加对IPv6的支持
// FIXME: 内存泄漏风险，需要在错误路径释放buffer
```

### 3.2 函数注释
```c
/**
 * @brief 解析HTTP响应并提取状态码和内容
 * 
 * @param response 完整的HTTP响应字符串
 * @param status_code 输出参数，存储解析出的状态码
 * @param content 输出参数，存储解析出的内容体
 * @return 成功返回0，失败返回-1
 * 
 * @note 调用者需确保response以null结尾
 * @warning content指向response内部，不要释放后单独使用
 */
int parse_http_response(const char *response, 
                       int *status_code, 
                       char **content);
```

## 四、函数长度控制

### 4.1 长度建议
- 单个函数建议控制在 **50-100行** 以内
- 超过100行应考虑拆分
- 复杂逻辑即使行数不多也应拆分

### 4.2 函数拆分示例
```c
// ❌ 过长的函数（100+行）
void process_user_request(request_t *request) {
    // 20行：验证输入
    // ...
    // 30行：查询数据库
    // ...
    // 25行：业务逻辑处理
    // ...
    // 30行：生成响应
    // ...
}

// ✅ 拆分后的函数
void process_user_request(request_t *request) {
    if (!validate_request_input(request)) {
        send_error_response(ERROR_INVALID_INPUT);
        return;
    }
    
    user_t *user = query_user_from_db(request->user_id);
    if (user == NULL) {
        send_error_response(ERROR_USER_NOT_FOUND);
        return;
    }
    
    process_result_t result = execute_business_logic(user, request);
    send_success_response(&result);
}
```

## 五、单一职责原则

### 5.1 核心概念
每个函数应该只做一件事，并把它做好。

```c
// ❌ 违反单一职责
void process_and_save_user(char *name, int age) {
    // 验证
    if (strlen(name) > MAX_NAME_LEN || age < 0) {
        return;
    }
    
    // 处理
    user_t user;
    strcpy(user.name, name);
    user.age = age;
    user.created_at = time(NULL);
    
    // 保存到文件
    FILE *fp = fopen("users.txt", "a");
    fprintf(fp, "%s,%d,%ld\n", user.name, user.age, user.created_at);
    fclose(fp);
    
    // 发送通知
    send_email(user.name);
    log_operation("user saved");
}

// ✅ 单一职责
bool validate_user_data(const char *name, int age);
user_t create_user(const char *name, int age);
bool save_user_to_file(const user_t *user);
void notify_user_created(const user_t *user);
```

## 六、减少嵌套层级

### 6.1 使用早返回（卫语句）
```c
// ❌ 深层嵌套
int process_data(data_t *data) {
    if (data != NULL) {
        if (data->is_valid) {
            if (data->size > 0) {
                if (has_permission(data)) {
                    return do_process(data);
                } else {
                    return ERROR_PERMISSION;
                }
            } else {
                return ERROR_EMPTY;
            }
        } else {
            return ERROR_INVALID;
        }
    } else {
        return ERROR_NULL;
    }
}

// ✅ 早返回减少嵌套
int process_data(data_t *data) {
    if (data == NULL) {
        return ERROR_NULL;
    }
    
    if (!data->is_valid) {
        return ERROR_INVALID;
    }
    
    if (data->size <= 0) {
        return ERROR_EMPTY;
    }
    
    if (!has_permission(data)) {
        return ERROR_PERMISSION;
    }
    
    return do_process(data);
}
```

### 6.2 使用continue简化循环
```c
// ❌ 循环内深层嵌套
void process_items(item_t *items, int count) {
    for (int i = 0; i < count; i++) {
        if (items[i].is_valid) {
            if (items[i].type == TYPE_A) {
                // 处理TYPE_A
                process_type_a(&items[i]);
            } else if (items[i].type == TYPE_B) {
                // 处理TYPE_B
                process_type_b(&items[i]);
            }
        }
    }
}

// ✅ 使用continue
void process_items(item_t *items, int count) {
    for (int i = 0; i < count; i++) {
        if (!items[i].is_valid) {
            continue;
        }
        
        switch (items[i].type) {
            case TYPE_A:
                process_type_a(&items[i]);
                break;
            case TYPE_B:
                process_type_b(&items[i]);
                break;
            default:
                log_warning("Unknown type");
                break;
        }
    }
}
```

## 七、避免魔法数字

### 7.1 使用宏定义常量
```c
// ❌ 魔法数字
if (status == 200) {
    // ...
}
for (int i = 0; i < 10; i++) {
    // ...
}
int buffer[1024];

// ✅ 有意义的常量
#define HTTP_STATUS_OK 200
#define MAX_RETRY_COUNT 10
#define BUFFER_SIZE 1024

if (status == HTTP_STATUS_OK) {
    // ...
}
for (int i = 0; i < MAX_RETRY_COUNT; i++) {
    // ...
}
int buffer[BUFFER_SIZE];
```

### 7.2 常量组织
```c
// 在头文件中定义
// config.h
#ifndef CONFIG_H
#define CONFIG_H

// 服务器配置
#define SERVER_PORT 8080
#define MAX_CONNECTIONS 100
#define CONNECTION_TIMEOUT_SEC 30

// 缓冲区配置
#define READ_BUFFER_SIZE 4096
#define WRITE_BUFFER_SIZE 4096
#define MAX_LINE_LENGTH 256

// 错误码
#define SUCCESS 0
#define ERROR_INVALID_PARAM -1
#define ERROR_OUT_OF_MEMORY -2
#define ERROR_IO_FAILURE -3

#endif
```

## 八、避免复杂表达式

### 8.1 拆分复杂条件
```c
// ❌ 复杂条件表达式
if ((user->age >= 18 && user->age <= 65) && 
    (user->status == ACTIVE || user->status == TRIAL) && 
    (user->subscription != NULL && user->subscription->is_valid)) {
    grant_access();
}

// ✅ 使用中间变量
bool is_age_valid = (user->age >= 18 && user->age <= 65);
bool is_status_valid = (user->status == ACTIVE || user->status == TRIAL);
bool has_valid_subscription = (user->subscription != NULL && 
                                user->subscription->is_valid);

if (is_age_valid && is_status_valid && has_valid_subscription) {
    grant_access();
}

// ✅ 或者使用辅助函数
bool is_user_eligible(const user_t *user) {
    return is_age_in_range(user->age, 18, 65) &&
           is_status_active(user->status) &&
           has_valid_subscription(user);
}

if (is_user_eligible(user)) {
    grant_access();
}
```

### 8.2 避免嵌套三元运算符
```c
// ❌ 嵌套三元运算符
char *status = count > 100 ? "high" : count > 50 ? "medium" : count > 10 ? "low" : "minimal";

// ✅ 使用if-else或switch
const char* get_status_level(int count) {
    if (count > 100) return "high";
    if (count > 50) return "medium";
    if (count > 10) return "low";
    return "minimal";
}

const char *status = get_status_level(count);
```

## 九、可读性检查清单

### 9.1 命名检查
- [ ] 所有变量名是否清晰表达其用途？
- [ ] 函数名是否使用"动词+名词"结构？
- [ ] 布尔变量是否使用is/has/can前缀？
- [ ] 是否避免了无意义名称（如temp、data、flag）？
- [ ] 常量是否使用大写字母和下划线命名？

### 9.2 格式检查
- [ ] 缩进是否一致（4空格或团队规范）？
- [ ] 相关代码块之间是否有空行？
- [ ] 长行是否合理换行（建议不超过80-120字符）？
- [ ] 结构体、枚举定义是否对齐？

### 9.3 函数检查
- [ ] 单个函数是否控制在50-100行内？
- [ ] 每个函数是否只做一件事？
- [ ] 函数嵌套是否超过3层？
- [ ] 是否使用了早返回减少嵌套？

### 9.4 注释检查
- [ ] 注释是否解释了"为什么"而非"是什么"？
- [ ] 是否有TODO/FIXME标记待处理问题？
- [ ] 公共函数是否有文档注释？
- [ ] 是否有冗余或过时的注释需要删除？

### 9.5 代码质量检查
- [ ] 是否有魔法数字需要定义为常量？
- [ ] 复杂表达式是否已拆分？
- [ ] 是否避免了嵌套三元运算符？
- [ ] 条件判断是否易于理解？

---

**总结**：可读性是代码质量的基础。优秀的代码应该像散文一样流畅，让读者能够快速理解代码的意图和逻辑。记住：代码是写给人看的，其次才是给机器执行的。
