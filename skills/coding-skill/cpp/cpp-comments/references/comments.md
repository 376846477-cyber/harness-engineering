# C++注释规范详细参考

## 1. 文件头注释

每个头文件和源文件开头必须包含文件头注释，说明文件基本信息。

```cpp
/**
 * @file user_service.h
 * @brief 用户管理服务 - 提供用户注册、登录、权限管理功能
 * @author 开发团队
 * @date 2024-01-15
 * @version 2.1.0
 * 
 * @details
 * 本模块实现了用户管理的核心功能，包括：
 * - 用户注册与身份验证
 * - 会话管理与Token刷新
 * - 基于角色的访问控制(RBAC)
 * 
 * @note 线程安全：所有公共方法均为线程安全
 * @see AuthService 认证服务
 * @see PermissionManager 权限管理器
 */
```

**必需字段：** @file, @brief, @author, @date  
**可选字段：** @version, @details, @note, @see, @copyright

---

## 2. 类注释

类定义前必须有详细注释，说明类的职责、用法和注意事项。

```cpp
/**
 * @class UserService
 * @brief 用户管理服务，提供用户CRUD操作和权限验证
 * 
 * @details
 * UserService负责管理用户生命周期，从注册到注销的所有操作。
 * 支持多线程并发访问，内部使用读写锁保护共享资源。
 * 
 * @section usage 使用示例
 * @code
 * UserService service(dbConnector);
 * User user = service.getUserById(12345);
 * bool success = service.authenticate("user", "pass");
 * @endcode
 * 
 * @section dependencies 依赖关系
 * - IDatabase: 数据库访问接口
 * - ILogger: 日志记录接口
 * - ICache: 缓存接口（可选）
 * 
 * @note 使用前必须调用initialize()进行初始化
 * @warning 不要在构造函数中执行耗时操作
 * 
 * @see User 用户实体类
 * @see IUserRepository 用户仓储接口
 */
class UserService : public IService {
public:
    // ...
};
```

**模板类注释：**
```cpp
/**
 * @template T
 * @brief 线程安全的通用对象池
 * 
 * @tparam T 池中存储的对象类型，必须满足：
 *         - 可默认构造
 *         - 可移动构造
 *         - 析构函数不抛异常
 */
template<typename T>
class ObjectPool {
    // ...
};
```

---

## 3. 函数注释

### 3.1 基础函数注释

```cpp
/**
 * @brief 根据用户ID获取用户信息
 * 
 * @param userId 用户唯一标识符，必须为正整数
 * @return User 用户信息对象
 * @throw std::invalid_argument 当userId <= 0时抛出
 * @throw UserNotFoundException 当用户不存在时抛出
 * @throw DatabaseException 当数据库访问失败时抛出
 * 
 * @pre userId > 0
 * @post 返回的User对象包含完整用户信息
 * 
 * @note 此方法是线程安全的
 * @note 结果会被缓存30秒
 * 
 * @see createUser()
 * @see updateUser()
 */
User getUserById(int64_t userId);
```

### 3.2 重载函数注释

```cpp
/**
 * @brief 搜索用户
 * @param keyword 搜索关键词
 * @param limit 返回结果上限，默认10
 * @return 用户列表
 */
std::vector<User> searchUsers(const std::string& keyword, size_t limit = 10);

/**
 * @brief 搜索用户（带过滤条件）
 * @param keyword 搜索关键词
 * @param filter 过滤条件
 * @param limit 返回结果上限
 * @return 符合条件的用户列表
 * @see UserFilter
 */
std::vector<User> searchUsers(const std::string& keyword, 
                               const UserFilter& filter, 
                               size_t limit = 10);
```

### 3.3 虚函数注释

```cpp
/**
 * @brief 处理用户登录事件（可重写）
 * 
 * @param user 登录用户信息
 * @param context 登录上下文（IP、设备等）
 * @return 是否允许登录
 * 
 * @note 子类可重写此方法实现自定义登录验证逻辑
 * @note 默认实现检查用户状态和密码有效期
 */
virtual bool onUserLogin(const User& user, const LoginContext& context);
```

### 3.4 回调函数注释

```cpp
/**
 * @brief 用户状态变更回调函数类型
 * 
 * @param userId 变更的用户ID
 * @param oldStatus 原状态
 * @param newStatus 新状态
 * @param timestamp 状态变更时间戳
 */
using UserStatusCallback = std::function<void(int64_t userId,
                                               UserStatus oldStatus,
                                               UserStatus newStatus,
                                               int64_t timestamp)>;
```

---

## 4. Doxygen格式详解

### 4.1 常用标签速查表

