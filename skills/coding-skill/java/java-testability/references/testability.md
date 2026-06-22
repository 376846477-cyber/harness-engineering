# 华为Java可测试性规范 - 详细版

基于华为Clean Code指导书第2章"可测试"特征

## 1. 可隔离（降低测试复杂度）

### 1.1 高内聚低耦合

子系统、模块、组件、子功能等要容易被隔离以降低测试复杂度，所以要满足高内聚低耦合原则。

**✅ 正确示例**:
```java
// 单一职责的类容易测试
public class UserValidator {
    public ValidationResult validate(User user) {
        if (user.getName() == null || user.getName().isEmpty()) {
            return ValidationResult.error("用户名不能为空");
        }
        if (user.getEmail() == null || !user.getEmail().contains("@")) {
            return ValidationResult.error("邮箱格式不正确");
        }
        return ValidationResult.ok();
    }
}

// 测试简单直接
@Test
void testValidate_validUser() {
    UserValidator validator = new UserValidator();
    User user = new User("张三", "zhangsan@example.com");

    ValidationResult result = validator.validate(user);

    assertTrue(result.isValid());
}
```

**❌ 错误示例**:
```java
// 职责过多的类难以测试
public class UserService {
    public void registerUser(String name, String email) {
        // 验证
        if (name == null) throw new IllegalArgumentException();
        // 数据库操作
        Connection conn = getConnection();
        // 发送邮件
        sendEmail(email);
        // 记录日志
        log();
        // 更新缓存
        updateCache();
    }
}

// 测试需要模拟数据库、邮件服务、缓存等，难以隔离
```

## 2. 可控制（测试前置条件可构造）

### 2.1 减少系统状态和全局变量

系统状态和全局变量不易构造、跟踪、查找，会增加测试构造的复杂度和难度。

**✅ 正确示例**:
```java
// 通过构造函数注入依赖
public class OrderService {
    private final OrderRepository repository;
    private final PaymentGateway paymentGateway;
    private final NotificationService notificationService;

    public OrderService(OrderRepository repository,
                       PaymentGateway paymentGateway,
                       NotificationService notificationService) {
        this.repository = repository;
        this.paymentGateway = paymentGateway;
        this.notificationService = notificationService;
    }

    public Order createOrder(OrderRequest request) {
        // 使用注入的依赖
        Order order = new Order(request);
        repository.save(order);
        paymentGateway.process(order);
        notificationService.notify(order);
        return order;
    }
}

// 测试时轻松注入mock
@Test
void testCreateOrder() {
    OrderRepository mockRepo = mock(OrderRepository.class);
    PaymentGateway mockPayment = mock(PaymentGateway.class);
    NotificationService mockNotify = mock(NotificationService.class);

    OrderService service = new OrderService(mockRepo, mockPayment, mockNotify);
    OrderRequest request = new OrderRequest("商品A", 100);

    Order result = service.createOrder(request);

    assertNotNull(result);
    verify(mockRepo).save(any(Order.class));
    verify(mockPayment).process(any(Order.class));
    verify(mockNotify).notify(any(Order.class));
}
```

**❌ 错误示例**:
```java
// 静态依赖全局状态，难以测试
public class OrderService {
    public Order createOrder(OrderRequest request) {
        Connection conn = DatabaseUtils.getConnection();  // 静态全局
        Cache cache = CacheManager.getInstance();          // 全局单例
        // ...
    }
}
```

### 2.2 支持依赖注入

对于必需的外部依赖，需要支持通过依赖注入的方式替换，以支撑对应状态下的测试。

**✅ 正确示例**:
```java
// 接口抽象便于mock
public interface Clock {
    long currentTimeMillis();
}

public class SystemClock implements Clock {
    @Override
    public long currentTimeMillis() {
        return System.currentTimeMillis();
    }
}

// 生产代码
public class TokenService {
    private final Clock clock;

    @Inject
    public TokenService(Clock clock) {
        this.clock = clock;
    }

    public String generateToken(User user) {
        long expireTime = clock.currentTimeMillis() + 3600000;
        return user.getId() + ":" + expireTime;
    }
}

// 测试代码 - 注入固定时间的clock
@Test
void testGenerateToken() {
    Clock fixedClock = () -> 1000000L;
    TokenService tokenService = new TokenService(fixedClock);
    User user = new User(1L, "test");

    String token = tokenService.generateToken(user);

    assertEquals("1:1360000", token);
}
```

