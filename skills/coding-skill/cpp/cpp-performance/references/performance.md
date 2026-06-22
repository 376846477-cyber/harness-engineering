# C++高效性规范详细参考

## 1. 移动语义优化

### 1.1 移动构造与移动赋值

**规则1.1.1：为资源管理类实现移动操作**

✓ 正确示例：
```cpp
class Buffer {
    char* data_;
    size_t size_;
public:
    Buffer(size_t size) : data_(new char[size]), size_(size) {}
    ~Buffer() { delete[] data_; }
    
    Buffer(const Buffer& other) : data_(new char[other.size_]), size_(other.size_) {
        std::memcpy(data_, other.data_, size_);
    }
    
    Buffer(Buffer&& other) noexcept : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }
    
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }
};
```

**规则1.1.2：使用std::move显式触发移动**

✓ 正确示例：
```cpp
std::vector<std::string> processNames(std::vector<std::string> names) {
    std::vector<std::string> result;
    result.reserve(names.size());
    for (auto& name : names) {
        if (isValid(name)) {
            result.push_back(std::move(name));  // 移动而非拷贝
        }
    }
    return result;
}
```

### 1.2 noexcept移动操作

**规则1.2.1：移动操作应标记为noexcept**

✓ 正确示例：
```cpp
class StringPool {
    std::vector<std::string> pool_;
public:
    StringPool(StringPool&& other) noexcept : pool_(std::move(other.pool_)) {}
    StringPool& operator=(StringPool&& other) noexcept {
        pool_ = std::move(other.pool_);
        return *this;
    }
};
```

### 1.3 完美转发

**规则1.3.1：使用std::forward实现完美转发**

✓ 正确示例：
```cpp
template<typename T, typename... Args>
std::unique_ptr<T> makeUnique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```

## 2. 零拷贝技术

### 2.1 使用std::string_view

**规则2.1.1：函数参数使用string_view替代const string&**

✓ 正确示例：
```cpp
void parseConfig(std::string_view content) {
    size_t pos = content.find('=');
    std::string_view key = content.substr(0, pos);
    std::string_view value = content.substr(pos + 1);
}

parseConfig(config);
parseConfig("name=test");  // 无临时std::string构造
```

### 2.2 使用std::span (C++20)

**规则2.2.1：使用span传递连续容器**

✓ 正确示例：
```cpp
void processSpan(std::span<int> data) {
    for (int& val : data) { val *= 2; }
}

std::vector<int> vec = {1, 2, 3};
int arr[] = {4, 5, 6};
processSpan(vec);
processSpan(arr);
```

### 2.3 避免不必要的拷贝

**规则2.3.1：使用const引用传递大对象**

✓ 正确示例：
```cpp
struct LargeData { std::array<double, 10000> values; };

double computeSum(const LargeData& data) {
    return std::accumulate(data.values.begin(), data.values.end(), 0.0);
}
```

## 3. 智能指针性能

### 3.1 unique_ptr vs shared_ptr

**规则3.1.1：优先使用unique_ptr**

✓ 正确示例：
```cpp
auto ptr = std::make_unique<Widget>();  // 零开销抽象
```

✗ 错误示例：
```cpp
std::shared_ptr<Resource> resource_;  // 原子引用计数开销
```

### 3.2 shared_ptr开销分析

**规则3.2.1：了解shared_ptr的性能影响**

✓ 正确示例：
```cpp
void useShared(const std::shared_ptr<Data>& data) {
    process(data);  // 引用传递，避免原子增减
}
```

✗ 错误示例：
```cpp
void processShared(std::shared_ptr<Data> data) {  // 每次调用原子增减
}
```

### 3.3 make_unique与make_shared

**规则3.3.1：使用make函数创建智能指针**

✓ 正确示例：
```cpp
auto ptr = std::make_unique<Widget>(arg1, arg2);  // 单次分配
auto ptr = std::make_shared<Widget>(arg1, arg2);  // 单次分配
```

✗ 错误示例：
```cpp
std::shared_ptr<Widget> ptr(new Widget());  // 两次分配
processWidget(std::unique_ptr<Widget>(new Widget()), computeValue());
// 可能内存泄漏：new Widget后，computeValue()抛异常
```

## 4. 数据结构选择

### 4.1 顺序容器比较

**规则4.1.1：根据访问模式选择容器**

| 容器 | 随机访问 | 头部插入 | 中间插入 | 缓存友好 |
|------|----------|----------|----------|----------|
| vector | O(1) | O(n) | O(n) | ✓ |
| deque | O(1) | O(1) | O(n) | ✓ |
| list | O(n) | O(1) | O(1) | ✗ |

### 4.2 关联容器选择

**规则4.2.1：根据需求选择map或unordered_map**

| 容器 | 查找 | 插入 | 有序遍历 |
|------|------|------|----------|
| map | O(log n) | O(log n) | ✓ |
| unordered_map | O(1)平均 | O(1)平均 | ✗ |

✓ 正确示例：
```cpp
std::unordered_map<std::string, int> lookup;
lookup.reserve(10000);
lookup["key"] = 1;  // O(1)平均

std::map<int, std::string> ordered;
auto begin = ordered.lower_bound(100);  // 范围查询
```

