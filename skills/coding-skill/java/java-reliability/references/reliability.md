# 华为Java可靠性规范 - 详细版

基于华为Clean Code指导书第2章"可靠性"特征

## 1. 系统输入可靠性

### 1.1 操作防呆设计

操作执行不当会导致系统异常，需要防呆设计。

**✅ 正确示例**:
```java
public class OperationService {

    public void executeCriticalOperation(OperationRequest request) {
        // 1. 确保操作风险被知悉
        if (request.isHighRisk()) {
            log.warn("高风险操作: {}", request.getType());
        }

        // 2. 准入检查
        if (!checkPrerequisites(request)) {
            throw new OperationDeniedException("不满足操作前置条件");
        }

        // 3. 最小化执行
        OperationContext context = buildMinimalContext(request);
        executeInternal(context);

        // 4. 误操作可恢复
        saveOperationHistory(request, OperationStatus.SUCCESS);
    }
}
```

### 1.2 系统过载保护

在系统过载时使用负载均衡和流控防止系统崩溃。

**✅ 正确示例**:
```java
public class RateLimiter {
    private final AtomicInteger currentRequests = new AtomicInteger(0);
    private final int maxRequests;

    public RateLimiter(int maxRequests) {
        this.maxRequests = maxRequests;
    }

    public boolean tryAcquire() {
        int current = currentRequests.get();
        if (current >= maxRequests) {
            // 流控：拒绝请求
            return false;
        }
        return currentRequests.incrementAndGet() <= maxRequests;
    }

    public void release() {
        currentRequests.decrementAndGet();
    }
}

// 使用
public class ApiService {
    private final RateLimiter rateLimiter = new RateLimiter(1000);

    public Response handle(Request request) {
        if (!rateLimiter.tryAcquire()) {
            return Response.tooManyRequests();
        }
        try {
            return doHandle(request);
        } finally {
            rateLimiter.release();
        }
    }
}
```

## 2. 系统间通信可靠性

### 2.1 消息重复发送

通信的丢失可以通过检测机制识别，通过重复发送消息来进行自愈。

**✅ 正确示例**:
```java
public class ReliableMessenger {
    private final Map<String, MessageContext> pendingMessages = new ConcurrentHashMap<>();

    public void sendWithRetry(String messageId, Message message, int maxRetries) {
        for (int attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                sendMessage(message);
                pendingMessages.remove(messageId);
                return;
            } catch (Exception e) {
                log.warn("消息发送失败，尝试 {}/{}", attempt, maxRetries, e);
                if (attempt == maxRetries) {
                    // 记录失败消息用于后续处理
                    pendingMessages.put(messageId, new MessageContext(message, attempt, e));
                }
            }
        }
    }
}
```

### 2.2 状态定时同步

多个物理节点维护同一状态时，需要定时同步防止状态失步。

**✅ 正确示例**:
```java
public class StateManager {
    private volatile State currentState = State.INITIAL;
    private final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();

    public void startStateSync(long intervalMs) {
        scheduler.scheduleAtFixedRate(this::syncState, intervalMs, intervalMs, TimeUnit.MILLISECONDS);
    }

    private synchronized void syncState() {
        State remoteState = fetchRemoteState();
        if (remoteState != currentState) {
            log.info("状态同步: {} -> {}", currentState, remoteState);
            currentState = remoteState;
            notifyStateChange(currentState);
        }
    }

    public State getState() {
        return currentState;
    }
}
```

## 3. 子系统内部可靠性

### 3.1 防御式编程

通过防御性的编码策略来弥补编码人员的潜在疏忽。

**✅ 正确示例**:
```java
public class Validator {

    public void processData(String input) {
        // 防御式检查
        if (input == null) {
            throw new IllegalArgumentException("输入不能为空");
        }

        input = input.trim();
        if (input.isEmpty()) {
            throw new IllegalArgumentException("输入不能为空字符串");
        }

        if (input.length() > MAX_LENGTH) {
            throw new IllegalArgumentException("输入长度超出限制");
        }

        // 业务处理
        doProcess(input);
    }

    // 对外暴露的方法要进行防御式检查
    public Result calculate(BigDecimal a, BigDecimal b) {
        if (a == null || b == null) {
            return Result.error("参数不能为空");
        }
        if (b.compareTo(BigDecimal.ZERO) == 0) {
            return Result.error("除数不能为零");
        }
        return Result.ok(a.divide(b, 2, RoundingMode.HALF_UP));
    }
}
```

