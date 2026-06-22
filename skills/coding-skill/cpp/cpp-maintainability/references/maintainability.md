# C++可维护性规范详细参考

## 一、模块化设计

### 原则
将系统分解为独立、可替换的模块，每个模块具有明确职责和清晰接口。

### 最佳实践

```cpp
// 模块示例：日志模块 (logger.hpp)
namespace logger {

class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(LogLevel level, std::string_view message) = 0;
};

class FileLogger : public ILogger {
public:
    explicit FileLogger(std::string_view filepath);
    void log(LogLevel level, std::string_view message) override;
private:
    std::ofstream file_;
};

} // namespace logger
```

### 模块划分原则
- 高内聚：模块内部元素紧密相关
- 低耦合：模块间依赖最小化
- 单一职责：每个模块只做一件事
- 接口稳定：公共API不频繁变动

---

## 二、接口与实现分离

### 头文件规则

```cpp
// widget.hpp - 仅声明
class Widget {
public:
    Widget();
    ~Widget();
    void process(int value);
    int getResult() const;
private:
    class Impl;
    std::unique_ptr<Impl> pimpl_;
};

// widget.cpp - 具体实现
class Widget::Impl {
public:
    void process(int value) { /* 实现 */ }
    int getResult() const { return result_; }
private:
    int result_ = 0;
};

Widget::Widget() : pimpl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;
void Widget::process(int value) { pimpl_->process(value); }
int Widget::getResult() const { return pimpl_->getResult(); }
```

### 分离的好处
- 减少编译依赖
- 隐藏实现细节
- 加快编译速度
- 保持ABI兼容

---

## 三、SOLID原则

### 单一职责原则 (SRP)
一个类应该只有一个引起它变化的原因。

```cpp
// 违反SRP
class User {
    void save() { /* 保存到数据库 */ }
    void sendEmail() { /* 发送邮件 */ }
    void validate() { /* 验证数据 */ }
};

// 遵循SRP
class User { /* 仅数据模型 */ };
class UserRepository { void save(const User& user); };
class EmailService { void send(const User& user, std::string_view msg); };
class UserValidator { bool validate(const User& user); };
```

### 开闭原则 (OCP)
对扩展开放，对修改关闭。

```cpp
// 使用策略模式实现OCP
class PaymentStrategy {
public:
    virtual ~PaymentStrategy() = default;
    virtual void pay(double amount) = 0;
};

class CreditCardPayment : public PaymentStrategy {
    void pay(double amount) override { /* 信用卡支付 */ }
};

class AlipayPayment : public PaymentStrategy {
    void pay(double amount) override { /* 支付宝支付 */ }
};

class PaymentProcessor {
public:
    void setStrategy(std::unique_ptr<PaymentStrategy> strategy) {
        strategy_ = std::move(strategy);
    }
    void process(double amount) { strategy_->pay(amount); }
private:
    std::unique_ptr<PaymentStrategy> strategy_;
};
```

### 里氏替换原则 (LSP)
子类必须能替换其基类而不影响程序正确性。

```cpp
// 违反LSP
class Rectangle {
public:
    virtual void setWidth(int w) { width_ = w; }
    virtual void setHeight(int h) { height_ = h; }
    int getArea() const { return width_ * height_; }
protected:
    int width_, height_;
};

class Square : public Rectangle {
    void setWidth(int w) override { width_ = height_ = w; }  // 破坏约束
    void setHeight(int h) override { width_ = height_ = h; }
};
```

### 接口隔离原则 (ISP)
不应强迫客户依赖它不使用的方法。

```cpp
// 违反ISP
class IMachine {
    virtual void print() = 0;
    virtual void scan() = 0;
    virtual void fax() = 0;
};

// 遵循ISP
class IPrinter { virtual void print() = 0; };
class IScanner { virtual void scan() = 0; };
class IFax { virtual void fax() = 0; };

class AllInOne : public IPrinter, public IScanner, public IFax { /* ... */ };
class SimplePrinter : public IPrinter { /* 仅打印 */ };
```