| 标签 | 用途 | 示例 |
|------|------|------|
| @file | 文件说明 | @file main.cpp |
| @class | 类说明 | @class UserService |
| @brief | 简要描述 | @brief 用户管理服务 |
| @details | 详细描述 | @details 实现细节... |
| @param | 参数说明 | @param id 用户ID |
| @tparam | 模板参数 | @tparam T 元素类型 |
| @return | 返回值说明 | @return 用户对象 |
| @throw/@exception | 异常说明 | @throw std::runtime_error |
| @pre | 前置条件 | @pre ptr != nullptr |
| @post | 后置条件 | @post 资源已释放 |
| @see | 交叉引用 | @see processUser() |
| @note | 注意事项 | @note 线程安全 |
| @warning | 警告信息 | @warning 不要并发调用 |
| @deprecated | 弃用标记 | @deprecated 使用新方法 |
| @todo | 待办事项 | @todo 支持异步操作 |
| @code/@endcode | 代码示例 | @code ... @endcode |
| @example | 示例说明 | @example demo.cpp |
| @since | 版本引入 | @since 2.0.0 |

### 4.2 注释风格选择

```cpp
// 推荐：单行Doxygen注释（简洁）
/// @brief 获取用户数量
size_t getUserCount() const;

// 推荐：多行注释（详细）
/**
 * @brief 批量导入用户数据
 * 
 * @param users 用户数据列表
 * @param skipInvalid 是否跳过无效数据
 * @return 成功导入的用户数量
 * @throw ImportException 导入失败时抛出
 */
size_t importUsers(const std::vector<UserData>& users, bool skipInvalid = true);

// 成员变量行后注释
struct Config {
    std::string host;       ///< 服务器地址
    uint16_t port;          ///< 服务端口，默认8080
    size_t maxConnections;  ///< 最大连接数
    bool enableSSL;         ///< 是否启用SSL，默认true
};

// 枚举值注释
enum class HttpStatus {
    OK = 200,           ///< 请求成功
    Created = 201,      ///< 资源创建成功
    BadRequest = 400,   ///< 请求参数错误
    Unauthorized = 401, ///< 未授权
    NotFound = 404,     ///< 资源不存在
    ServerError = 500   ///< 服务器内部错误
};
```

---

## 5. 行内注释

### 5.1 原则

- **解释"为什么"，不解释"是什么"**
- **复杂逻辑必须有注释**
- **注释应位于代码上方或右侧**

```cpp
// 好的注释：解释原因
// 使用二分查找而非线性查找，因为用户列表已按ID排序
// 时间复杂度从O(n)优化到O(log n)
auto it = std::lower_bound(users.begin(), users.end(), userId, 
    [](const User& u, int64_t id) { return u.getId() < id; });

// 不好的注释：描述代码做了什么（代码本身已说明）
// 遍历用户列表  <-- 多余
for (const auto& user : users) {
    // ...
}

// 好的注释：说明业务逻辑
// 允许用户登录失败3次，第4次冻结账户30分钟
// 参考《账户安全规范》第3.2节
if (loginAttempts >= MAX_LOGIN_ATTEMPTS) {
    lockAccount(userId, std::chrono::minutes(30));
    return LoginResult::AccountLocked;
}
```

### 5.2 代码块注释

```cpp
// ============== 数据验证 ==============
if (user.name.empty()) {
    return ValidationError::EmptyName;
}
if (user.email.find('@') == std::string::npos) {
    return ValidationError::InvalidEmail;
}

// ============== 权限检查 ==============
if (!hasPermission(userId, Permission::Write)) {
    return PermissionError::AccessDenied;
}

// ============== 数据持久化 ==============
try {
    repository.save(user);
    cache.set(userKey, user, CACHE_TTL);
} catch (const DatabaseException& e) {
    logger.error("保存用户失败: {}", e.what());
    throw;
}
```

---

## 6. 注释与代码同步

### 6.1 同步检查规则

| 场景 | 同步要求 |
|------|----------|
| 修改函数签名 | 更新@param、@return |
| 添加新参数 | 添加@param说明 |
| 修改返回类型 | 更新@return说明 |
| 添加异常抛出 | 添加@throw说明 |
| 修改类职责 | 更新@brief和@details |
| 删除功能 | 删除相关注释 |
| 废弃接口 | 添加@deprecated，提供替代方案 |

### 6.2 @deprecated使用规范

```cpp
/**
 * @brief 获取用户名称（已弃用）
 * 
 * @deprecated 此方法已弃用，请使用getUserProfile().getName()
 * @param userId 用户ID
 * @return 用户名称
 * 
 * @warning 此方法将在v3.0.0移除
 */
[[deprecated("Use getUserProfile().getName() instead")]]
std::string getUserName(int64_t userId);

/**
 * @brief 获取用户完整档案
 * 
 * @param userId 用户ID
 * @return 用户档案信息
 * @since 2.5.0
 */
UserProfile getUserProfile(int64_t userId);
```

