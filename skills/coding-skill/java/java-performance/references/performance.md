# 华为Java高效性规范 - 详细版

基于华为Clean Code指导书第2章"高效"特征

## 1. 提升CPU资源利用率

### 1.1 选择高效的算法

大数据排序用快速排序/归并排序等，大数据查找用hash、二分、红黑树等。

**✅ 正确示例**:
```java
// 使用HashSet进行O(1)查找
Set<String> userIds = new HashSet<>(userList.stream()
    .map(User::getId)
    .collect(Collectors.toSet()));

// 判断是否存在
boolean exists = userIds.contains(targetId);

// 使用Map进行快速查找
Map<String, User> userMap = users.stream()
    .collect(Collectors.toMap(User::getId, u -> u));

User user = userMap.get(userId);  // O(1)
```

**❌ 错误示例**:
```java
// 列表遍历查找 - O(n)
for (User user : users) {
    if (user.getId().equals(targetId)) {
        return user;
    }
}
```

### 1.2 空间换时间

合理使用缓存减少重复计算。

**✅ 正确示例**:
```java
public class FibonacciCalculator {
    private final Map<Integer, BigInteger> cache = new HashMap<>();

    public BigInteger fibonacci(int n) {
        if (n <= 1) {
            return BigInteger.valueOf(n);
        }
        return cache.computeIfAbsent(n, k ->
            fibonacci(k - 1).add(fibonacci(k - 2))
        );
    }
}
```

### 1.3 减少CPU外设访问开销

频繁访问的数据放到缓存。

**✅ 正确示例**:
```java
public class Configuration {
    // 内存缓存配置，避免每次读取
    private static final Map<String, String> configCache = new ConcurrentHashMap<>();

    public static String getConfig(String key) {
        return configCache.computeIfAbsent(key, k -> {
            // 从文件/数据库读取一次
            return loadFromFile(key);
        });
    }
}
```

### 1.4 减少冗余或无效计算

按需初始化，避免重复计算。

**✅ 正确示例**:
```java
public class ReportGenerator {
    // 避免重复计算，使用延迟加载
    private List<ReportData> cachedData;

    public List<ReportData> getData() {
        if (cachedData == null) {
            cachedData = loadDataExpensive();
        }
        return cachedData;
    }

    // 避免在循环中重复调用方法
    public void processItems(List<Item> items) {
        int size = items.size();  // 预先计算
        for (int i = 0; i < size; i++) {  // 使用size而不是每次调用size()
            process(items.get(i));
        }
    }
}
```

**❌ 错误示例**:
```java
// 循环中重复计算
for (int i = 0; i < items.size(); i++) {  // 每次迭代都调用size()
    process(items.get(i));
}
```

### 1.5 合理使用并行编程

对于CPU密集型任务，合理使用多线程。

**✅ 正确示例**:
```java
public class ParallelProcessor {
    // 使用流式API并行处理
    public List<Result> process(List<Task> tasks) {
        return tasks.parallelStream()
            .map(this::processTask)
            .collect(Collectors.toList());
    }

    // 使用线程池
    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    public CompletableFuture<Result> submit(Task task) {
        return CompletableFuture.supplyAsync(() -> processTask(task), executor);
    }
}
```

## 2. 提升内存资源利用率

### 2.1 减少内存浪费

合理选择数据结构，避免空间浪费。

**✅ 正确示例**:
```java
// 根据实际需求选择集合类型
// 如果知道大小，使用ArrayList并指定初始容量
List<User> users = new ArrayList<>(1000);

// 如果只需要唯一元素，使用HashSet而不是List
Set<String> uniqueNames = new HashSet<>(nameList);

// 使用基本类型避免装箱拆箱
int[] ids = new int[1000];  // vs Integer[]

// 使用合适的对象大小
// 大量字符串使用StringBuilder
StringBuilder sb = new StringBuilder();
for (String s : strings) {
    sb.append(s).append(",");
}
```

**❌ 错误示例**:
```java
// 默认容量太小导致频繁扩容
List<User> users = new ArrayList<>();  // 默认为10

// 使用包装类型存储大量数据
Integer[] ids = new Integer[1000];  // 每个Integer有额外开销
```

