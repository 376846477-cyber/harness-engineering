# 华为Java可读性规范 - 详细版

基于华为Clean Code指导书第2章"可读"特征

## 1. 传递足够的有用信息

### 1.1 利用名称传递信息

好的名字可以自注释，可以承载更多信息。

**原则**：
- 函数名体现行为和意图
- 变量名带重要细节后缀
- 大作用域使用长名字，小作用域使用短名字

**✅ 正确示例**:
```java
// 函数名体现行为和意图
public User findUserById(Long userId) { ... }
public boolean validatePassword(String rawPassword, String encryptedPassword) { ... }

// 变量名带重要细节后缀
long timeoutMs = 5000;
BigDecimal amountInCents = new BigDecimal("9999");

// 大作用域使用长名字
private Map<ApplicationId, Map<ServiceName, List<InstanceInfo>>> globalServiceRegistry;

// 小作用域使用短名字
for (int i = 0; i < 10; i++) { ... }
```

**❌ 错误示例**:
```java
// 名字无法传递信息
public Object get(Object o) { ... }
int x;  // 作用域大但名字太短
```

### 1.2 利用格式传递信息

空行表示逻辑独立或函数间隔离，空格表示信息分隔，缩进表示嵌套。

**✅ 正确示例**:
```java
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    public User createUser(CreateUserRequest request) {
        validateRequest(request);

        User user = new User();
        user.setName(request.getName());
        user.setEmail(request.getEmail());
        user.setPassword(passwordEncoder.encode(request.getPassword()));

        return userRepository.save(user);
    }

    private void validateRequest(CreateUserRequest request) {
        if (request == null) {
            throw new IllegalArgumentException("请求不能为空");
        }
    }
}
```

### 1.3 注释作为代码的补充

优先通过代码传递信息，针对可读性差的优先通过重构让代码变得可读，注释作为最后选择。

**✅ 正确示例**:
```java
/**
 * 发送验证码短信
 *
 * @param phoneNumber 手机号
 * @param code 验证码
 * @throws SmsException 短信发送失败时抛出
 */
public void sendVerificationCode(String phoneNumber, String code) {
    // 业务解释：短信验证码有效期5分钟
    // 技术说明：使用第三方短信平台，失败会自动重试3次
    SmsRequest request = new SmsRequest(phoneNumber, code);
    smsGateway.send(request);
}
```

**❌ 错误示例**:
```java
// 很快能从代码推断的信息不需要注释
int i = i + 1;  // i加1

// 与代码不同步的错误注释
// 注释说返回用户列表，实际返回用户数量
public List<User> getUsers() {
    return userRepository.count();
}
```

## 2. 减少信息失真（误解）

### 2.1 业务术语统一

业务术语在团队内统一定义，避免术语不一致导致理解错误。

**✅ 正确示例**:
```java
// 使用统一术语：订单状态
public enum OrderStatus {
    PENDING_PAYMENT,    // 待支付
    PAID,               // 已支付
    SHIPPED,            // 已发货
    COMPLETED,          // 已完成
    CANCELLED           // 已取消
}
```

### 2.2 命名准确具体

准确具体的名称可以减少读者误解，布尔值加上is/has/can/should等前缀更明确。

**✅ 正确示例**:
```java
boolean needPassword = true;    // 需要密码
boolean isAuthenticated = true; // 已认证
boolean canAccess = true;       // 可以访问
boolean shouldRetry = true;     // 应该重试
```

**❌ 错误示例**:
```java
boolean readPassword = true;  // 两种理解：需要读取密码 vs 已经读取了密码
boolean enabled = true;       // 什么被启用？不明确
```

## 3. 减少信息干扰

### 3.1 无废弃代码和无效注释

废弃代码和无效注释会增加理解成本。

**✅ 正确示例**:
```java
public void processOrder(Order order) {
    validateOrder(order);
    calculateTotal(order);
    saveOrder(order);
}
```