### 依赖倒置原则 (DIP)
依赖抽象，不依赖具体实现。

```cpp
// 违反DIP
class UserService {
    MySQLDatabase db_;  // 直接依赖具体实现
public:
    void saveUser(const User& user) { db_.insert(user); }
};

// 遵循DIP
class IDatabase {
public:
    virtual ~IDatabase() = default;
    virtual void insert(const User& user) = 0;
};

class UserService {
    std::shared_ptr<IDatabase> db_;  // 依赖抽象
public:
    explicit UserService(std::shared_ptr<IDatabase> db) : db_(db) {}
    void saveUser(const User& user) { db_->insert(user); }
};
```

---

## 四、依赖注入

### 构造函数注入（推荐）

```cpp
class OrderProcessor {
public:
    explicit OrderProcessor(
        std::shared_ptr<IPaymentService> payment,
        std::shared_ptr<IInventoryService> inventory,
        std::shared_ptr<INotificationService> notification
    ) : payment_(payment), inventory_(inventory), notification_(notification) {}
    
    void process(const Order& order) {
        inventory_->reserve(order.items());
        payment_->charge(order.total());
        notification_->notify(order.customerId());
    }
    
private:
    std::shared_ptr<IPaymentService> payment_;
    std::shared_ptr<IInventoryService> inventory_;
    std::shared_ptr<INotificationService> notification_;
};
```

### Setter注入

```cpp
class ReportGenerator {
public:
    void setDataLoader(std::shared_ptr<IDataLoader> loader) {
        dataLoader_ = loader;
    }
private:
    std::shared_ptr<IDataLoader> dataLoader_;
};
```

---

## 五、避免全局变量

### 问题

```cpp
// 全局状态 - 难以测试、难以追踪
int g_counter = 0;
Database* g_db = nullptr;

void process() {
    g_counter++;
    g_db->query("...");
}
```

### 解决方案

```cpp
// 使用依赖注入
class Counter {
public:
    void increment() { ++count_; }
    int get() const { return count_; }
private:
    int count_ = 0;
};

class Processor {
public:
    Processor(Counter& counter, IDatabase& db)
        : counter_(counter), db_(db) {}
    void process() {
        counter_.increment();
        db_.query("...");
    }
private:
    Counter& counter_;
    IDatabase& db_;
};
```

---

## 六、使用命名空间

### 组织策略

```cpp
namespace company::project::module {

class ClassName {
    // ...
};

namespace detail {
    // 内部实现细节
    void internalHelper();
}

} // namespace company::project::module

// 使用
using company::project::module::ClassName;
```

### 命名空间规则
- 使用嵌套命名空间表示层次结构
- 避免在头文件使用 `using namespace`
- 匿名命名空间替代 `static` 函数
- `.cpp` 文件可用 `using` 简化代码

```cpp
// 匿名命名空间
namespace {
    void localFunction() { /* 仅本编译单元可见 */ }
}
```

---

## 七、头文件组织

### 头文件结构

```cpp
#ifndef COMPANY_PROJECT_MODULE_HPP
#define COMPANY_PROJECT_MODULE_HPP

// 1. 标准库
#include <memory>
#include <string>
#include <vector>

// 2. 第三方库
#include <boost/optional.hpp>

// 3. 项目内头文件
#include "company/project/base.hpp"

// 4. 前向声明
namespace external { class ExternalClass; }

// 5. 命名空间开始
namespace company::project {

// 6. 类声明
class Module {
public:
    // 公共接口
private:
    // 实现细节
    std::unique_ptr<external::ExternalClass> ext_;
};

} // namespace company::project

#endif // COMPANY_PROJECT_MODULE_HPP
```

### 最佳实践
- 使用 `#pragma once` 或传统include guard
- 最小化头文件依赖，优先前向声明
- 头文件自包含（能独立编译）
- 按稳定度排序include

---

## 八、Pimpl模式

### 完整实现

