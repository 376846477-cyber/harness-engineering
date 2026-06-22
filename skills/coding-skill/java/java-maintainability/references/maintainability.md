# 华为Java可维护性规范 - 详细版

基于华为Clean Code指导书第2章"可维护"特征

## 1. 通过抽象隔离变化

### 1.1 类型抽象（泛型）

将特定的数据类型抽象成泛型，通过抽象的泛型将不变的算法与可变的类型隔离。

**✅ 正确示例**:
```java
// 泛型抽象 - 算法复用
public interface Repository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    T save(T entity);
    void delete(ID id);
}

public class UserRepository implements Repository<User, Long> {
    @Override
    public User findById(Long id) { ... }
    // ...
}

public class OrderRepository implements Repository<Order, String> {
    @Override
    public Order findById(String id) { ... }
    // ...
}
```

### 1.2 实现抽象出接口（封装）

将实现封装，并抽象出接口，通过接口隔离不同的变化。

**✅ 正确示例**:
```java
// 定义稳定的接口
public interface PaymentGateway {
    PaymentResult pay(PaymentRequest request);
   RefundResult refund(RefundRequest request);
}

// 多种实现
public class AlipayGateway implements PaymentGateway { ... }
public class WechatPayGateway implements PaymentGateway { ... }
public class CreditCardGateway implements PaymentGateway { ... }

// 依赖抽象
public class OrderService {
    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

**❌ 错误示例**:
```java
// 直接依赖具体实现
public class OrderService {
    private final AlipayGateway alipayGateway;  // 紧耦合

    public void pay(Order order) {
        alipayGateway.pay(order);  // 难以切换支付方式
    }
}
```

### 1.3 抽象构建框架，实现延迟到子类（多态）

识别流程中不可变和可变部分，将可变部分抽象成接口，通过多态实现扩展。

**✅ 正确示例**:
```java
public abstract class DataProcessor {
    // 框架稳定
    public final void process() {
        beforeProcess();
        doProcess();
        afterProcess();
    }

    protected void beforeProcess() { ... }
    protected abstract void doProcess();
    protected void afterProcess() { ... }
}

public class XmlDataProcessor extends DataProcessor {
    @Override
    protected void doProcess() { ... }
}

public class JsonDataProcessor extends DataProcessor {
    @Override
    protected void doProcess() { ... }
}
```

## 2. 缩小依赖范围

### 2.1 接口最小化/参数最少化

接口变化时使用的越多影响范围越广，遵循接口隔离原则，不要让客户依赖不需要的东西。

**✅ 正确示例**:
```java
// 单一职责的接口
public interface Reader {
    void open();
    String read();
    void close();
}

public interface Writer {
    void open();
    void write(String data);
    void close();
}

// 组合使用
public class FileHandler {
    private final Reader reader;
    private final Writer writer;
}
```

**❌ 错误示例**:
```java
// 臃肿的接口
public interface FileManager {
    void openFile(String path);
    void closeFile();
    String readFile();
    void writeFile(String content);
    void deleteFile();
    void copyFile(String from, String to);
    // 接口方法太多，承担了太多职责
}
```

## 3. 向着稳定的方向依赖

### 3.1 依赖用户角度定义的接口

站在需求的角度定义接口，而不是实现方式的角度，这样会让接口更加稳定。

**✅ 正确示例**:
```java
// 站在业务角度定义接口
public interface OrderQueryService {
    Order getOrderById(String orderId);
    List<Order> getOrdersByUserId(String userId);
    List<Order> getOrdersByStatus(OrderStatus status);
}

// 实现侧适配业务接口
public class OrderRepository implements OrderQueryService {
    // 适配底层数据库接口
}
```

### 3.2 子类不破坏父类接口契约

子类不应该破坏其父类与客户之间的契约，即里氏替换原则。

**✅ 正确示例**:
```java
public class Animal {
    public void speak() {
        System.out.println("animal makes sound");
    }
}

public class Dog extends Animal {
    @Override
    public void speak() {
        System.out.println("dog barks");  // 符合契约
    }
}

