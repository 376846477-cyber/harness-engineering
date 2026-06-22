---
name: cpp-performance
description: C++编码规范 - 高效性规范。适用于所有C++代码开发场景：(1) 代码编写时考虑性能；(2) 代码审查时检查性能；(3) 算法、内存管理、移动语义优化。触发关键词：C++、高效、performance、性能、优化、算法、移动语义、内存、零拷贝
---

# C++高效性规范

基于Clean Code指导书

## 核心原则

在不影响其他代码质量属性的前提下，养成编写高效代码的习惯，**不要过早做性能优化**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 选择高效的数据结构 |
| 1.2 | 使用移动语义避免拷贝 |
| 1.3 | 减少冗余或无效计算 |
| 1.4 | 合理使用缓存 |
| 2.1 | 减少内存分配 |
| 3.1 | 减少IO次数 |
| 4.1 | 利用编译器优化 |

## 移动语义

```cpp
class UserService {
public:
    // 移动构造
    UserService(UserService&& other) noexcept
        : users_(std::move(other.users_))
        , cache_(std::move(other.cache_)) {}

    // 移动赋值
    UserService& operator=(UserService&& other) noexcept {
        if (this != &other) {
            users_ = std::move(other.users_);
            cache_ = std::move(other.cache_);
        }
        return *this;
    }

    // 按值传参+移动
    void add_user(User user) {
        users_.push_back(std::move(user));
    }

private:
    std::vector<User> users_;
    std::unordered_map<int, User> cache_;
};
```

## 数据结构选择

```cpp
// O(1)查找
std::unordered_map<int, User> user_map;

// 有序数据
std::map<int, User> sorted_users;

// 频繁插入删除
std::list<User> user_list;

// 栈操作
std::vector<User> user_stack;
```

## 详细规范

见 [references/performance.md](references/performance.md)

## 检查清单

- [ ] 是否使用了合适的数据结构？
- [ ] 是否使用了移动语义？
- [ ] 是否有重复计算可以缓存？
- [ ] 是否有不必要的内存分配？
- [ ] 是否利用了编译器优化？