```cpp
// processor.hpp
#pragma once
#include <memory>

namespace app {

class Processor {
public:
    Processor();
    ~Processor();
    
    Processor(Processor&&) noexcept;
    Processor& operator=(Processor&&) noexcept;
    
    void process(int value);
    int getResult() const;
    
private:
    class Impl;
    std::unique_ptr<Impl> pimpl_;
};

} // namespace app

// processor.cpp
#include "processor.hpp"
#include <vector>
#include <algorithm>  // 这些依赖对用户隐藏

class app::Processor::Impl {
public:
    void process(int value) {
        data_.push_back(value);
    }
    int getResult() const {
        return std::accumulate(data_.begin(), data_.end(), 0);
    }
private:
    std::vector<int> data_;
};

app::Processor::Processor() : pimpl_(std::make_unique<Impl>()) {}
app::Processor::~Processor() = default;
app::Processor::Processor(Processor&&) noexcept = default;
app::Processor& app::Processor::operator=(Processor&&) noexcept = default;

void app::Processor::process(int value) { pimpl_->process(value); }
int app::Processor::getResult() const { return pimpl_->getResult(); }
```

### Pimpl优势
- 编译防火墙：修改实现不触发重编译
- ABI兼容：修改实现不改变二进制接口
- 隐藏实现细节：减少头文件暴露

---

## 九、模板元编程规范

### 9.1 基本概念

模板元编程（Template Metaprogramming, TMP）是在编译期执行计算的编程范式，可以提升运行时性能，但会增加编译时间和代码复杂度。

**原则：优先使用普通代码，必要时才使用模板元编程。**

### 9.2 类型萃取（Type Traits）

**规则9.2.1：使用标准库类型萃取**

```cpp
#include <type_traits>

// 检查类型属性
static_assert(std::is_integral<int>::value);
static_assert(std::is_pointer<int*>::value);
static_assert(std::is_class<std::string>::value);

// 条件编译
template<typename T>
void process(T value) {
    if constexpr (std::is_integral_v<T>) {
        std::cout << "Integral: " << value << "\n";
    } else if constexpr (std::is_floating_point_v<T>) {
        std::cout << "Floating: " << std::setprecision(10) << value << "\n";
    } else if constexpr (std::is_pointer_v<T>) {
        std::cout << "Pointer: " << *value << "\n";
    }
}

// 移除/添加类型修饰
static_assert(std::is_same_v<std::remove_pointer_t<int*>, int>);
static_assert(std::is_same_v<std::add_const_t<int>, const int>);
static_assert(std::is_same_v<std::remove_reference_t<int&>, int>);
```

**规则9.2.2：自定义类型萃取**

```cpp
// 检测类是否有特定成员函数
template<typename T, typename = void>
struct has_size : std::false_type {};

template<typename T>
struct has_size<T, std::void_t<decltype(std::declval<T>().size())>> 
    : std::true_type {};

// 使用
template<typename Container>
auto get_size(const Container& c) {
    if constexpr (has_size<Container>::value) {
        return c.size();
    } else {
        return std::distance(c.begin(), c.end());
    }
}

// 检测是否可迭代
template<typename T, typename = void>
struct is_iterable : std::false_type {};

template<typename T>
struct is_iterable<T, std::void_t<
    decltype(std::declval<T>().begin()),
    decltype(std::declval<T>().end())
>> : std::true_type {};

static_assert(is_iterable<std::vector<int>>::value);
static_assert(!is_iterable<int>::value);
```

### 9.3 SFINAE（替换失败非错误）

**规则9.3.1：使用SFINAE实现条件约束**

```cpp
#include <type_traits>

// C++17: std::enable_if_t 方式
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
add_one(T value) {
    return value + 1;
}

template<typename T>
std::enable_if_t<std::is_floating_point_v<T>, T>
add_one(T value) {
    return value + 1.0;
}

// C++17: void_t 检测成员
template<typename T, typename = void>
struct has_to_string : std::false_type {};

template<typename T>
struct has_to_string<T, std::void_t<
    decltype(std::declval<T>().to_string())
>> : std::true_type {};

// 使用SFINAE选择实现
template<typename T>
std::string convert_to_string(const T& value) {
    if constexpr (has_to_string<T>::value) {
        return value.to_string();
    } else if constexpr (std::is_arithmetic_v<T>) {
        return std::to_string(value);
    } else {
        return static_cast<std::ostringstream&>(
            std::ostringstream() << value).str();
    }
}
```