### 2.3 减少输入数量

减少输入的数量，可以减少输入之间的组合数量，从而降低测试复杂度和测试成本。

**✅ 正确示例**:
```java
// 使用参数对象减少参数
public class CreateUserCommand {
    private final String name;
    private final String email;
    private final String phone;
    private final int age;

    // 构造方法或builder
}

public class UserService {
    public User createUser(CreateUserCommand command) {
        // 业务逻辑
    }
}

// 测试时只需要创建command对象
```

**❌ 错误示例**:
```java
// 参数过多
public User createUser(String name, String email, String phone,
                      int age, String address, String country,
                      String language, String timezone) {
    // ...
}

// 测试时需要构造大量参数
```

## 3. 可观测（预期过程与结果可观测）

### 3.1 结果可观测

结果容易观测，如无法直接观测，需要通过间接手段进行转换。

**✅ 正确示例**:
```java
// 提供结果查询接口
public class OrderService {
    public Order createOrder(OrderRequest request) {
        Order order = doCreate(request);
        return order;
    }

    public Order getOrder(String orderId) {
        return orderRepository.findById(orderId);
    }
}

// 测试
@Test
void testCreateOrder() {
    Order order = orderService.createOrder(request);

    // 直接验证
    assertNotNull(order.getId());

    // 通过查询验证
    Order saved = orderService.getOrder(order.getId());
    assertEquals(order.getId(), saved.getId());
}
```

### 3.2 过程可观测

关键过程数据集合可以系统外部获得，比如通过查询接口获取过程数据。

**✅ 正确示例**:
```java
public class DataProcessService {
    // 保留关键过程数据
    private final List<ProcessRecord> processRecords = new CopyOnWriteArrayList<>();

    public void process(Data data) {
        ProcessRecord record = new ProcessRecord();
        record.setStartTime(System.currentTimeMillis());
        try {
            doProcess(data);
            record.setStatus(Status.SUCCESS);
        } catch (Exception e) {
            record.setStatus(Status.FAILED);
            record.setErrorMessage(e.getMessage());
        } finally {
            record.setEndTime(System.currentTimeMillis());
            processRecords.add(record);
        }
    }

    // 提供查询接口
    public List<ProcessRecord> getProcessRecords() {
        return Collections.unmodifiableList(processRecords);
    }
}

// 测试可以验证过程数据
@Test
void testProcess() {
    service.process(data);

    List<ProcessRecord> records = service.getProcessRecords();
    assertEquals(1, records.size());
    assertEquals(Status.SUCCESS, records.get(0).getStatus());
}
```

## 4. 可定位（问题容易被定位）

### 4.1 异常输出日志

问题检测后如果能自愈的则自愈，并输出日志用于改进。日志尽量精简，做到刚刚好够用。

**✅ 正确示例**:
```java
public class PaymentService {

    public PaymentResult pay(Order order) {
        try {
            return gateway.pay(order);
        } catch (PaymentException e) {
            // 关键信息日志
            log.warn("支付失败，订单ID: {}, 错误: {}", order.getId(), e.getMessage());

            // 尝试降级处理
            return tryFallbackPayment(order);
        }
    }

    private PaymentResult tryFallbackPayment(Order order) {
        try {
            return fallbackGateway.pay(order);
        } catch (PaymentException e) {
            // 记录详细错误信息用于定位
            log.error("支付降级也失败，订单ID: {}, 渠道: fallback, 错误: {}",
                     order.getId(), e.getMessage(), e);
            return PaymentResult.fail("支付失败，请稍后重试");
        }
    }
}
```

**❌ 错误示例**:
```java
// 日志太多或太少
public void process() {
    log.info("开始处理");           // 太多无意义的日志
    log.info("验证输入");
    log.info("验证完成");
    log.info("开始业务处理");
    // ... 几十行日志

    try {
        // 业务逻辑
    } catch (Exception e) {
        // 没有日志，难以定位问题
    }
}
```

## 完整检查清单

- [ ] 类是否单一职责（高内聚低耦合）？
- [ ] 是否使用依赖注入（避免静态全局依赖）？
- [ ] 外部依赖是否抽象成接口便于mock？
- [ ] 输入参数是否过多（考虑参数对象）？
- [ ] 结果是否方便验证？
- [ ] 关键过程数据是否可查询？
- [ ] 异常是否有适当的日志记录？