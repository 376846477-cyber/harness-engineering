# C++可读性规范详细参考

## 一、名称传递信息

好的名称是最重要的可读性要素，名称应该自解释代码意图。

### 函数命名

**原则**：函数名应体现行为和意图，使用动词开头。

```cpp
// 不推荐 - 名称模糊
void process(int x);
bool check(User& u);
void handle();

// 推荐 - 名称清晰
void validateUserCredentials(const Credentials& creds);
bool hasPermission(const User& user, Permission perm);
void sendEmailNotification(const Email& email);

// 推荐 - 使用强类型参数提高可读性
void setViewport(int width, int height);  // 一般
void setViewport(Width w, Height h);       // 更好，避免参数顺序错误
```

### 变量命名

**原则**：变量名应包含重要细节，如单位、状态等。

```cpp
// 不推荐 - 缺少上下文
int size;
int count;
bool flag;

// 推荐 - 带单位或状态
size_t bufferSizeInBytes;
int activeConnectionCount;
bool isAuthenticated;

// 推荐 - 布尔变量使用is/has/can/should前缀
bool isOpen;
bool hasChildren;
bool canWrite;
bool shouldRetry;
```

### 避免无意义名称

```cpp
// 不推荐 - 无意义缩写
int tmp, temp, val, ret;
void foo();
void bar();

// 不推荐 - 匈牙利命名法（现代C++不推荐）
int nCount;    // 冗余
string strName; // 冗余

// 推荐 - 使用完整、有意义的名称
int retryCount;
string userName;
```

---

## 二、格式传递信息

代码格式应帮助读者理解代码结构，一致的格式能降低认知负担。

### 缩进与空行

```cpp
// 推荐 - 使用空行分隔逻辑块
void processOrder(const Order& order) {
    // 验证阶段
    if (!validateOrder(order)) {
        throw InvalidOrderException{};
    }
    
    // 计算阶段
    auto total = calculateTotal(order);
    auto discount = calculateDiscount(order);
    
    // 保存阶段
    saveOrder(order, total, discount);
}

// 推荐 - 相关代码放在一起
void configure() {
    // 日志配置
    logger.setLevel(LogLevel::Debug);
    logger.setOutput("app.log");
    
    // 网络配置
    server.setPort(8080);
    server.setTimeout(30s);
}
```

### 对齐与分组

```cpp
// 不推荐 - 随意排列的成员
class User {
    string address;
    int age;
    string name;
    bool active;
    int id;
};

// 推荐 - 按访问级别和逻辑分组
class User {
public:
    // 构造与析构
    User() = default;
    explicit User(int id);
    
    // 访问器
    int getId() const;
    const string& getName() const;
    bool isActive() const;
    
    // 修改器
    void setName(string_view name);
    void setActive(bool active);
    
private:
    // 数据成员
    int id_ = 0;
    string name_;
    bool active_ = false;
    string address_;
};
```

---

## 三、注释作为补充

注释应解释"为什么"，而非"是什么"。代码应自解释，注释作为补充。

### 注释原则

```cpp
// 不推荐 - 重复代码含义
i++;  // i加1
return count;  // 返回计数

// 推荐 - 解释原因和背景
// 使用位运算代替乘法，性能提升约20%（基准测试结果）
index = value >> 3;

// 推荐 - 解释非显而易见的业务逻辑
// 注意：退款金额不能超过原订单金额的50%，这是风控系统的限制
if (refundAmount > orderAmount * 0.5) {
    throw RefundLimitExceeded{};
}
```

### TODO与FIXME注释

```cpp
// TODO(author): 待实现功能说明
// TODO(zhangsan): 添加缓存机制减少数据库查询

// FIXME(author): 已知问题说明
// FIXME(lisi): 多线程下存在竞态条件，需要加锁
```

### 文档注释

```cpp
/**
 * @brief 计算两个日期之间的工作日数量
 * 
 * @param startDate 开始日期（包含）
 * @param endDate 结束日期（包含）
 * @return 工作日数量（周一至周五）
 * 
 * @throws InvalidDateRange 如果startDate > endDate
 * 
 * @note 不考虑法定节假日
 */
int countWorkingDays(Date startDate, Date endDate);
```