### 9.4 C++20 Concepts

**规则9.4.1：优先使用Concepts替代SFINAE**

```cpp
#include <concepts>
#include <ranges>

// 定义概念
template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

template<typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
};

template<typename T>
concept Printable = requires(std::ostream& os, T value) {
    { os << value } -> std::same_as<std::ostream&>;
};

// 使用概念约束模板
template<Numeric T>
T add(T a, T b) {
    return a + b;
}

template<Addable T>
T sum(const std::vector<T>& values) {
    T result{};
    for (const auto& v : values) {
        result = result + v;
    }
    return result;
}

// 更复杂的概念
template<typename T>
concept Container = requires(T c) {
    typename T::value_type;
    typename T::iterator;
    { c.begin() } -> std::same_as<typename T::iterator>;
    { c.end() } -> std::same_as<typename T::iterator>;
    { c.size() } -> std::convertible_to<std::size_t>;
};

template<Container C>
void print_container(const C& container) {
    for (const auto& item : container) {
        std::cout << item << " ";
    }
    std::cout << "\n";
}

// requires子句
template<typename T>
    requires std::copyable<T> && std::equality_comparable<T>
bool find_and_remove(std::vector<T>& vec, const T& value) {
    auto it = std::find(vec.begin(), vec.end(), value);
    if (it != vec.end()) {
        vec.erase(it);
        return true;
    }
    return false;
}
```

### 9.5 编译期计算

**规则9.5.1：使用constexpr替代模板元编程**

```cpp
// 传统TMP方式（不推荐，仅演示）
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N-1>::value;
};

template<>
struct Factorial<0> {
    static constexpr int value = 1;
};

// C++14/17 constexpr方式（推荐）
constexpr int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i) {
        result *= i;
    }
    return result;
}

// 编译期计算
static_assert(factorial(5) == 120);
static_assert(factorial(10) == 3628800);

// 编译期类型列表
template<typename... Types>
struct TypeList {};

using MyTypes = TypeList<int, float, double, std::string>;

// 编译期类型计数
template<typename List>
struct TypeCount;

template<typename... Types>
struct TypeCount<TypeList<Types...>> {
    static constexpr std::size_t value = sizeof...(Types);
};

static_assert(TypeCount<MyTypes>::value == 4);
```

### 9.6 可变参数模板

**规则9.6.1：正确使用参数包展开**

```cpp
// 基本参数包展开
template<typename... Args>
void print_all(Args&&... args) {
    (std::cout << ... << args) << "\n";  // C++17 折叠表达式
}

// 递归展开（C++14及之前）
template<typename T>
void print(T value) {
    std::cout << value << "\n";
}

template<typename T, typename... Args>
void print(T first, Args... rest) {
    std::cout << first << ", ";
    print(rest...);  // 递归展开
}

// 使用折叠表达式（C++17推荐）
template<typename... Args>
auto sum_all(Args... args) {
    return (args + ...);  // 一元右折叠
}

template<typename... Args>
auto product_all(Args... args) {
    return (... * args);  // 一元左折叠
}

// 完美转发参数包
template<typename... Args>
auto make_vector(Args&&... args) {
    std::vector<std::common_type_t<Args...>> result;
    result.reserve(sizeof...(args));
    (result.push_back(std::forward<Args>(args)), ...);  // 逗号折叠
    return result;
}

auto vec = make_vector(1, 2, 3, 4, 5);
```

### 9.7 CRTP（奇异递归模板模式）

**规则9.7.1：使用CRTP实现静态多态**

