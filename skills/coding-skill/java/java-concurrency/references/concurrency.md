# 华为Java多线程并发规范详细参考

## 规则8.1 共享变量同步

- 多个线程访问的共享变量必须同步
- 使用synchronized或java.util.concurrent包

```java
// 错误：多线程不安全
private int count = 0;
public void increment() { count++; }

// 正确：使用synchronized
private int count = 0;
public synchronized void increment() { count++; }

// 正确：使用AtomicInteger
private AtomicInteger count = new AtomicInteger(0);
public void increment() { count.incrementAndGet(); }
```

## 规则8.2 使用线程安全集合

| 不安全 | 安全替代 |
|-------|---------|
| ArrayList | CopyOnWriteArrayList |
| HashMap | ConcurrentHashMap |
| HashSet | CopyOnWriteArraySet |
| LinkedList | LinkedBlockingQueue |

## 规则8.3 死锁预防

### 固定加锁顺序
```java
// 按对象ID固定顺序
public void transfer(Account a, Account b, int amount) {
    if (a.getId() < b.getId()) {
        synchronized (a) {
            synchronized (b) {
                // 转账
            }
        }
    } else {
        synchronized (b) {
            synchronized (a) {
                // 转账
            }
        }
    }
}
```

### 避免嵌套锁
```java
// 不好：容易死锁
public synchronized void method1() {
    method2();
}
public synchronized void method2() {
    // 持有锁时调用其他synchronized方法
}
```

## 规则8.4 volatile使用

- 用于单纯的状态标志
- 适用于boolean标记

```java
// 正确：状态标志
private volatile boolean running = true;

public void stop() {
    running = false;
}

public void run() {
    while (running) {
        // 循环
    }
}
```

## 规则8.5 ThreadLocal清理

- 使用完ThreadLocal后调用remove()
- 避免内存泄漏

```java
private static final ThreadLocal<DateFormat> df =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

// 使用后清理
try {
    String date = df.get().format(new Date());
} finally {
    df.remove();
}
```

## 规则8.6 避免过早发布对象引用

- 发布对象前确保线程安全
- 避免this引用在构造函数中逸出

```java
// ❌ 错误：this引用逸出
public class ThisEscape {
    public ThisEscape(EventSource source) {
        source.registerListener(this);  // 逸出
    }
}

// ✅ 正确：使用工厂方法
public class SafeListener {
    private final EventListener listener;

    private SafeListener() {
        this.listener = new EventListener() {
            // ...
        };
    }

    public static SafeListener newInstance(EventSource source) {
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
```

## 线程安全检查清单

1. ✅ 共享变量有同步保护
2. ✅ 使用线程安全集合
3. ✅ 避免死锁（固定顺序）
4. ✅ volatile用于状态标志
5. ✅ ThreadLocal使用后清理
6. ✅ 避免对象引用逸出