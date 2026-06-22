# C++可靠性规范详细参考

## 一、RAII资源管理

### 1.1 核心原则
RAII（Resource Acquisition Is Initialization）是C++资源管理的基石，确保资源生命周期与对象生命周期绑定。

**核心规则：**
- 构造函数获取资源，析构函数释放资源
- 禁止裸资源（裸指针、裸文件句柄、裸锁）
- 所有资源都应封装在RAII类中
- 使用标准库RAII设施优先于自定义实现

### 1.2 内存资源管理

**正确示例：使用智能指针**
```cpp
class ResourceManager {
public:
    std::unique_ptr<Resource> createResource(const Config& config) {
        return std::make_unique<Resource>(config);
    }
    
    std::shared_ptr<SharedData> getSharedData() {
        return std::make_shared<SharedData>();
    }
};

// 使用weak_ptr避免循环引用
class Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;  // 避免循环引用
};
```

**错误示例：裸指针泄漏风险**
```cpp
// 错误：裸指针容易泄漏
Resource* createResource() {
    return new Resource();  // 调用者可能忘记delete
}

// 错误：异常导致泄漏
void process() {
    Resource* r = new Resource();
    doSomething();  // 如果抛出异常，r泄漏
    delete r;
}
```

### 1.3 文件资源管理

```cpp
// 正确：使用fstream自动管理
void processFile(const std::string& path) {
    std::ifstream file(path);
    if (!file.is_open()) {
        throw std::runtime_error("Cannot open file");
    }
    // 文件在作用域结束时自动关闭
}

// 错误：手动管理容易忘记关闭
void processFileBad(const std::string& path) {
    FILE* file = fopen(path.c_str(), "r");
    // ... 如果提前返回，file可能泄漏
    fclose(file);
}
```

### 1.4 锁资源管理

```cpp
class ThreadSafeCounter {
    mutable std::mutex mutex_;
    int count_ = 0;
    
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mutex_);  // RAII锁
        ++count_;
    }
    
    int get() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return count_;
    }
    
    // 使用scoped_lock管理多个锁（避免死锁）
    void transfer(ThreadSafeCounter& other) {
        std::scoped_lock lock(mutex_, other.mutex_);
        // 两个锁同时获取，避免死锁
    }
};
```

### 1.5 自定义RAII类

```cpp
template<typename Handle, auto Deleter>
class ScopedHandle {
    Handle handle_;
    
public:
    explicit ScopedHandle(Handle h = Handle{}) : handle_(h) {}
    ~ScopedHandle() { 
        if (handle_) Deleter(handle_); 
    }
    
    ScopedHandle(const ScopedHandle&) = delete;
    ScopedHandle& operator=(const ScopedHandle&) = delete;
    
    ScopedHandle(ScopedHandle&& other) noexcept : handle_(other.handle_) {
        other.handle_ = Handle{};
    }
    
    Handle get() const { return handle_; }
    explicit operator bool() const { return handle_ != Handle{}; }
};

// 使用示例
using ScopedFile = ScopedHandle<FILE*, fclose>;
using ScopedDir = ScopedHandle<DIR*, closedir>;
```

---

## 二、智能指针使用规范

### 2.1 选择指南

| 指针类型 | 所有权语义 | 使用场景 |
|---------|----------|---------|
| `unique_ptr` | 独占所有权 | 默认选择，性能最优 |
| `shared_ptr` | 共享所有权 | 多处共享，无法确定生命周期 |
| `weak_ptr` | 无所有权 | 打破循环引用，观察者模式 |
| `裸指针` | 无所有权 | 非拥有观察指针、性能关键路径 |

### 2.2 unique_ptr最佳实践

```cpp
class Factory {
public:
    // 工厂函数返回unique_ptr
    std::unique_ptr<Product> createProduct(ProductType type) {
        switch (type) {
            case ProductType::A: return std::make_unique<ProductA>();
            case ProductType::B: return std::make_unique<ProductB>();
        }
        throw std::invalid_argument("Unknown product type");
    }
};

// 自定义删除器
auto fileDeleter = [](FILE* f) { 
    if (f) fclose(f); 
};
std::unique_ptr<FILE, decltype(fileDeleter)> file(fopen("test.txt", "r"), fileDeleter);

// 数组形式
std::unique_ptr<int[]> arr = std::make_unique<int[]>(100);
```

### 2.3 shared_ptr最佳实践