---

## 四、函数长度控制

### 单一职责原则

```cpp
// 不推荐 - 函数过长，职责不清
void processRequest(Request& req) {
    // 100行代码，包含验证、解析、处理、响应...
}

// 推荐 - 拆分为小函数，每个函数只做一件事
void processRequest(Request& req) {
    validateRequest(req);
    auto data = parseRequest(req);
    auto result = processData(data);
    sendResponse(result);
}

// 每个子函数不超过30行
void validateRequest(const Request& req) {
    checkRequiredFields(req);
    validateToken(req.token());
    checkRateLimit(req.clientId());
}
```

### 函数长度建议

| 行数范围 | 评价 | 建议 |
|---------|------|------|
| 1-10行 | 优秀 | 保持 |
| 11-20行 | 良好 | 可接受 |
| 21-40行 | 一般 | 考虑拆分 |
| >40行 | 较差 | 必须拆分 |

---

## 五、减少嵌套（早返回）

### 卫语句（Guard Clauses）

```cpp
// 不推荐 - 深层嵌套
Result process(const Input& input) {
    if (input.isValid()) {
        if (input.hasData()) {
            if (input.isAuthorized()) {
                // 实际处理逻辑
                return doProcess(input);
            } else {
                return Error::Unauthorized;
            }
        } else {
            return Error::NoData;
        }
    } else {
        return Error::InvalidInput;
    }
}

// 推荐 - 早返回，减少嵌套
Result process(const Input& input) {
    if (!input.isValid()) return Error::InvalidInput;
    if (!input.hasData()) return Error::NoData;
    if (!input.isAuthorized()) return Error::Unauthorized;
    
    return doProcess(input);
}
```

### 使用continue和break

```cpp
// 不推荐 - 嵌套的if-else
void processItems(const vector<Item>& items) {
    for (const auto& item : items) {
        if (item.isValid()) {
            if (item.needsProcessing()) {
                process(item);
            }
        }
    }
}

// 推荐 - 使用continue跳过
void processItems(const vector<Item>& items) {
    for (const auto& item : items) {
        if (!item.isValid()) continue;
        if (!item.needsProcessing()) continue;
        
        process(item);
    }
}
```

### 提取方法

```cpp
// 不推荐 - 嵌套的条件判断
void handleEvent(const Event& event) {
    if (event.type() == EventType::Click) {
        if (event.target() == "button") {
            handleClick(event);
        } else if (event.target() == "link") {
            handleLink(event);
        }
    } else if (event.type() == EventType::Key) {
        // ...
    }
}

// 推荐 - 提取方法
void handleEvent(const Event& event) {
    switch (event.type()) {
        case EventType::Click: handleClickEvent(event); break;
        case EventType::Key: handleKeyEvent(event); break;
        default: handleUnknownEvent(event);
    }
}
```

---

## 六、避免魔法数字

### 使用constexpr和命名常量

```cpp
// 不推荐 - 魔法数字
if (status == 200) { /* ... */ }
if (size > 1048576) { /* ... */ }
sleep(3000);

// 推荐 - 命名常量
constexpr int HTTP_OK = 200;
constexpr size_t MAX_FILE_SIZE = 1024 * 1024;  // 1MB
constexpr auto RETRY_DELAY = 3s;

if (status == HTTP_OK) { /* ... */ }
if (size > MAX_FILE_SIZE) { /* ... */ }
sleep(RETRY_DELAY);
```

### 使用枚举和枚举类

```cpp
// 不推荐 - 整数常量
const int STATUS_PENDING = 0;
const int STATUS_RUNNING = 1;
const int STATUS_COMPLETED = 2;

// 推荐 - 强类型枚举
enum class Status {
    Pending,
    Running,
    Completed,
    Failed
};

void updateStatus(Status newStatus) {
    switch (newStatus) {
        case Status::Pending: /* ... */ break;
        case Status::Running: /* ... */ break;
        case Status::Completed: /* ... */ break;
        case Status::Failed: /* ... */ break;
    }
}
```

---

## 七、明智使用auto

### 推荐使用auto的场景

