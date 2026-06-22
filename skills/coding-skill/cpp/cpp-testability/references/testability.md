# C++可测试性规范详细参考

## 1. 依赖注入

### 1.1 构造函数注入

```cpp
class Database {
public:
    virtual ~Database() = default;
    virtual bool save(const std::string& key, const std::string& value) = 0;
    virtual std::string load(const std::string& key) = 0;
};

class UserService {
private:
    std::shared_ptr<Database> db_;
public:
    explicit UserService(std::shared_ptr<Database> db) : db_(db) {}
    
    bool saveUser(const User& user) {
        return db_->save(user.id(), user.serialize());
    }
};

class MockDatabase : public Database {
public:
    MOCK_METHOD(bool, save, (const std::string&, const std::string&), (override));
    MOCK_METHOD(std::string, load, (const std::string&), (override));
};

TEST(UserServiceTest, SaveUserSuccess) {
    auto mockDb = std::make_shared<MockDatabase>();
    EXPECT_CALL(*mockDb, save("user1", "data"))
        .WillOnce(Return(true));
    
    UserService service(mockDb);
    EXPECT_TRUE(service.saveUser(User("user1", "data")));
}
```

### 1.2 setter注入

```cpp
class ReportGenerator {
private:
    IFormatter* formatter_ = nullptr;
public:
    void setFormatter(IFormatter* formatter) {
        formatter_ = formatter;
    }
    
    std::string generate(const Data& data) {
        return formatter_ ? formatter_->format(data) : "";
    }
};
```

## 2. 接口抽象

### 2.1 接口设计原则

```cpp
class IFileReader {
public:
    virtual ~IFileReader() = default;
    virtual std::string read(const std::string& path) = 0;
    virtual bool exists(const std::string& path) = 0;
};

class FileReader : public IFileReader {
public:
    std::string read(const std::string& path) override {
        std::ifstream file(path);
        return std::string((std::istreambuf_iterator<char>(file)),
                           std::istreambuf_iterator<char>());
    }
    
    bool exists(const std::string& path) override {
        return std::filesystem::exists(path);
    }
};
```

## 3. 避免全局状态

### 3.1 错误示例

```cpp
int g_counter = 0;

void increment() {
    g_counter++;
}

TEST(CounterTest, Increment) {
    increment();
    EXPECT_EQ(g_counter, 1);
}
```

### 3.2 正确示例

```cpp
class Counter {
private:
    int count_ = 0;
public:
    void increment() { count_++; }
    int get() const { return count_; }
};

TEST(CounterTest, Increment) {
    Counter counter;
    counter.increment();
    EXPECT_EQ(counter.get(), 1);
}
```

## 4. Google Test框架

### 4.1 基本测试

```cpp
TEST(MathTest, AddTwoNumbers) {
    EXPECT_EQ(2 + 2, 4);
    EXPECT_NE(2 + 2, 5);
    EXPECT_GT(5, 3);
    EXPECT_LT(1, 2);
    EXPECT_LE(2, 2);
    EXPECT_GE(3, 3);
}

TEST(StringTest, StringOperations) {
    std::string str = "hello";
    EXPECT_STREQ(str.c_str(), "hello");
    EXPECT_STRCASEEQ("Hello", "HELLO");
    EXPECT_TRUE(str.find("ell") != std::string::npos);
}
```

### 4.2 测试夹具

```cpp
class StackTest : public ::testing::Test {
protected:
    Stack<int> stack_;
    
    void SetUp() override {
        stack_.push(1);
        stack_.push(2);
        stack_.push(3);
    }
    
    void TearDown() override {
        while (!stack_.empty()) {
            stack_.pop();
        }
    }
};

TEST_F(StackTest, PopReturnsTopElement) {
    EXPECT_EQ(stack_.pop(), 3);
    EXPECT_EQ(stack_.size(), 2);
}

TEST_F(StackTest, EmptyAfterPopAll) {
    stack_.pop();
    stack_.pop();
    stack_.pop();
    EXPECT_TRUE(stack_.empty());
}
```

### 4.3 参数化测试

```cpp
class PrimeTest : public ::testing::TestWithParam<int> {};

TEST_P(PrimeTest, IsPrime) {
    int n = GetParam();
    EXPECT_TRUE(isPrime(n));
}

INSTANTIATE_TEST_SUITE_P(
    PrimeNumbers,
    PrimeTest,
    ::testing::Values(2, 3, 5, 7, 11, 13, 17, 19, 23, 29)
);
```

## 5. Google Mock使用

### 5.1 基本Mock

```cpp
class MockNetworkClient : public INetworkClient {
public:
    MOCK_METHOD(HttpResponse, get, (const std::string& url), (override));
    MOCK_METHOD(HttpResponse, post, (const std::string& url, const std::string& body), (override));
    MOCK_METHOD(void, setTimeout, (int ms), (override));
};

TEST(ApiClientTest, FetchDataSuccess) {
    MockNetworkClient mockClient;
    
    HttpResponse response{200, "{\"data\":\"value\"}"};
    EXPECT_CALL(mockClient, get("https://api.example.com/data"))
        .WillOnce(Return(response));
    
    ApiClient api(&mockClient);
    auto result = api.fetchData();
    EXPECT_EQ(result, "{\"data\":\"value\"}");
}
```

### 5.2 匹配器