```cpp
class SharedCache {
    std::shared_ptr<const Data> data_;
    
public:
    std::shared_ptr<const Data> getData() const {
        return data_;  // 共享只读数据
    }
    
    void update(std::shared_ptr<const Data> newData) {
        data_ = std::move(newData);  // 原子更新
    }
};

// enable_shared_from_this用于异步回调
class AsyncOperation : public std::enable_shared_from_this<AsyncOperation> {
public:
    void start() {
        auto self = shared_from_this();  // 在回调中保持存活
        asyncService.execute([self]() {
            self->onComplete();
        });
    }
    
    void onComplete() {
        // ...
    }
};
```

### 2.4 避免的问题

```cpp
// 错误：重复管理
void bad1() {
    Resource* raw = new Resource();
    std::shared_ptr<Resource> p1(raw);
    std::shared_ptr<Resource> p2(raw);  // 双重删除！
}

// 错误：this指针问题
class BadExample {
public:
    void registerCallback() {
        callback([this]() {  // this可能悬垂
            doSomething();
        });
    }
};

// 正确：使用shared_from_this
class GoodExample : public std::enable_shared_from_this<GoodExample> {
public:
    void registerCallback() {
        auto self = shared_from_this();
        callback([self]() {
            self->doSomething();
        });
    }
};
```

---

## 三、防御式编程

### 3.1 参数校验

```cpp
class BankAccount {
    double balance_;
    
public:
    void withdraw(double amount) {
        // 前置条件检查
        if (amount <= 0) {
            throw std::invalid_argument("Amount must be positive");
        }
        if (amount > balance_) {
            throw std::runtime_error("Insufficient funds");
        }
        
        balance_ -= amount;
    }
    
    // 使用断言检查不变量
    void transfer(BankAccount& to, double amount) {
        assert(this != &to);  // 不能转给自己
        double oldFromBalance = balance_;
        double oldToBalance = to.balance_;
        
        withdraw(amount);
        to.deposit(amount);
        
        // 后置条件检查
        assert(balance_ == oldFromBalance - amount);
        assert(to.balance_ == oldToBalance + amount);
    }
};
```

### 3.2 边界安全

```cpp
// 使用at()进行边界检查
void processVector(const std::vector<int>& data, size_t index) {
    try {
        int value = data.at(index);  // 抛出out_of_range
        // 处理value
    } catch (const std::out_of_range& e) {
        logError("Index out of range: " + std::string(e.what()));
    }
}

// 使用span避免拷贝和边界问题
void processData(std::span<const int> data) {
    for (int val : data) {
        // 安全访问
    }
}

// 字符串安全
std::string safeCopy(const std::string& src, size_t maxLen) {
    if (src.size() > maxLen) {
        return src.substr(0, maxLen);
    }
    return src;
}
```

### 3.3 使用Optional和Expected

```cpp
#include <optional>
#include <expected>

// 使用optional表示可能失败的操作
std::optional<User> findUser(int id) {
    auto it = users.find(id);
    if (it != users.end()) {
        return it->second;
    }
    return std::nullopt;
}

// 使用expected携带错误信息（C++23）
std::expected<Data, ErrorCode> loadData(const std::string& path) {
    std::ifstream file(path);
    if (!file) {
        return std::unexpected(ErrorCode::FileNotFound);
    }
    
    Data data;
    if (!(file >> data)) {
        return std::unexpected(ErrorCode::ParseError);
    }
    
    return data;
}

// 使用示例
void processData() {
    auto user = findUser(42);
    if (user) {
        std::cout << "Found: " << user->name << std::endl;
    } else {
        std::cout << "User not found" << std::endl;
    }
    
    auto result = loadData("config.json");
    if (result) {
        // 使用 *result 或 result.value()
    } else {
        handleError(result.error());
    }
}
```

---

## 四、输入验证

### 4.1 基本验证

```cpp
class InputValidator {
public:
    static bool isValidEmail(const std::string& email) {
        static const std::regex pattern(R"(^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$)");
        return std::regex_match(email, pattern);
    }
    
    static std::string sanitizeHtml(const std::string& input) {
        std::string result;
        result.reserve(input.size());
        
        for (char c : input) {
            switch (c) {
                case '<': result += "&lt;"; break;
                case '>': result += "&gt;"; break;
                case '&': result += "&amp;"; break;
                case '"': result += "&quot;"; break;
                case '\'': result += "&#39;"; break;
                default: result += c;
            }
        }
        
        return result;
    }
    
    static bool isValidPath(const std::string& path) {
        // 防止路径遍历攻击
        if (path.find("..") != std::string::npos) {
            return false;
        }
        if (path.find('\0') != std::string::npos) {
            return false;
        }
        return true;
    }
};
```