### 4.3 vector优化技巧

**规则4.3.1：预分配vector容量**

✓ 正确示例：
```cpp
std::vector<int> result;
result.reserve(10000);  // 预分配
for (int i = 0; i < 10000; i++) {
    result.push_back(i);  // 无重分配
}
```

## 5. 缓存友好代码

### 5.1 数据布局优化

**规则5.1.1：使用结构数组替代数组结构**

✓ 正确示例：
```cpp
struct ParticlesSOA {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    
    void updatePositions(float dt) {
        for (size_t i = 0; i < x.size(); i++) {
            x[i] += vx[i] * dt;  // 连续访问，缓存友好
        }
    }
};
```

✗ 错误示例：
```cpp
struct Particle { float x, y, z, vx, vy, vz; };
std::vector<Particle> particles;  // 加载整个Particle，缓存不友好
```

### 5.2 内存对齐

**规则5.2.1：对齐数据结构提升访问性能**

✓ 正确示例：
```cpp
struct alignas(64) CacheLineAligned { int data[15]; };  // 缓存行对齐

struct alignas(16) SIMDReady { float values[4]; };  // SIMD友好

void processSIMD(const SIMDReady* data, size_t n) {
    for (size_t i = 0; i < n; i++) {
        __m128 vals = _mm_load_ps(data[i].values);  // 对齐加载
        vals = _mm_mul_ps(vals, _mm_set1_ps(2.0f));
        _mm_store_ps(data[i].values, vals);
    }
}

struct alignas(64) ThreadLocalCounter {  // 避免伪共享
    std::atomic<int> count{0};
    char padding[60];
};
```

### 5.3 访问模式优化

**规则5.3.1：优化数据访问顺序**

✓ 正确示例：
```cpp
void matrixMultiply(const float* A, const float* B, float* C, size_t N) {
    for (size_t i = 0; i < N; i++) {
        for (size_t k = 0; k < N; k++) {
            float temp = A[i * N + k];
            for (size_t j = 0; j < N; j++) {
                C[i * N + j] += temp * B[k * N + j];  // 缓存友好
            }
        }
    }
}
```

## 6. 编译器优化

### 6.1 优化选项

**规则6.1.1：选择合适的优化级别**

```bash
# 开发调试
g++ -O0 -g -fsanitize=address main.cpp

# 常规发布
g++ -O2 -DNDEBUG main.cpp

# 最高性能
g++ -O3 -march=native -DNDEBUG main.cpp

# 链接时优化
g++ -O3 -flto -march=native main.cpp

# PGO优化
g++ -O3 -fprofile-generate main.cpp && ./a.out
g++ -O3 -fprofile-use main.cpp
```

### 6.2 constexpr编译期计算

**规则6.2.1：使用constexpr进行编译期计算**

✓ 正确示例：
```cpp
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

constexpr int FACTORIAL_10 = factorial(10);  // 编译期计算

constexpr std::array<int, 10> makeLookupTable() {
    std::array<int, 10> table{};
    for (int i = 0; i < 10; i++) { table[i] = i * i; }
    return table;
}

constexpr auto SQUARE_TABLE = makeLookupTable();  // 编译期生成
```

## 7. 性能分析工具

**规则7.1.1：使用性能分析工具定位瓶颈**

```bash
# perf性能分析
perf record -g ./program && perf report

# Valgrind工具集
valgrind --tool=callgrind ./program && kcachegrind callgrind.out.*
valgrind --tool=cachegrind ./program  # 缓存分析

# gperftools
LD_PRELOAD=/usr/lib/libtcmalloc.so ./program
LD_PRELOAD=/usr/lib/libprofiler.so CPUPROFILE=out.prof ./program

# Sanitizer
g++ -fsanitize=address -g program.cpp
g++ -fsanitize=thread -g program.cpp
```

---

## 性能优化检查清单

### 移动语义
- [ ] 为资源管理类实现移动构造/赋值
- [ ] 移动操作标记noexcept
- [ ] 使用std::move显式转移所有权
- [ ] 实现完美转发(std::forward)

### 零拷贝
- [ ] 函数参数使用string_view
- [ ] 连续容器使用span
- [ ] 大对象传递使用const引用
- [ ] 避免不必要的临时对象

### 智能指针
- [ ] 优先使用unique_ptr
- [ ] 使用make_unique/make_shared
- [ ] 理解shared_ptr的原子开销
- [ ] 避免循环引用

### 数据结构
- [ ] 随机访问使用vector
- [ ] 频繁头部插入使用deque
- [ ] 无序查找使用unordered_map
- [ ] 需要有序遍历使用map
- [ ] vector使用reserve预分配

### 缓存优化
- [ ] 考虑SOA布局
- [ ] 数据结构内存对齐
- [ ] 行优先访问矩阵
- [ ] 避免伪共享

### 编译优化
- [ ] 选择合适的优化级别(-O2/-O3)
- [ ] 使用-march=native
- [ ] 启用LTO链接时优化
- [ ] 使用constexpr编译期计算

### 性能分析
- [ ] 使用perf定位热点
- [ ] 使用Valgrind检测问题
- [ ] 编写基准测试
- [ ] 测量优化前后对比
