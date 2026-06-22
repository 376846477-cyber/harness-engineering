---
name: cpp-concurrency
description: C++编码规范 - 并发规范。适用于所有C++代码开发场景：(1) 编写多线程代码；(2) 共享变量访问控制；(3) 死锁预防；(4) 异步编程。触发关键词：C++、线程、thread、mutex、atomic、并发、concurrent、锁、lock、future、async
---

# C++并发规范

基于C++ Core Guidelines

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 共享变量必须同步访问 |
| 规则1.2 | 优先使用std::atomic而非mutex（简单场景） |
| 规则1.3 | 避免死锁：固定加锁顺序 |
| 规则1.4 | 使用std::lock_guard/std::unique_lock管理锁 |
| 规则1.5 | 优先使用消息传递而非共享内存 |

## 线程安全示例

```cpp
class SafeCounter {
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mutex_);
        ++count_;
    }

    int64_t get() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return count_;
    }

private:
    mutable std::mutex mutex_;
    int64_t count_ = 0;
};
```

## 使用atomic

```cpp
class AtomicFlag {
public:
    void set() { flag_.store(true, std::memory_order_release); }
    bool check() const { return flag_.load(std::memory_order_acquire); }

private:
    std::atomic<bool> flag_{false};
};
```

## 详细规范

见 [references/concurrency.md](references/concurrency.md)