```cpp
using ::testing::_;
using ::testing::Contains;
using ::testing::HasSubstr;
using ::testing::StartsWith;
using ::testing::Field;
using ::testing::Property;

TEST(MatcherTest, VariousMatchers) {
    MockDatabase mockDb;
    
    EXPECT_CALL(mockDb, save(_, _))
        .WillRepeatedly(Return(true));
    
    EXPECT_CALL(mockDb, load(StartsWith("user_")))
        .WillRepeatedly(Return("data"));
    
    EXPECT_CALL(mockDb, save("admin", HasSubstr("password")))
        .WillOnce(Return(false));
}
```

### 5.3 动作

```cpp
using ::testing::Return;
using ::testing::ReturnRef;
using ::testing::SetArgReferee;
using ::testing::Invoke;
using ::testing::DoAll;

TEST(ActionTest, ComplexActions) {
    MockProcessor mockProc;
    std::vector<int> data;
    
    EXPECT_CALL(mockProc, process(_))
        .WillOnce(Invoke([](const std::string& input) {
            return "processed: " + input;
        }));
    
    EXPECT_CALL(mockProc, getData(_))
        .WillOnce(SetArgReferee<0>(std::vector<int>{1, 2, 3}));
}
```

## 6. 测试替身

### 6.1 Stub（桩）

```cpp
class StubLogger : public ILogger {
public:
    void log(LogLevel level, const std::string& msg) override {}
    bool isEnabled() const override { return false; }
};

TEST(ServiceTest, ProcessWithStubLogger) {
    StubLogger stub;
    Service service(&stub);
    service.process();
}
```

### 6.2 Fake（假对象）

```cpp
class FakeDatabase : public IDatabase {
private:
    std::map<std::string, std::string> data_;
public:
    bool save(const std::string& key, const std::string& value) override {
        data_[key] = value;
        return true;
    }
    
    std::string load(const std::string& key) override {
        auto it = data_.find(key);
        return it != data_.end() ? it->second : "";
    }
};

TEST(UserRepositoryTest, SaveAndLoad) {
    FakeDatabase fakeDb;
    UserRepository repo(&fakeDb);
    
    repo.save(User{"1", "Alice"});
    auto user = repo.load("1");
    
    EXPECT_EQ(user.name(), "Alice");
}
```

### 6.3 Spy（间谍）

```cpp
class SpyEmailSender : public IEmailSender {
public:
    std::vector<std::string> sentTo;
    std::vector<std::string> sentContent;
    
    void send(const std::string& to, const std::string& content) override {
        sentTo.push_back(to);
        sentContent.push_back(content);
    }
};

TEST(NotificationTest, SendWelcomeEmail) {
    SpyEmailSender spy;
    NotificationService service(&spy);
    
    service.sendWelcome("user@example.com");
    
    EXPECT_EQ(spy.sentTo.size(), 1);
    EXPECT_EQ(spy.sentTo[0], "user@example.com");
    EXPECT_TRUE(spy.sentContent[0].find("Welcome") != std::string::npos);
}
```

## 7. 单元测试最佳实践

### 7.1 AAA模式

```cpp
TEST(CalculatorTest, AddTwoNumbers) {
    Calculator calc;
    
    int result = calc.add(2, 3);
    
    EXPECT_EQ(result, 5);
}
```

### 7.2 单一职责

```cpp
TEST(UserValidatorTest, ValidateEmail) {
    UserValidator validator;
    EXPECT_TRUE(validator.isValidEmail("test@example.com"));
}

TEST(UserValidatorTest, ValidateEmptyEmail) {
    UserValidator validator;
    EXPECT_FALSE(validator.isValidEmail(""));
}

TEST(UserValidatorTest, ValidateInvalidEmail) {
    UserValidator validator;
    EXPECT_FALSE(validator.isValidEmail("invalid"));
}
```

### 7.3 测试隔离

```cpp
class FileProcessorTest : public ::testing::Test {
protected:
    std::unique_ptr<TempDirectory> tempDir_;
    
    void SetUp() override {
        tempDir_ = std::make_unique<TempDirectory>();
    }
    
    void TearDown() override {
        tempDir_.reset();
    }
};
```

## 8. 检查清单

### 8.1 设计检查

- [ ] 是否使用依赖注入而非硬编码依赖
- [ ] 是否为外部依赖定义接口抽象
- [ ] 是否避免使用全局变量和单例
- [ ] 类是否可独立实例化和测试
- [ ] 是否使用工厂模式创建复杂对象

### 8.2 测试质量检查

- [ ] 每个测试是否只验证一个行为
- [ ] 测试是否遵循AAA模式
- [ ] 测试是否独立，不依赖执行顺序
- [ ] 是否在SetUp/TearDown中正确初始化和清理
- [ ] 测试名称是否清晰描述场景和预期

### 8.3 Mock使用检查

- [ ] 是否正确使用测试替身类型
- [ ] Mock期望是否设置了合理的调用次数
- [ ] 是否使用了适当的匹配器
- [ ] 是否避免了过度Mock

### 8.4 覆盖率检查

- [ ] 单元测试覆盖率是否达到80%以上
- [ ] 关键业务逻辑是否100%覆盖
- [ ] 边界条件和异常分支是否测试
- [ ] 是否同时关注分支覆盖而非仅行覆盖

### 8.5 GTest/GMock检查

- [ ] 是否使用EXPECT_*进行非致命断言
- [ ] 是否使用ASSERT_*进行致命断言
- [ ] 测试夹具是否正确使用SetUp/TearDown
- [ ] 参数化测试是否覆盖多种输入场景
- [ ] Mock对象是否在测试结束时自动销毁