### 4.2 数值范围验证

```cpp
template<typename T>
class RangeValidator {
    T min_, max_;
    
public:
    RangeValidator(T min, T max) : min_(min), max_(max) {}
    
    std::optional<T> validate(T value) const {
        if (value >= min_ && value <= max_) {
            return value;
        }
        return std::nullopt;
    }
    
    T validateOrThrow(T value) const {
        if (value < min_ || value > max_) {
            throw std::out_of_range("Value out of range");
        }
        return value;
    }
};

// 使用示例
RangeValidator<int> ageValidator(0, 150);
RangeValidator<double> temperatureValidator(-273.15, 1000.0);
```

---

## 五、错误处理策略

### 5.1 异常安全级别

```cpp
// 基本保证：不泄漏资源，对象处于有效状态
class BasicGuarantee {
    std::vector<int> data_;
    
public:
    void append(const std::vector<int>& items) {
        // 如果push_back失败，data_仍然有效
        for (int item : items) {
            data_.push_back(item);  // 可能抛出bad_alloc
        }
    }
};

// 强保证：事务性，要么成功要么回滚
class StrongGuarantee {
    std::vector<int> data_;
    
public:
    void append(const std::vector<int>& items) {
        std::vector<int> temp = data_;  // 复制
        for (int item : items) {
            temp.push_back(item);
        }
        data_.swap(temp);  // noexcept操作
    }
};

// 不抛出保证：永不失败
class NoThrowGuarantee {
    std::vector<int> data_;
    
public:
    void swap(NoThrowGuarantee& other) noexcept {
        data_.swap(other.data_);  // vector::swap是noexcept
    }
};
```

### 5.2 错误码与异常的选择

```cpp
// 错误码：适用于可预期的错误
enum class FileError {
    Success,
    NotFound,
    PermissionDenied,
    DiskFull
};

FileError readFile(const std::string& path, std::string& content) {
    std::ifstream file(path);
    if (!file) {
        if (errno == ENOENT) return FileError::NotFound;
        if (errno == EACCES) return FileError::PermissionDenied;
        return FileError::DiskFull;
    }
    
    content = std::string(std::istreambuf_iterator<char>(file), {});
    return FileError::Success;
}

// 异常：适用于意外错误、需要跨层传递
class NetworkException : public std::runtime_error {
    int errorCode_;
    
public:
    NetworkException(const std::string& msg, int code)
        : std::runtime_error(msg), errorCode_(code) {}
    
    int errorCode() const { return errorCode_; }
};

void sendRequest(const Request& req) {
    if (!connection.isAlive()) {
        throw NetworkException("Connection lost", 1001);
    }
}
```

### 5.3 资源清理策略

```cpp
// 使用RAII自动清理
void processFile(const std::string& path) {
    std::ifstream file(path);
    std::vector<int> buffer(1000);
    
    // 无论成功还是异常，file和buffer都会自动清理
    processBuffer(buffer);
}

// 使用scope_guard（C++20或第三方库）
template<typename F>
class ScopeGuard {
    F func_;
    bool active_ = true;
    
public:
    explicit ScopeGuard(F f) : func_(std::move(f)) {}
    ~ScopeGuard() { if (active_) func_(); }
    
    void dismiss() { active_ = false; }
    
    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
};

// 使用示例
void transaction() {
    beginTransaction();
    ScopeGuard rollback([]{ rollbackTransaction(); });
    
    doWork();
    
    commitTransaction();
    rollback.dismiss();  // 成功则取消回滚
}
```

---

## 六、降级处理

### 6.1 多层降级策略

```cpp
class DataFetcher {
    Cache cache_;
    Database db_;
    Config config_;
    
public:
    std::optional<Data> fetch(int id) {
        // 第一层：缓存
        if (auto cached = cache_.get(id)) {
            return cached;
        }
        
        // 第二层：数据库
        try {
            if (auto data = db_.query(id)) {
                cache_.set(id, *data);
                return data;
            }
        } catch (const DatabaseError& e) {
            logError(e.what());
        }
        
        // 第三层：默认值
        if (config_.hasDefault(id)) {
            return config_.getDefault(id);
        }
        
        // 第四层：空值
        return std::nullopt;
    }
};
```