---

## 7. TODO/FIXME规范

### 7.1 格式规范

```cpp
// TODO(author): 简要描述 [优先级] [截止日期]
// FIXME(author): 问题描述 [严重程度]

// 标准TODO
// TODO(zhangsan): 添加连接池支持 [P2]
// TODO(lisi): 实现分布式缓存 [P3] [2024-Q2]

// 带上下文的TODO
// TODO(wangwu): 优化大文件上传性能
// 当前实现会占用大量内存，应改为流式处理
// 参考方案：https://example.com/streaming-upload

// FIXME示例
// FIXME(zhangsan): 并发情况下可能产生死锁 [High]
// 复现步骤：多线程同时调用lock()和release()
// 临时解决方案：使用超时锁
```

### 7.2 优先级定义

| 标记 | 优先级 | 说明 |
|------|--------|------|
| P0/Critical | 最高 | 阻塞发布的问题 |
| P1/High | 高 | 必须在当前版本解决 |
| P2/Medium | 中 | 计划在下一版本解决 |
| P3/Low | 低 | 时间允许时处理 |
| P4/Nice-to-have | 最低 | 改进项，可延后 |

### 7.3 FIXME使用场景

- **已知Bug但暂时无法修复**
- **临时代码需要重构**
- **平台兼容性问题**
- **性能问题待优化**

```cpp
// FIXME(zhangsan): Windows平台路径处理有误 [P1]
// 当前使用/分隔符，Windows需要\\分隔符
// 应使用std::filesystem::path统一处理

// FIXME(lisi): 内存泄漏，需要深入排查 [P0]
// Valgrind报告此处有8字节泄漏
// 暂时通过定期重启服务规避

// HACK(zhangsan): 绕过第三方库的bug
// 参考issue: https://github.com/lib/lib/issues/123
// 当第三方库修复后移除此代码
std::this_thread::sleep_for(std::chrono::milliseconds(100)); // 延时等待初始化
```

---

## 8. 检查清单

### 8.1 文件头注释检查

- [ ] 包含@file标签
- [ ] @brief描述清晰（一句话说明文件职责）
- [ ] @author和@date信息完整
- [ ] 复杂文件包含@details说明
- [ ] 相关类/函数的@see引用完整

### 8.2 类注释检查

- [ ] 包含@class和@brief
- [ ] 详细描述类职责和用法
- [ ] 包含使用示例（@code/@endcode）
- [ ] 说明线程安全性
- [ ] 标注依赖关系
- [ ] 模板类说明@tparam约束

### 8.3 函数注释检查

- [ ] 包含@brief描述
- [ ] 所有参数都有@param说明
- [ ] 有返回值时包含@return说明
- [ ] 可能抛出的异常有@throw说明
- [ ] 前置条件使用@pre标注
- [ ] 后置条件使用@post标注
- [ ] 特殊行为使用@note/@warning说明
- [ ] 相关函数使用@see引用

### 8.4 行内注释检查

- [ ] 注释解释"为什么"而非"是什么"
- [ ] 复杂算法有逻辑说明
- [ ] 业务规则有参考文档
- [ ] 无冗余注释
- [ ] 注释位置合理（上方或右侧）

### 8.5 同步性检查

- [ ] 函数签名修改后注释已更新
- [ ] 新增功能有对应注释
- [ ] 删除代码时删除相关注释
- [ ] @deprecated接口有替代方案说明
- [ ] TODO/FIXME有责任人标注

---

## 9. 最佳实践

### 9.1 自解释代码优先

```cpp
// 不好的做法：需要注释解释
int d; // 经过的天数

// 好的做法：变量名自解释
int elapsedDaysSinceLastLogin;

// 更好的做法：使用语义化类型
struct Days { int value; };
Days daysSinceLastLogin;
```

### 9.2 注释应提供价值

```cpp
// 无价值注释
i++; // i加1

// 有价值注释：解释业务规则
retryCount++;
// 重试超过3次后升级为人工处理
if (retryCount > MAX_AUTO_RETRY) {
    escalateToHuman(userId, reason);
}
```

### 9.3 使用英文还是中文

**推荐规则：**
- 代码标识符：英文
- 注释内容：与团队协商，保持一致
- TODO/FIXME：英文关键词 + 中文描述

```cpp
/// @brief 用户服务类
class UserService {
public:
    // TODO(zhangsan): 实现批量导入功能
    void batchImport(const std::vector<User>& users);
};
```

---

**文件版本：** v2.0  
**最后更新：** 2024-01-15  
**维护者：** C++编码规范团队