### 2.2 合理的内存生命周期管理

及时释放不需要的内存。

**✅ 正确示例**:
```java
public class DataProcessor {
    public void process() {
        // 使用try-with-resources自动释放
        try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
            // 处理
        }  // 自动关闭

        // 及时清理引用
        List<TempData> tempList = new ArrayList<>();
        try {
            // 使用tempList
        } finally {
            tempList.clear();  // 帮助GC
        }
    }

    // 使用弱引用缓存大对象
    private final WeakHashMap<Key, Value> cache = new WeakHashMap<>();
}
```

### 2.3 减少内存碎片

对象池化和复用减少内存分配。

**✅ 正确示例**:
```java
public class ObjectPool<T> {
    private final Queue<T> available;
    private final Supplier<T> factory;

    public ObjectPool(Supplier<T> factory, int size) {
        this.factory = factory;
        this.available = new ArrayDeque<>(size);
        for (int i = 0; i < size; i++) {
            available.add(factory.get());
        }
    }

    public T acquire() {
        return available.poll();
    }

    public void release(T obj) {
        available.offer(obj);
    }
}

// 复用StringBuilder
public class StringProcessor {
    private final StringBuilder builder = new StringBuilder(1024);

    public String process(String input) {
        builder.setLength(0);  // 重置而不是创建新的
        builder.append("Processed: ");
        builder.append(input);
        return builder.toString();
    }
}
```

## 3. IO优化

### 3.1 减少IO次数

批量操作减少IO次数。

**✅ 正确示例**:
```java
public class BatchProcessor {
    // 批量插入而非逐条插入
    public void batchInsert(List<Entity> entities) {
        jdbcTemplate.batchUpdate(
            "INSERT INTO users (name) VALUES (?)",
            entities,
            entities.size(),
            (ps, entity) -> ps.setString(1, entity.getName())
        );
    }

    // 缓冲IO
    try (BufferedInputStream bis = new BufferedInputStream(
            new FileInputStream("largefile.bin"))) {
        // 使用缓冲减少系统调用
    }
}
```

### 3.2 异步IO

非阻塞IO提高吞吐量。

**✅ 正确示例**:
```java
public class AsyncFileWriter {
    private final AsynchronousFileChannel channel;

    public CompletableFuture<Void> write(String data, long position) {
        ByteBuffer buffer = StandardCharsets.UTF_8.encode(data);
        return CompletableFuture.runAsync(() -> {
            try {
                channel.write(buffer, position).get();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });
    }
}
```

## 4. 数据库优化

### 4.1 减少查询次数

使用join和批量查询。

**✅ 正确示例**:
```java
public class UserDao {
    // 使用JOIN一次查询获取关联数据
    @Query("SELECT u FROM User u LEFT JOIN FETCH u.roles WHERE u.id = :id")
    User findByIdWithRoles(@Param("id") Long id);

    // 使用IN批量查询
    @Query("SELECT u FROM User u WHERE u.id IN :ids")
    List<User> findByIds(@Param("ids") List<Long> ids);
}
```

### 4.2 正确使用索引

避免全表扫描。

**✅ 正确示例**:
```java
// 使用索引列进行查询
@Query("SELECT u FROM User u WHERE u.email = :email")
User findByEmail(@Param("email") String email);

// 避免在索引列上使用函数
@Query("SELECT u FROM User u WHERE u.status = 'ACTIVE'")  // 正确
// @Query("SELECT u FROM User u WHERE UPPER(u.name) = :name")  // 错误，无法使用索引
```

## 完整检查清单

- [ ] 是否使用了合适的数据结构（HashSet vs List）？
- [ ] 是否有重复计算可以缓存？
- [ ] 循环中是否有重复调用方法？
- [ ] 是否可以使用并行处理？
- [ ] 集合是否指定了初始容量？
- [ ] 是否使用了基本类型避免装箱？
- [ ] 资源是否正确释放（try-with-resources）？
- [ ] 数据库操作是否使用批量处理？