```cpp
// CRTP基类
template<typename Derived>
class Comparable {
public:
    bool operator>(const Derived& other) const {
        return other < static_cast<const Derived&>(*this);
    }
    
    bool operator<=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) > other);
    }
    
    bool operator>=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) < other);
    }
    
    bool operator==(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) < other) &&
               !(other < static_cast<const Derived&>(*this));
    }
};

// 派生类
class Integer : public Comparable<Integer> {
public:
    explicit Integer(int value) : value_(value) {}
    
    bool operator<(const Integer& other) const {
        return value_ < other.value_;
    }
    
private:
    int value_;
};

// 使用
Integer a(5), b(10);
bool result = (a < b);   // 直接调用
bool result2 = (a > b);  // CRTP生成
bool result3 = (a == b); // CRTP生成

// CRTP Mixin模式
template<typename Derived>
class Printable {
public:
    void print() const {
        static_cast<const Derived*>(this)->print_impl();
    }
};

class Person : public Printable<Person> {
public:
    Person(std::string name, int age) 
        : name_(std::move(name)), age_(age) {}
    
    void print_impl() const {
        std::cout << name_ << " (" << age_ << ")\n";
    }
    
private:
    std::string name_;
    int age_;
};
```

### 9.8 模板元编程最佳实践

**规则9.8.1：避免过度使用TMP**

```cpp
// ✗ 过度设计：简单函数使用复杂模板
template<typename T>
std::enable_if_t<std::is_integral_v<T> && sizeof(T) == 4, T>
add(T a, T b) { return a + b; }

// ✓ 简洁明了：使用普通函数或简单约束
template<std::integral T>
T add(T a, T b) { return a + b; }

// 或者更简单
int add(int a, int b) { return a + b; }
```

**规则9.8.2：提供清晰的错误信息**

```cpp
// 使用static_assert提供友好错误
template<typename T>
void serialize(const T& obj) {
    static_assert(std::is_trivially_copyable_v<T>,
        "serialize() requires trivially copyable types");
    // ...
}

// 使用concepts提供更好的错误提示
template<typename T>
concept Serializable = requires(T t) {
    { t.serialize() } -> std::convertible_to<std::string>;
    { T::deserialize(std::string{}) } -> std::same_as<T>;
};

template<Serializable T>
std::string to_json(const T& obj) {
    return obj.serialize();
}
```

**规则9.8.3：限制模板实例化**

```cpp
// 在.cpp文件中显式实例化常用模板
// widget.hpp
template<typename T>
class Widget { /* ... */ };

extern template class Widget<int>;
extern template class Widget<double>;

// widget.cpp
template class Widget<int>;
template class Widget<double>;
```

---

## 可维护性检查清单

### 模块设计
- [ ] 模块边界清晰，职责单一
- [ ] 模块间依赖关系明确且最小化
- [ ] 公共接口稳定，向后兼容

### 接口设计
- [ ] 接口与实现分离
- [ ] 使用纯虚类定义抽象接口
- [ ] 避免在头文件暴露实现细节

### SOLID遵循
- [ ] 每个类只有一个职责
- [ ] 通过扩展而非修改添加功能
- [ ] 子类可以替换基类使用
- [ ] 接口小而专注
- [ ] 依赖抽象而非具体实现

### 依赖管理
- [ ] 使用依赖注入而非硬编码依赖
- [ ] 避免全局变量和单例
- [ ] 管理依赖生命周期

### 代码组织
- [ ] 命名空间组织合理
- [ ] 头文件包含最小化
- [ ] 前向声明替代不必要的include
- [ ] 使用Pimpl隐藏实现

### 可测试性
- [ ] 组件可独立测试
- [ ] 外部依赖可mock/stub

### 模板元编程
- [ ] 优先使用普通代码，必要时才使用模板元编程
- [ ] 优先使用C++20 Concepts替代SFINAE
- [ ] 使用constexpr替代编译期模板计算
- [ ] 提供清晰的static_assert错误信息
- [ ] 避免过度复杂的模板设计
- [ ] 显式实例化常用模板以减少编译时间
- [ ] 无隐藏的全局状态
