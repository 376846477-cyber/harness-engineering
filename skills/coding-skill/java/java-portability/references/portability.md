# 华为Java可移植性规范 - 详细版

基于华为Clean Code指导书第2章"可移植"特征

## 1. 避免平台相关的API

### 1.1 文件路径分隔符

不同操作系统使用不同的路径分隔符，使用Java提供的API处理路径。

**✅ 正确示例**:
```java
import java.io.File;

// 使用File.separator获取平台分隔符
String path = "data" + File.separator + "files" + File.separator + "config.txt";
File file = new File(path);

// 使用URI构建路径
Path configPath = Paths.get("data", "files", "config.txt");

// 使用Paths.get接受可变参数
Path path1 = Paths.get("data", "files", "config.txt");
Path path2 = Paths.get("/home", "user", "config.txt");
```

**❌ 错误示例**:
```java
// 硬编码路径分隔符
String path = "data/files/config.txt";  // Linux可以，Windows不行

// 或者
String path = "data\\files\\config.txt";  // Windows可以，Linux不行
```

### 1.2 行分隔符

不同系统使用不同的行分隔符。

**✅ 正确示例**:
```java
// 使用系统行分隔符
String line = "line1" + System.lineSeparator() + "line2";

// 或者使用Java 11+的String.lines()
String content = "line1\nline2\nline3";
content.lines().forEach(System.out::println);
```

**❌ 错误示例**:
```java
// 硬编码\n或\r\n
String content = "line1\r\nline2\r\n";  // 在Linux下可能有问题

// 写入文件时
writer.write("line1\r\n");
```

### 1.3 临时目录

不同系统使用不同的临时目录。

**✅ 正确示例**:
```java
// 使用系统临时目录
String tempDir = System.getProperty("java.io.tmpdir");
File tempFile = File.createTempFile("prefix", ".tmp");

// 使用Path API
Path tempPath = Files.createTempFile("prefix", ".tmp");
```

**❌ 错误示例**:
```java
// 硬编码临时目录
File tempFile = new File("/tmp/myapp/data.tmp");  // Windows没有/tmp
```

### 1.4 编码/字符集

不同系统默认编码可能不同。

**✅ 正确示例**:
```java
// 明确指定字符集
String content = new String(bytes, StandardCharsets.UTF_8);
BufferedReader reader = new BufferedReader(
    new InputStreamReader(new FileInputStream(file), StandardCharsets.UTF_8));

// 文件写入指定编码
Files.write(path, content.getBytes(StandardCharsets.UTF_8));
```

**❌ 错误示例**:
```java
// 使用默认编码
String content = new String(bytes);  // 依赖系统默认编码

// 或者
BufferedReader reader = new BufferedReader(new FileReader(file));
```

## 2. 避免硬件相关的代码

### 2.1 位运算和整数范围

不同平台的整数类型范围可能不同。

**✅ 正确示例**:
```java
// 使用Java标准类型，避免假设
int value = 100;
long bigValue = 1000L;

// 需要无符号时使用Java 8+的无符号方法
int unsigned = Integer.toUnsignedLong(-1);  // 4294967295L

// 避免假设int是32位，使用Java提供的位操作方法
int rotated = Integer.rotateLeft(value, 4);
int bitCount = Integer.bitCount(value);
```

**❌ 错误示例**:
```java
// 假设特定位宽
int mask = 0xFF;  // 假设8位

// 移位超出范围
int value = 1 << 32;  // 未定义行为
```

### 2.2 浮点数精度

浮点数精度问题在所有平台都存在。

**✅ 正确示例**:
```java
// 使用BigDecimal进行精确计算
BigDecimal price = new BigDecimal("19.99");
BigDecimal quantity = new BigDecimal("3");
BigDecimal total = price.multiply(quantity);  // 精确结果

// 比较使用compareTo而非equals
if (bigDecimal1.compareTo(bigDecimal2) == 0) { ... }
```

**❌ 错误示例**:
```java
// 使用double进行货币计算
double price = 19.99;
double total = price * 3;  // 结果可能是59.96999999999
```

### 2.3 大小端字节序

不同CPU架构使用不同的字节序。

