---
name: cpp-comments
description: C++编码规范 - 注释规范。适用于所有C++代码开发场景：(1) 编写Doxygen文档注释；(2) 代码审查时检查注释规范；(3) 添加函数注释、类注释。触发关键词：C++、注释、Doxygen、comment、doc、文档
---

# C++注释规范

基于Doxygen文档规范

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 公开头文件中的类和函数使用Doxygen注释 |
| 规则1.2 | 使用///或/** */风格 |
| 规则1.3 | 注释说明"为什么"而非"是什么" |
| 规则1.4 | 保持注释与代码同步 |

## Doxygen注释示例

### 类注释
```cpp
/// 用户管理服务。
///
/// 提供用户CRUD操作和业务逻辑处理。
class UserService {
public:
    /// 根据用户ID获取用户信息。
    ///
    /// @param id 用户唯一标识符
    /// @return 用户信息，不存在时返回std::nullopt
    /// @throws DatabaseError 数据库访问失败时抛出
    std::optional<User> find_by_id(int64_t id);

private:
    std::shared_ptr<Repository> repo_;  ///< 数据仓库实例
};
```

### 函数注释
```cpp
/// 搜索符合条件的用户列表。
///
/// @param criteria 搜索条件
/// @param limit 返回结果上限
/// @return 用户列表
std::vector<User> search(const SearchCriteria& criteria, size_t limit = 100);
```

## 详细规范

见 [references/comments.md](references/comments.md)
