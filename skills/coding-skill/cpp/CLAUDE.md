# C++编码规范

## 自动应用的规范

1. **命名规范** - 函数/变量snake_case，类名PascalCase，常量kCamelCase，宏UPPER_SNAKE_CASE
2. **格式规范** - 缩进4空格，行宽120字符，大括号换行
3. **注释规范** - Doxygen文档注释，注释与代码同步
4. **异常规范** - 使用RAII管理资源，异常安全保证
5. **并发规范** - 使用std::mutex/std::atomic保护共享状态
6. **安全规范** - 禁止缓冲区溢出，输入校验，敏感数据保护
7. **内存规范** - 优先使用智能指针，避免裸new/delete
8. **现代C++** - 优先使用C++17/20特性

## 详细规范

- `/cpp-naming` - 命名规范
- `/cpp-formatting` - 格式规范
- `/cpp-comments` - 注释规范
- `/cpp-exceptions` - 异常处理规范
- `/cpp-concurrency` - 并发规范
- `/cpp-security-input` - 输入校验与安全规范
- `/cpp-security-crypto` - 加密安全规范
- `/cpp-security-serialization` - 序列化安全规范
- `/cpp-readability` - 可读性规范
- `/cpp-maintainability` - 可维护性规范
- `/cpp-reliability` - 可靠性规范
- `/cpp-testability` - 可测试性规范
- `/cpp-performance` - 高效性规范
- `/cpp-portability` - 可移植性规范
- `/cpp-clean-code` - Clean Code总览