### 6.2 熔断器模式

```cpp
class CircuitBreaker {
    enum class State { Closed, Open, HalfOpen };
    
    State state_ = State::Closed;
    int failureCount_ = 0;
    int successCount_ = 0;
    std::chrono::steady_clock::time_point lastFailure_;
    
    const int failureThreshold_ = 5;
    const int successThreshold_ = 2;
    const std::chrono::milliseconds timeout_{5000};
    
public:
    template<typename F>
    auto execute(F&& func) -> decltype(func()) {
        if (state_ == State::Open) {
            if (std::chrono::steady_clock::now() - lastFailure_ > timeout_) {
                state_ = State::HalfOpen;
                successCount_ = 0;
            } else {
                throw std::runtime_error("Circuit breaker is open");
            }
        }
        
        try {
            auto result = func();
            onSuccess();
            return result;
        } catch (...) {
            onFailure();
            throw;
        }
    }
    
private:
    void onSuccess() {
        failureCount_ = 0;
        if (state_ == State::HalfOpen) {
            if (++successCount_ >= successThreshold_) {
                state_ = State::Closed;
            }
        }
    }
    
    void onFailure() {
        lastFailure_ = std::chrono::steady_clock::now();
        if (++failureCount_ >= failureThreshold_) {
            state_ = State::Open;
        }
    }
};
```

### 6.3 限流器

```cpp
class RateLimiter {
    std::chrono::steady_clock::time_point lastRequest_;
    int tokens_;
    const int maxTokens_;
    const std::chrono::milliseconds refillInterval_;
    const int refillAmount_;
    mutable std::mutex mutex_;
    
public:
    RateLimiter(int maxTokens, std::chrono::milliseconds interval, int refill)
        : tokens_(maxTokens), maxTokens_(maxTokens),
          refillInterval_(interval), refillAmount_(refill) {}
    
    bool tryAcquire() {
        std::lock_guard<std::mutex> lock(mutex_);
        
        auto now = std::chrono::steady_clock::now();
        auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(now - lastRequest_);
        
        // 补充令牌
        int newTokens = (elapsed.count() / refillInterval_.count()) * refillAmount_;
        tokens_ = std::min(maxTokens_, tokens_ + newTokens);
        lastRequest_ = now;
        
        if (tokens_ > 0) {
            --tokens_;
            return true;
        }
        return false;
    }
};
```

---

## 七、内存安全

### 7.1 避免内存泄漏

```cpp
// 使用工具检测
// - Valgrind (Linux)
// - AddressSanitizer (GCC/Clang: -fsanitize=address)
// - Visual Studio CRT Debug Heap

// 常见泄漏场景及修复
class LeakExample {
    std::vector<int*> ptrs_;
    
public:
    // 错误：容器中的裸指针
    void add(int* p) {
        ptrs_.push_back(p);
    }
    
    // 正确：使用智能指针
    void addGood(std::unique_ptr<int> p) {
        ptrs_good_.push_back(std::move(p));
    }
    
    std::vector<std::unique_ptr<int>> ptrs_good_;
};

// 循环引用问题
class Node {
    std::shared_ptr<Node> next_;
    std::weak_ptr<Node> prev_;  // 使用weak_ptr打破循环
    
public:
    void setNext(std::shared_ptr<Node> n) { next_ = std::move(n); }
    void setPrev(std::shared_ptr<Node> p) { prev_ = p; }
};
```

### 7.2 避免悬垂指针

```cpp
// 错误：返回局部变量的指针
const int* badReturn() {
    int local = 42;
    return &local;  // 悬垂指针！
}

// 正确：返回值或智能指针
int goodReturn() {
    return 42;
}

std::unique_ptr<int> goodReturnPtr() {
    return std::make_unique<int>(42);
}

// 错误：容器迭代器失效
void badIterator() {
    std::vector<int> v = {1, 2, 3};
    int* p = &v[0];
    v.push_back(4);  // 可能重新分配，p失效
    std::cout << *p;  // 未定义行为
}

// 正确：在修改后重新获取
void goodIterator() {
    std::vector<int> v = {1, 2, 3};
    v.push_back(4);
    int* p = &v[0];  // 修改后获取
    std::cout << *p;
}
```

### 7.3 使用安全容器