**✅ 正确示例**:
```java
// 使用Java标准类处理字节序
ByteBuffer buffer = ByteBuffer.wrap(bytes);
buffer.order(ByteOrder.BIG_ENDIAN);  // 或LITTLE_ENDIAN

// 使用DataInputStream/DataOutputStream（使用BE）
DataOutputStream dos = new DataOutputStream(outputStream);
dos.writeInt(12345);
```

## 3. 避免依赖特定JVM版本

### 3.1 使用兼容的API

尽量使用广泛支持的API，必要时做版本检查。

**✅ 正确示例**:
```java
// 版本检查
private static final boolean IS_JAVA_8_OR_ABOVE =
    Runtime.version().feature() >= 8;

// 使用反射调用新API
try {
    Method method = clazz.getMethod("newMethod");
    method.invoke(obj);
} catch (NoSuchMethodException e) {
    // 回退到旧实现
}
```

**❌ 错误示例**:
```java
// 直接使用高版本API导致低版本JVM失败
List<String> list = List.of("a", "b", "c");  // Java 9+

// 应该在低版本兼容
List<String> list = Arrays.asList("a", "b", "c");
```

### 3.2 模块系统

Java 9+的模块系统需要明确声明依赖。

**✅ 正确示例**:
```java
// module-info.java
module com.example.myapp {
    requires java.sql;
    requires com.google.gson;

    exports com.example.myapp.api;
}
```

## 4. 依赖库的可移植性

### 4.1 选择跨平台依赖

选择支持多平台的库。

**✅ 正确示例**:
```java
// 使用标准Java API或跨平台库
import java.time.*;  // Java 8+标准时间API
import java.nio.file.*;  // Java 7+文件API

// 跨平台的日志库
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

// JSON处理
import com.fasterxml.jackson.databind.ObjectMapper;
```

**❌ 错误示例**:
```java
// 使用平台特定的原生库
// Windows: using COM
// Linux: using native shared objects

// 如果必须使用，添加平台检测
String os = System.getProperty("os.name");
if (os.contains("Windows")) {
    // Windows特定代码
} else if (os.contains("Linux")) {
    // Linux特定代码
}
```

### 4.2 本地库加载

加载本地库时考虑多平台。

**✅ 正确示例**:
```java
public class NativeLibraryLoader {
    private static final String OS = System.getProperty("os.name").toLowerCase();
    private static final String ARCH = System.getProperty("os.arch").toLowerCase();

    public static void loadLibrary(String libName) {
        String mappedLibName = libName;

        if (OS.contains("windows")) {
            mappedLibName = libName + ".dll";
        } else if (OS.contains("linux")) {
            mappedLibName = "lib" + libName + ".so";
        } else if (OS.contains("mac")) {
            mappedLibName = "lib" + libName + ".dylib";
        }

        // 从classpath加载
        System.load(NativeLibraryLoader.class.getResource("/native/" + mappedLibName).getFile());
    }
}
```

## 5. 时区和locale处理

### 5.1 时区

不同时区可能导致时间计算错误。

**✅ 正确示例**:
```java
// 使用ZoneId明确指定时区
ZonedDateTime now = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));

// 存储和传输使用UTC
Instant timestamp = Instant.now();
String utcString = timestamp.toString();  // "2024-01-15T10:30:00Z"

// 解析时指定时区
ZonedDateTime zdt = ZonedDateTime.parse("2024-01-15T10:30:00+08:00", DateTimeFormatter.ISO_DATE_TIME);
```

### 5.2 Locale

不同locale的格式化和解析可能不同。

**✅ 正确示例**:
```java
// 明确指定locale
NumberFormat nf = NumberFormat.getInstance(Locale.US);
String formatted = nf.format(1234.56);

// 使用标准locale
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd")
    .withLocale(Locale.CHINA);
```

## 完整检查清单

- [ ] 文件路径是否使用了File.separator或Paths？
- [ ] 行分隔符是否使用了System.lineSeparator()？
- [ ] 临时目录是否使用了java.io.tmpdir？
- [ ] 字符集是否明确指定（UTF-8）？
- [ ] 浮点数计算是否使用BigDecimal？
- [ ] API是否与JVM版本兼容？
- [ ] 依赖库是否跨平台？
- [ ] 时间处理是否使用UTC或明确时区？