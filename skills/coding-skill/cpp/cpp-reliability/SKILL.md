---
name: cpp-reliability
description: C++编码规范 - 可靠性规范。适用于所有C++代码开发场景：(1) 代码编写时考虑可靠性；(2) 代码审查时检查可靠性；(3) 防御式编程、容错、自愈机制实现。触发关键词：C++、可靠、reliability、容错、自愈、防御式编程、RAII、故障恢复
---

# C++可靠性规范

基于Clean Code指导书

## 核心原则

系统必须预备**预防错误、容错、故障恢复（自愈）**的能力。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | RAII管理所有资源 |
| 1.2 | 操作防呆设计 |
| 1.3 | 系统过载保护 |
| 2.1 | 防御式编程 |
| 2.2 | 核心流程依赖最小化 |
| 2.3 | 降级处理策略 |

## 防御式编程

```cpp
std::optional<User> find_user(int64_t id) {
    if (id <= 0) {
        return std::nullopt;
    }

    auto user = repo_->find(id);
    if (!user) {
        LOG(WARNING) << "用户不存在: " << id;
        return std::nullopt;
    }

    return user;
}
```

## RAII资源管理

```cpp
class ScopedTimer {
public:
    explicit ScopedTimer(const std::string& name)
        : name_(name), start_(std::chrono::steady_clock::now()) {}

    ~ScopedTimer() {
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start_);
        LOG(INFO) << name_ << " 耗时: " << ms.count() << "ms";
    }

    ScopedTimer(const ScopedTimer&) = delete;
    ScopedTimer& operator=(const ScopedTimer&) = delete;

private:
    std::string name_;
    std::chrono::steady_clock::time_point start_;
};
```

## 详细规范

见 [references/reliability.md](references/reliability.md)

## 检查清单

- [ ] 资源是否使用RAII管理？
- [ ] 关键操作是否有防呆设计？
- [ ] 是否有降级处理策略？
- [ ] 异常是否正确处理？