```cpp
// 使用string_view避免悬垂
void processString(std::string_view sv) {
    // sv不拥有数据，调用者必须保证生命周期
    std::cout << sv << std::endl;
}

// 使用span安全访问数组
void processArray(std::span<int> arr) {
    for (int& val : arr) {
        val *= 2;
    }
}

int data[] = {1, 2, 3, 4, 5};
processArray(data);  // 自动推导大小
```

---

## 八、异常安全

### 8.1 异常安全函数设计

```cpp
class SafeContainer {
    std::vector<int> data_;
    
public:
    // 强异常安全保证
    void append(const std::vector<int>& items) {
        std::vector<int> temp = data_;
        temp.insert(temp.end(), items.begin(), items.end());
        data_.swap(temp);  // noexcept
    }
    
    // 基本异常安全保证
    void push(int value) {
        data_.push_back(value);  // 可能抛出，但不会破坏不变量
    }
    
    // noexcept函数
    int size() const noexcept {
        return static_cast<int>(data_.size());
    }
    
    void swap(SafeContainer& other) noexcept {
        data_.swap(other.data_);
    }
};
```

### 8.2 异常规范

```cpp
// C++11及以后
void mayThrow();  // 可能抛出任何异常
void noThrow() noexcept;  // 保证不抛出
void conditionalNoThrow() noexcept(expr);  // 条件noexcept

// 标准库模式
template<typename T>
typename std::enable_if<std::is_nothrow_move_constructible<T>::value>::type
safeSwap(T& a, T& b) noexcept {
    T temp = std::move(a);
    a = std::move(b);
    b = std::move(temp);
}
```

### 8.3 异常处理最佳实践

```cpp
// 捕获特定异常，从具体到一般
void handleErrors() {
    try {
        riskyOperation();
    } catch (const SpecificError& e) {
        // 处理特定错误
    } catch (const std::runtime_error& e) {
        // 处理运行时错误
    } catch (const std::exception& e) {
        // 处理所有标准异常
        logError(e.what());
    } catch (...) {
        // 处理未知异常
        logError("Unknown exception");
        throw;  // 重新抛出
    }
}

// RAII确保清理
void safeOperation() {
    std::unique_ptr<Resource> r = allocateResource();
    operationThatMightThrow();
    // 即使抛出异常，r也会被正确清理
}
```

---

## 九、核心流程依赖最小化

### 9.1 依赖注入

```cpp
// 接口抽象
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(const std::string& msg) = 0;
};

// 核心业务不依赖具体实现
class CoreService {
    ILogger& logger_;
    
public:
    explicit CoreService(ILogger& logger) : logger_(logger) {}
    
    void doWork() {
        logger_.log("Starting work");
        // 核心逻辑
        logger_.log("Work completed");
    }
};

// 生产环境实现
class FileLogger : public ILogger {
    void log(const std::string& msg) override {
        // 写入文件
    }
};

// 测试环境实现
class MockLogger : public ILogger {
    std::vector<std::string> messages_;
    
    void log(const std::string& msg) override {
        messages_.push_back(msg);
    }
};
```

### 9.2 降级服务

```cpp
class PaymentService {
    IPaymentGateway& primary_;
    IPaymentGateway& backup_;
    bool useBackup_ = false;
    
public:
    PaymentService(IPaymentGateway& primary, IPaymentGateway& backup)
        : primary_(primary), backup_(backup) {}
    
    Result process(const Order& order) {
        try {
            if (!useBackup_) {
                return primary_.process(order);
            }
        } catch (const GatewayError& e) {
            useBackup_ = true;
            logError("Primary gateway failed, switching to backup");
        }
        
        return backup_.process(order);
    }
    
    void reset() {
        useBackup_ = false;
    }
};
```

### 9.3 异步解耦

```cpp
class AsyncTaskQueue {
    std::queue<std::function<void()>> tasks_;
    std::mutex mutex_;
    std::condition_variable cv_;
    std::atomic<bool> running_{true};
    std::thread worker_;
    
public:
    AsyncTaskQueue() : worker_([this] { run(); }) {}
    
    ~AsyncTaskQueue() {
        running_ = false;
        cv_.notify_all();
        worker_.join();
    }
    
    void submit(std::function<void()> task) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            tasks_.push(std::move(task));
        }
        cv_.notify_one();
    }
    
private:
    void run() {
        while (running_) {
            std::function<void()> task;
            {
                std::unique_lock<std::mutex> lock(mutex_);
                cv_.wait(lock, [this] { 
                    return !tasks_.empty() || !running_; 
                });
                
                if (!running_) return;
                
                task = std::move(tasks_.front());
                tasks_.pop();
            }
            
            try {
                task();
            } catch (const std::exception& e) {
                logError(e.what());
            }
        }
    }
};
```

