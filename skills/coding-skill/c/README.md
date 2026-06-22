# C语言编码规范Skill

## 规范来源

- **Clean Code指导书**
- **MISRA C:2012**
- **SEI CERT C Coding Standard**
- **GNU Coding Standards**
- **C安全编程规范**

## 包含的Skills

### 基础规范（5个）

| Skill | 描述 |
|-------|------|
| `c-naming` | 命名规范 - 文件名、函数名、变量名、常量、宏 |
| `c-formatting` | 排版格式规范 - 缩进、空行、行宽、include排序 |
| `c-comments` | 注释规范 - 函数注释、文件注释、行注释 |
| `c-error-handling` | 错误处理规范 - 错误码、errno、goto清理 |
| `c-concurrency` | 并发规范 - pthread、mutex、信号量 |

### 安全规范（3个）

| Skill | 描述 |
|-------|------|
| `c-security-input` | 输入校验与安全规范 - 缓冲区溢出、注入防护 |
| `c-security-crypto` | 加密安全规范 - 加密算法、密钥管理、密码存储 |
| `c-security-serialization` | 序列化安全规范 - 数据校验、格式安全 |

### Clean Code特征规范（7个）

| Skill | 描述 |
|-------|------|
| `c-readability` | 可读性规范 - 命名、格式、注释、惯用法 |
| `c-maintainability` | 可维护性规范 - 模块化、接口、耦合 |
| `c-reliability` | 可靠性规范 - 防御式编程、容错、资源管理 |
| `c-testability` | 可测试性规范 - 依赖注入、单元测试 |
| `c-performance` | 高效性规范 - 算法、内存管理、优化 |
| `c-portability` | 可移植性规范 - 跨平台、编译器兼容 |
| `c-clean-code` | Clean Code总览 - 七个核心特征概览及索引 |

## 使用方式

在CLAUDE.md中引入本skill，模型将自动在C语言代码开发过程中应用相应规范。
