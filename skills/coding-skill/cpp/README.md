# C++编码规范Skill

## 规范来源

- **Clean Code指导书**
- **C++ Core Guidelines**
- **Google C++ Style Guide**
- **SEI CERT C++ Coding Standard**
- **C++安全编程规范**

## 包含的Skills

### 基础规范（5个）

| Skill | 描述 |
|-------|------|
| `cpp-naming` | 命名规范 - 命名空间、类名、函数名、变量名、常量、宏 |
| `cpp-formatting` | 排版格式规范 - 缩进、空行、行宽、include排序 |
| `cpp-comments` | 注释规范 - Doxygen文档注释、行注释、块注释 |
| `cpp-exceptions` | 异常处理规范 - try/catch、RAII、错误码 |
| `cpp-concurrency` | 并发规范 - std::thread、mutex、atomic、future |

### 安全规范（3个）

| Skill | 描述 |
|-------|------|
| `cpp-security-input` | 输入校验与安全规范 - 缓冲区溢出、注入防护 |
| `cpp-security-crypto` | 加密安全规范 - 加密算法、密钥管理、密码存储 |
| `cpp-security-serialization` | 序列化安全规范 - 反序列化安全、数据校验 |

### Clean Code特征规范（7个）

| Skill | 描述 |
|-------|------|
| `cpp-readability` | 可读性规范 - 命名、格式、注释、惯用法 |
| `cpp-maintainability` | 可维护性规范 - 抽象、SOLID、设计模式 |
| `cpp-reliability` | 可靠性规范 - RAII、防御式编程、容错 |
| `cpp-testability` | 可测试性规范 - 依赖注入、GTest、Mock |
| `cpp-performance` | 高效性规范 - 算法、内存管理、移动语义 |
| `cpp-portability` | 可移植性规范 - 跨平台、编译器兼容 |
| `cpp-clean-code` | Clean Code总览 - 七个核心特征概览及索引 |

## 使用方式

在CLAUDE.md中引入本skill，模型将自动在C++代码开发过程中应用相应规范。