**❌ 错误示例**:
```java
public void processOrder(Order order) {
    // 这个方法暂时不用
    // validateOrder(order);
    calculateTotal(order);
    // saveOrder(order);  // TODO: 稍后实现
    saveOrder(order);
}
```

### 3.2 风格一致

选择一种风格并保持一致，比"正确"的风格更重要。

**✅ 正确示例**:
```java
// 统一使用K&R风格
if (condition) {
    doSomething();
} else {
    doOtherThing();
}

// 统一使用大括号
public void method() {
    // ...
}
```

## 4. 降低信息理解难度

### 4.1 功能单一

一段代码如果同时处理几件事情，会使逻辑变得复杂。

**✅ 正确示例**:
```java
// 拆分成单一职责的函数
public void processUserRegistration(UserRegistrationRequest request) {
    validateRequest(request);
    createUser(request);
    sendWelcomeEmail(request.getEmail());
    logRegistration(request);
}

private void validateRequest(UserRegistrationRequest request) { ... }
private void createUser(UserRegistrationRequest request) { ... }
private void sendWelcomeEmail(String email) { ... }
private void logRegistration(UserRegistrationRequest request) { ... }
```

**❌ 错误示例**:
```java
// 一个方法处理所有事情
public void processUserRegistration(UserRegistrationRequest request) {
    // 验证
    if (request.getName() == null) throw new IllegalArgumentException();
    if (!request.getEmail().contains("@")) throw new IllegalArgumentException();
    // 创建用户
    User user = new User();
    user.setName(request.getName());
    user.setEmail(request.getEmail());
    userRepository.save(user);
    // 发送邮件
    emailService.send(request.getEmail(), "欢迎");
    // 记录日志
    logger.info("用户注册: " + request.getName());
}
```

### 4.2 简化语句逻辑

减少嵌套层级，使用早返回（保护语句），简化复杂表达式。

**✅ 正确示例**:
```java
// 使用保护语句减少嵌套
public User getUser(Long userId) {
    if (userId == null) {
        return null;
    }

    User user = userRepository.findById(userId);
    if (user == null) {
        return null;
    }

    if (!user.isActive()) {
        return null;
    }

    return user;
}

// 解释性变量让代码文档化
boolean isEligibleForDiscount =
    user.isPremium()
    && order.getTotalAmount().compareTo(new BigDecimal("100")) > 0
    && !user.hasActivePromotion();

if (isEligibleForDiscount) {
    applyDiscount(order);
}
```

**❌ 错误示例**:
```java
// 嵌套过深
public User getUser(Long userId) {
    if (userId != null) {
        User user = userRepository.findById(userId);
        if (user != null) {
            if (user.isActive()) {
                return user;
            }
        }
    }
    return null;
}
```

### 4.3 减少变量影响范围

变量个数尽量少，变量改变频率尽量低，变量作用域尽量小。

**✅ 正确示例**:
```java
// 使用const/final让变量只设置一次
private static final int MAX_RETRY_COUNT = 3;

// 在最小作用域内声明变量
public void process() {
    if (condition) {
        Result result = compute();
        use(result);
    }
    // result在这里不可见
}
```

### 4.4 名字不要过长

命名要准确具体，但名字也不能过长。名字太长说明功能不单一，需要拆分。

**✅ 正确示例**:
```java
// 拆分后的简洁命名
class NavigationController { }
class DataSourceWrapper { }

// 或者使用组合
class WrappingViewController { }
```

**❌ 错误示例**:
```java
// 名字过长
newNavigationControllerWrappingViewControllerForDataSourceOfClass
```

## 完整检查清单

- [ ] 名称是否传递了足够的信息？
- [ ] 名称是否有二义性？
- [ ] 代码格式是否一致？
- [ ] 是否有废弃代码和无效注释？
- [ ] 函数是否功能单一？
- [ ] 嵌套层级是否过深（>3层）？
- [ ] 变量作用域是否尽可能小？
- [ ] 名字长度是否合适？