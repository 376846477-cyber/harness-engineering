---
name: java-concurrency
description: Java编码规范 - 多线程并发规范。适用于所有Java代码开发场景：(1) 编写多线程代码；(2) 线程同步和锁的使用；(3) 共享变量访问控制；(4) 死锁预防。触发关键词：Java、线程、thread、并发、concurrent、synchronized、锁、lock、volatile、ThreadLocal
---

# Java多线程并发规范

基于Java语言通用编程规范V8"编程实践-多线程并发"

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则8.1 | 共享变量必须同步访问 |
| 规则8.2 | 使用安全的多线程集合 |
| 规则8.3 | 避免死锁：固定加锁顺序 |
| 规则8.4 | volatile用于单纯状态标志 |
| 规则8.5 | ThreadLocal使用后清理 |

## 线程安全示例

**✅ 正确示例**:
```java
// 使用synchronized
public synchronized void increment() {
    count++;
}

// 使用ConcurrentHashMap
private final ConcurrentMap<String, Object> cache = new ConcurrentHashMap<>();

// 使用volatile
private volatile boolean running = true;

// 使用ThreadLocal
private static final ThreadLocal<DateFormat> df = ThreadLocal.withInitial(() ->
    new SimpleDateFormat("yyyy-MM-dd"));
```

**❌ 错误示例**:
```java
// 错误的双重检查锁定
if (instance == null) {
    synchronized (this) {
        if (instance == null) {
            instance = new Singleton();
        }
    }
}
```

## 死锁预防

```java
// 固定加锁顺序防止死锁
public void transfer(Account from, Account to, int amount) {
    if (from.getId() < to.getId()) {
        synchronized (from) {
            synchronized (to) {
                // 转账逻辑
            }
        }
    } else {
        synchronized (to) {
            synchronized (from) {
                // 转账逻辑
            }
        }
    }
}
```

## 详细并发规范

见 [references/concurrency.md](references/concurrency.md)