```cpp
// 推荐 - 类型冗长或显而易见
auto iter = container.find(key);
auto lambda = [](int x) { return x * 2; };
auto ptr = std::make_shared<Widget>();
auto [key, value] = *map.begin();  // 结构化绑定

// 推荐 - 避免意外类型转换
auto result = compute();  // 精确保留返回类型
```

### 不推荐过度使用auto

```cpp
// 不推荐 - 类型信息重要但被隐藏
auto x = compute();  // x的类型是什么？int? double? 自定义类型?
auto y = getValue(); // 需要看函数签名才知道类型

// 推荐 - 显式类型更有利于理解
int count = getCount();
double ratio = calculateRatio();
Widget* widget = getWidget();

// 不推荐 - 使用auto来"节省"打字
auto i = 0;      // int，但不明显
auto s = "hello"; // const char*，不是string！

// 推荐
int i = 0;
string s = "hello";
```

---

## 八、Pythonic C++（现代C++特性）

### 范围for循环

```cpp
// 传统写法
for (size_t i = 0; i < items.size(); ++i) {
    process(items[i]);
}

// Pythonic写法
for (const auto& item : items) {
    process(item);
}

// 带索引（C++20）
for (const auto& [index, item] : views::enumerate(items)) {
    cout << index << ": " << item << endl;
}
```

### 结构化绑定

```cpp
// 传统写法
auto result = computePair();
int first = result.first;
int second = result.second;

// Pythonic写法
auto [first, second] = computePair();

// 遍历map
for (const auto& [key, value] : configMap) {
    cout << key << " = " << value << endl;
}
```

### std::optional替代指针

```cpp
// 传统写法 - 使用nullptr表示无值
Item* findItem(int id) {
    auto it = items.find(id);
    return it != items.end() ? &it->second : nullptr;
}

// Pythonic写法 - 使用optional
std::optional<Item> findItem(int id) {
    auto it = items.find(id);
    if (it != items.end()) {
        return it->second;
    }
    return std::nullopt;
}

// 使用
if (auto item = findItem(42)) {
    process(*item);
}
```

### 使用标准算法

```cpp
// 传统写法
bool found = false;
for (const auto& item : items) {
    if (item.id() == targetId) {
        found = true;
        break;
    }
}

// Pythonic写法
bool found = std::any_of(items.begin(), items.end(),
    [&](const auto& item) { return item.id() == targetId; });

// C++20 ranges（更Pythonic）
bool found = std::ranges::any_of(items,
    [&](const auto& item) { return item.id() == targetId; });
```

---

## 可读性检查清单

### 命名检查
- [ ] 函数名是否表达行为意图？
- [ ] 变量名是否包含必要信息（单位、状态）？
- [ ] 布尔变量是否使用is/has/can前缀？
- [ ] 是否避免了缩写和无意义名称？

### 格式检查
- [ ] 是否使用一致的缩进风格？
- [ ] 逻辑块之间是否有空行分隔？
- [ ] 相关代码是否组织在一起？
- [ ] 类成员是否按逻辑分组？

### 注释检查
- [ ] 注释是否解释"为什么"而非"是什么"？
- [ ] 是否避免重复代码含义的注释？
- [ ] 公共API是否有文档注释？
- [ ] 是否正确标记TODO/FIXME？

### 结构检查
- [ ] 函数长度是否在40行以内？
- [ ] 嵌套深度是否不超过3层？
- [ ] 是否使用卫语句减少嵌套？
- [ ] 是否避免魔法数字？

### 现代C++检查
- [ ] 是否合理使用auto？
- [ ] 是否使用范围for替代索引循环？
- [ ] 是否使用结构化绑定？
- [ ] 是否使用constexpr定义常量？
- [ ] 是否使用枚举类替代整数常量？

---

## 总结

可读性是代码质量的基础，遵循以下核心原则：

1. **名称即文档** - 好的名称减少注释需求
2. **格式传递结构** - 一致的格式降低认知负担
3. **注释解释原因** - 补充代码无法表达的信息
4. **短小精悍** - 函数长度控制，单一职责
5. **扁平结构** - 减少嵌套，早返回
6. **拒绝魔法** - 使用命名常量和枚举
7. **现代语法** - 合理使用现代C++特性

记住：代码是写给人看的，顺便能在机器上运行。
