# C++异常处理规范详细参考

## 异常安全保证

### 四级异常安全保证

| 等级 | 名称 | 保证内容 | 适用场景 |
|-----|------|---------|---------|
| 不抛出 | `noexcept` | 操作绝不抛出异常 | 析构函数、移动操作、swap |
| 强 | Strong | 操作失败时状态回滚到调用前 | 事务性操作、数据库操作 |
| 基本 | Basic | 操作失败时无资源泄露、对象有效 | 大多数常规操作 |
| 无 | None | 无任何保证 | 应避免 |

### 强异常安全示例

```cpp
class Container {
    std::vector<int> data_;
public:
    void push_back(int value) {
        std::vector<int> temp = data_;  // 先复制
        temp.push_back(value);          // 修改副本
        data_.swap(temp);               // 原子交换，不抛异常
    }
};
```

## RAII资源管理

### RAII核心原则

1. **构造函数获取资源**
2. **析构函数释放资源**
3. **禁止拷贝或实现深拷贝/移动语义**
4. **使用智能指针管理堆资源**

### RAII实现示例

```cpp
class FileHandle {
    FILE* file_;
public:
    explicit FileHandle(const char* filename)
        : file_(std::fopen(filename, "r")) {
        if (!file_) {
            throw std::runtime_error("Failed to open file");
        }
    }
    
    ~FileHandle() noexcept {
        if (file_) {
            std::fclose(file_);
        }
    }
    
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    
    FileHandle(FileHandle&& other) noexcept : file_(other.file_) {
        other.file_ = nullptr;
    }
    
    FILE* get() const noexcept { return file_; }
};
```

## 智能指针

### unique_ptr

- 独占所有权，不可拷贝
- 零开销抽象
- 适用于单一所有者场景

```cpp
std::unique_ptr<Widget> createWidget() {
    auto ptr = std::make_unique<Widget>();
    return ptr;  // 自动移动，不拷贝
}
```

### shared_ptr

- 共享所有权，引用计数
- 线程安全的引用计数
- 适用于多所有者场景

```cpp
std::shared_ptr<Resource> shared = std::make_shared<Resource>();
std::shared_ptr<Resource> another = shared;  // 引用计数+1
```

### weak_ptr

- 不增加引用计数
- 用于打破循环引用
- lock()返回shared_ptr

```cpp
std::weak_ptr<Node> parent_;
std::shared_ptr<Node> getParent() const {
    return parent_.lock();  // 返回shared_ptr，可能为空
}
```

## try/catch最佳实践

### 按引用捕获

```cpp
try {
    process();
} catch (const std::exception& e) {  // 正确：引用捕获，无切片
    log(e.what());
} catch (const BaseException& e) {    // 按派生到基类顺序
    handle(e);
} catch (...) {                       // 最后兜底
    log("Unknown exception");
    throw;  // 重新抛出
}
```

### 禁止做法

```cpp
try {
    process();
} catch (std::exception e) {  // 错误：值捕获导致对象切片
    handle(e);
}

catch (...) {  // 错误：捕获所有但不处理
    // 什么都不做
}

catch (BaseException& e) {  // 错误：顺序错误
    handle(e);
} catch (DerivedException& e) {  // 永远不会被匹配
    handle(e);
}
```

## 自定义异常类

### 基本自定义异常

```cpp
class DatabaseException : public std::runtime_error {
public:
    explicit DatabaseException(const std::string& msg)
        : std::runtime_error(msg) {}
    
    explicit DatabaseException(const char* msg)
        : std::runtime_error(msg) {}
};

class ConnectionException : public DatabaseException {
    std::string connectionId_;
public:
    ConnectionException(const std::string& msg, std::string connId)
        : DatabaseException(msg), connectionId_(std::move(connId)) {}
    
    const std::string& getConnectionId() const noexcept {
        return connectionId_;
    }
};
```

### 使用标准异常层次

| 基类 | 用途 |
|-----|------|
| `std::exception` | 所有异常基类 |
| `std::logic_error` | 逻辑错误（可避免） |
| `std::runtime_error` | 运行时错误 |
| `std::invalid_argument` | 参数无效 |
| `std::out_of_range` | 越界访问 |
| `std::bad_alloc` | 内存分配失败 |

## noexcept关键字

### 使用场景

```cpp
class Widget {
public:
    ~Widget() noexcept = default;  // 析构函数必须noexcept
    
    Widget(Widget&& other) noexcept  // 移动构造应noexcept
        : data_(std::move(other.data_)) {}
    
    Widget& operator=(Widget&& other) noexcept {
        data_ = std::move(other.data_);
        return *this;
    }
    
    void clear() noexcept {  // 不抛异常的操作
        data_.clear();
    }
    
    void process() noexcept(false);  // 明确标记可能抛异常
};
```

### 条件noexcept

```cpp
template<typename T>
void swap(T& a, T& b) noexcept(
    noexcept(std::is_nothrow_move_constructible<T>::value) &&
    noexcept(std::is_nothrow_move_assignable<T>::value)
) {
    T temp = std::move(a);
    a = std::move(b);
    b = std::move(temp);
}
```

## 异常与构造函数/析构函数

### 构造函数异常

```cpp
class ResourceManager {
    FileHandle file_;
    NetworkHandle network_;
public:
    ResourceManager(const std::string& path, const std::string& addr)
        : file_(path.c_str()),              // 构造，异常时file_已析构
          network_(addr) {                  // 构造，异常时全部已析构
    }
};
```

### 析构函数禁止抛出异常

```cpp
class BadExample {
public:
    ~BadExample() {
        closeFile();  // 如果抛出异常，程序可能终止
    }
};

class GoodExample {
public:
    ~GoodExample() noexcept {
        try {
            closeFile();
        } catch (const std::exception& e) {
            std::cerr << "Error in destructor: " << e.what() << '\n';
            // 吞掉异常，不要重新抛出
        } catch (...) {
            std::cerr << "Unknown error in destructor\n";
        }
    }
};
```

## 检查清单

### 代码审查检查项

- [ ] 所有析构函数标记为`noexcept`
- [ ] 移动构造/赋值标记为`noexcept`
- [ ] 使用`const&`捕获异常，避免对象切片
- [ ] catch块按派生到基类顺序排列
- [ ] 资源使用RAII包装，不裸露裸指针
- [ ] 自定义异常继承`std::exception`层次
- [ ] 析构函数内捕获所有异常不重新抛出
- [ ] 使用`make_unique/make_shared`创建智能指针
- [ ] 强异常安全操作使用copy-and-swap
- [ ] 不在析构函数中抛出异常

### 禁止事项

- 禁止析构函数抛出异常
- 禁止按值捕获异常
- 禁止空catch块吞掉异常不处理
- 禁止catch块顺序错误
- 禁止异常跨越模块边界（ABI兼容性）
- 禁止在 noexcept 函数中抛出异常（触发std::terminate）
