---
name: cpp-exceptions
description: C++编码规范 - 异常处理规范。适用于所有C++代码开发场景：(1) 编写try/catch代码；(2) 代码审查时检查异常处理；(3) 自定义异常类；(4) RAII资源管理。触发关键词：C++、异常、try、catch、throw、exception、RAII、智能指针
---

# C++异常处理规范

基于C++ Core Guidelines

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 使用RAII管理所有资源 |
| 规则1.2 | 优先使用智能指针而非裸指针 |
| 规则1.3 | 不要在析构函数中抛出异常 |
| 规则1.4 | 按引用捕获异常 |
| 规则1.5 | 异常安全保证（基本/强/不抛出） |

## RAII示例

```cpp
class FileHandle {
public:
    explicit FileHandle(const std::string& path)
        : file_(std::fopen(path.c_str(), "r"))
    {
        if (!file_) {
            throw std::runtime_error("无法打开文件: " + path);
        }
    }

    ~FileHandle() {
        if (file_) {
            std::fclose(file_);
        }
    }

    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    FILE* get() const { return file_; }

private:
    FILE* file_;
};
```

## 智能指针

```cpp
// 使用unique_ptr管理独占所有权
auto user = std::make_unique<User>("Alice");

// 使用shared_ptr管理共享所有权
auto repo = std::make_shared<Repository>();

// 使用weak_ptr打破循环引用
std::weak_ptr<Observer> weak_observer;
```

## 详细规范

见 [references/exceptions.md](references/exceptions.md)
