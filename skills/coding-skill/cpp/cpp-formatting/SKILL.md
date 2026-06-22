---
name: cpp-formatting
description: C++编码规范 - 排版格式规范。适用于所有C++代码开发场景：(1) 编写C++代码时的格式检查；(2) 代码审查时检查缩进、空行；(3) 格式化代码；(4) include排序。触发关键词：C++、格式、缩进、空行、include、格式化、format、clang-format
---

# C++排版格式规范

基于C++ Core Guidelines和Google C++ Style Guide

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 缩进4空格，不使用制表符 |
| 规则1.2 | 行宽不超过120字符 |
| 规则1.3 | include按标准库/第三方/本项目排序 |
| 规则1.4 | 类间2空行，函数间1-2空行 |
| 规则1.5 | 大括号换行（Allman风格）或同行（K&R风格，团队统一） |

## include排序

```cpp
// 对应头文件
#include "user_service.h"

// C标准库
#include <cstdio>
#include <cstring>

// C++标准库
#include <algorithm>
#include <memory>
#include <string>
#include <vector>

// 第三方库
#include <glog/logging.h>
#include <gtest/gtest.h>

// 本项目其他头文件
#include "common/error_codes.h"
#include "storage/repository.h"
```

## 缩进示例

```cpp
class UserService {
public:
    UserService(std::shared_ptr<Repository> repo)
        : repo_(std::move(repo))
        , cache_(std::make_shared<Cache>())
    {
    }

    std::optional<User> find_by_id(int64_t id) {
        if (id <= 0) {
            return std::nullopt;
        }
        return repo_->find(id);
    }

private:
    std::shared_ptr<Repository> repo_;
    std::shared_ptr<Cache> cache_;
};
```

## 详细规范

见 [references/formatting.md](references/formatting.md)