// 可以用子类替代父类
Animal animal = new Dog();
animal.speak();  // 正常工作
```

**❌ 错误示例**:
```java
public class CustomList extends ArrayList {
    @Override
    public int size() {
        return -1;  // 破坏父类契约！
    }
}
```

## 4. 引入中间层降低耦合

### 4.1 适配器模式

接口本身变化，为保证前向兼容，可以增加适配层解决。

**✅ 正确示例**:
```java
// 业务定义的统一接口
public interface Cache {
    void set(String key, Object value);
    Object get(String key);
    void delete(String key);
}

// 适配不同的缓存实现
public class RedisCacheAdapter implements Cache {
    private final RedisClient redisClient;

    @Override
    public void set(String key, Object value) {
        redisClient.set(key, serialize(value));
    }

    @Override
    public Object get(String key) {
        return deserialize(redisClient.get(key));
    }
    // ...
}
```

### 4.2 生产消费者模式

接口调用时机不确定时，使用消息队列解耦。

**✅ 正确示例**:
```java
public class AsyncTaskQueue {
    private final BlockingQueue<Task> queue = new LinkedBlockingQueue<>();

    public void submit(Task task) {
        queue.offer(task);
    }

    public Task take() throws InterruptedException {
        return queue.take();
    }
}

// 生产者和消费者通过队列解耦
public class TaskProducer {
    private final AsyncTaskQueue queue;

    public void produce(Task task) {
        queue.submit(task);  // 不直接调用消费者
    }
}
```

### 4.3 观察者模式

事件生产方不关心事件接收方，通过注册机制实现依赖倒置。

**✅ 正确示例**:
```java
public interface Observer {
    void update(Event event);
}

public class Subject {
    private final List<Observer> observers = new ArrayList<>();

    public void attach(Observer observer) {
        observers.add(observer);
    }

    public void detach(Observer observer) {
        observers.remove(observer);
    }

    protected void notifyObservers(Event event) {
        for (Observer observer : observers) {
            observer.update(event);  // 不关心谁在监听
        }
    }
}
```

## 5. 抽取公共部分

### 5.1 继承 vs 组合

只有is-a关系的类才可以使用继承，优先使用对象组合。

**✅ 正确示例**:
```java
// 组合优于继承
public class Car {
    private final Engine engine;
    private final Wheels wheels;

    public Car(Engine engine, Wheels wheels) {
        this.engine = engine;
        this.wheels = wheels;
    }
}

// 继承用于真正的is-a关系
public class Dog extends Animal {
    // Dog确实是Animal的一种
}
```

**❌ 错误示例**:
```java
// 滥用继承
public class Stack extends ArrayList {  // Stack不是ArrayList
    public void push(Object e) { add(e); }
    public Object pop() { return remove(size() - 1); }
}
```

### 5.2 提取公共代码

重复代码是万恶之源，需要提取到公共方法或基类中。

**✅ 正确示例**:
```java
// 提取公共方法
public abstract class BaseService {
    protected void validate(Object request) {
        if (request == null) {
            throw new IllegalArgumentException("请求不能为空");
        }
    }

    protected void log(String message) {
        System.out.println("[INFO] " + message);
    }
}

public class UserService extends BaseService {
    public void createUser(UserRequest request) {
        validate(request);
        // 业务逻辑
        log("用户创建成功");
    }
}

public class OrderService extends BaseService {
    public void createOrder(OrderRequest request) {
        validate(request);
        // 业务逻辑
        log("订单创建成功");
    }
}
```

## 完整检查清单

- [ ] 是否通过接口抽象隔离了变化？
- [ ] 接口是否遵循单一职责（方法数量合理）？
- [ ] 是否依赖抽象而非具体实现？
- [ ] 子类是否能替换父类而不破坏契约？
- [ ] 是否有重复代码需要提取？
- [ ] 继承关系是否符合is-a关系（优先用组合）？
- [ ] 依赖方向是否稳定（依赖抽象而非实现）？