### 3.2 资源管理

系统资源要遵循先申请再使用，不使用要释放，释放后不使用，不重复释放的原则。

**✅ 正确示例**:
```java
public class ResourceManager {

    // 使用try-with-resources确保资源释放
    public String readFile(String path) throws IOException {
        try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
            return reader.lines().collect(Collectors.joining("\n"));
        }
        // 自动关闭
    }

    // 使用CompletableFuture管理异步资源
    public CompletableFuture<Result> processAsync() {
        ExecutorService executor = Executors.newFixedThreadPool(10);
        try {
            return CompletableFuture.supplyAsync(() -> doProcess(), executor);
        } finally {
            executor.shutdown();
        }
    }

    // 谨慎使用finally确保资源释放
    public void processWithCleanup() {
        FileInputStream fis = null;
        try {
            fis = new FileInputStream("file.txt");
            // 处理
        } catch (IOException e) {
            log.error("文件处理失败", e);
        } finally {
            if (fis != null) {
                try {
                    fis.close();
                } catch (IOException e) {
                    log.warn("关闭文件失败", e);
                }
            }
        }
    }
}
```

### 3.3 核心流程依赖最小化

从事前预防角度看，核心流程的依赖要最小化，避免一个流程的异常牵连到核心流程。

**✅ 正确示例**:
```java
public class OrderService {

    public void createOrder(Order order) {
        // 核心流程：保存订单（必须成功）
        saveOrder(order);

        // 非核心流程：异步执行，不阻塞主流程
        CompletableFuture.runAsync(() -> {
            try {
                sendNotification(order);
                updateStatistics(order);
                syncToThirdParty(order);
            } catch (Exception e) {
                log.error("非核心操作失败", e);  // 只记录，不影响主流程
            }
        });
    }
}
```

### 3.4 流程异常降级处理

从事后容错的角度看，流程异常要有对应的降级兼容处理。

**✅ 正确示例**:
```java
public class PaymentService {

    public PaymentResult pay(Order order) {
        try {
            return doPay(order);
        } catch (PaymentException e) {
            log.error("支付失败，尝试降级处理", e);
            return tryFallbackPayment(order);
        }
    }

    private PaymentResult tryFallbackPayment(Order order) {
        // 降级：尝试使用备选支付渠道
        try {
            return fallbackGateway.pay(order);
        } catch (Exception e) {
            // 再次降级：允许稍后重试
            return PaymentResult.pending(order.getId(), "请稍后重试支付");
        }
    }
}
```

### 3.5 子系统崩溃处理

对于可靠性要求高的子系统，通过主备倒换、隔离等方式进行自愈。

**✅ 正确示例**:
```java
public class SubsystemManager {
    private volatile Subsystem primary;
    private Subsystem backup;
    private final AtomicBoolean isPrimaryFailed = new AtomicBoolean(false);

    public Result execute(Command command) {
        try {
            return primary.execute(command);
        } catch (SubsystemException e) {
            log.error("主子系统异常，触发故障检测", e);
            if (detectFailure()) {
                return switchToBackup(command);
            }
            throw e;
        }
    }

    private boolean detectFailure() {
        // 连续失败次数超过阈值才判定为故障
        return failureCount.incrementAndGet() >= FAILURE_THRESHOLD;
    }

    private Result switchToBackup(Command command) {
        synchronized (this) {
            if (isPrimaryFailed.compareAndSet(false, true)) {
                log.warn("切换到备份子系统");
                primary = backup;
                backup = createNewBackup();
            }
        }
        return primary.execute(command);
    }
}
```

## 完整检查清单

- [ ] 关键操作是否有防呆设计？
- [ ] 是否有流控和负载均衡机制？
- [ ] 通信失败是否有重试机制？
- [ ] 状态是否需要同步？
- [ ] 资源是否正确管理（获取→使用→释放）？
- [ ] 核心流程依赖是否最小化？
- [ ] 是否有降级处理策略？
- [ ] 子系统故障是否可检测和恢复？