---

## 十、检查清单

### 10.1 RAII检查清单

- [ ] 所有动态分配的内存都使用智能指针
- [ ] 文件句柄使用fstream或RAII包装器
- [ ] 锁使用lock_guard/scoped_lock
- [ ] 自定义资源实现了RAII包装
- [ ] 析构函数标记为noexcept
- [ ] 禁用拷贝构造/赋值（或正确实现）

### 10.2 内存安全检查清单

- [ ] 无裸new/delete（除库实现外）
- [ ] 检查循环引用，使用weak_ptr打破
- [ ] 容器修改后不使用旧迭代器
- [ ] 不返回局部变量指针/引用
- [ ] 使用静态分析工具检测泄漏
- [ ] 测试覆盖异常路径

### 10.3 异常安全检查清单

- [ ] 确定每个函数的异常安全级别
- [ ] 关键操作提供强保证
- [ ] 使用swap实现事务性操作
- [ ] 正确使用noexcept规范
- [ ] 捕获特定异常，从具体到一般
- [ ] 异常中不抛出异常

### 10.4 防御式编程检查清单

- [ ] 验证所有外部输入
- [ ] 检查参数边界条件
- [ ] 使用at()代替[]进行边界检查
- [ ] 检查指针有效性
- [ ] 使用optional/expected处理可能失败的操作
- [ ] 记录不变量断言

### 10.5 容错机制检查清单

- [ ] 实现重试机制（带退避）
- [ ] 实现超时控制
- [ ] 实现熔断器模式
- [ ] 实现限流机制
- [ ] 提供多级降级策略
- [ ] 核心流程有备选路径

### 10.6 依赖管理检查清单

- [ ] 使用接口抽象外部依赖
- [ ] 通过构造函数注入依赖
- [ ] 核心逻辑不依赖具体实现
- [ ] 异步处理非关键路径
- [ ] 提供默认实现或降级服务
- [ ] 可测试性：依赖可替换

---

## 十一、工具推荐

### 11.1 静态分析工具

| 工具 | 功能 | 使用方式 |
|-----|------|---------|
| Clang-Tidy | 全面代码检查 | `clang-tidy main.cpp -- -std=c++20` |
| Clazy | Qt相关检查 | `clazy main.cpp` |
| Cppcheck | 内存错误检测 | `cppcheck --enable=all main.cpp` |
| Coverity | 商业静态分析 | CI/CD集成 |

### 11.2 运行时检测工具

| 工具 | 功能 | 编译选项 |
|-----|------|---------|
| AddressSanitizer | 内存错误检测 | `-fsanitize=address -fno-omit-frame-pointer` |
| UndefinedBehaviorSanitizer | 未定义行为检测 | `-fsanitize=undefined` |
| ThreadSanitizer | 数据竞争检测 | `-fsanitize=thread` |
| Valgrind | 内存泄漏检测 | `valgrind --leak-check=full ./program` |

### 11.3 调试辅助

```cpp
// 使用assert检查不变量
#include <cassert>

void process(std::vector<int>& data) {
    assert(!data.empty() && "Data must not be empty");
    // ...
}

// 使用static_assert编译期检查
static_assert(sizeof(void*) == 8, "64-bit platform required");

// 自定义断言（带日志）
#ifdef DEBUG
#define ASSERT_MSG(cond, msg) \
    if (!(cond)) { \
        std::cerr << "Assertion failed: " << #cond << "\n" \
                  << "Message: " << msg << "\n" \
                  << "File: " << __FILE__ << "\n" \
                  << "Line: " << __LINE__ << std::endl; \
        std::abort(); \
    }
#else
#define ASSERT_MSG(cond, msg) ((void)0)
#endif
```

---

## 参考资料

1. **C++ Core Guidelines** - https://isocpp.github.io/CppCoreGuidelines/
2. **Effective C++** (Scott Meyers) - Items on resource management
3. **Effective Modern C++** (Scott Meyers) - Smart pointers and move semantics
4. **C++ Concurrency In Action** (Anthony Williams) - Thread-safe programming
5. **Exceptional C++** (Herb Sutter) - Exception safety patterns
