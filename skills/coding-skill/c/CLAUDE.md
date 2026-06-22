# C语言编码规范

## 自动应用的规范

1. **命名规范** - 函数名模块前缀_snake_case，变量名snake_case，宏UPPER_SNAKE_CASE
2. **格式规范** - 缩进4空格，行宽120字符，大括号换行
3. **注释规范** - 函数头注释，注释与代码同步
4. **错误处理** - 检查返回值，使用goto做统一清理
5. **并发规范** - 使用pthread_mutex保护共享状态
6. **安全规范** - 禁止缓冲区溢出，输入校验，敏感数据保护
7. **内存规范** - 每个malloc对应free，避免内存泄露
8. **类型安全** - 使用size_t表示大小，避免隐式类型转换

## 详细规范

- `/c-naming` - 命名规范
- `/c-formatting` - 格式规范
- `/c-comments` - 注释规范
- `/c-error-handling` - 错误处理规范
- `/c-concurrency` - 并发规范
- `/c-security-input` - 输入校验与安全规范
- `/c-security-crypto` - 加密安全规范
- `/c-security-serialization` - 序列化安全规范
- `/c-readability` - 可读性规范
- `/c-maintainability` - 可维护性规范
- `/c-reliability` - 可靠性规范
- `/c-testability` - 可测试性规范
- `/c-performance` - 高效性规范
- `/c-portability` - 可移植性规范
- `/c-clean-code` - Clean Code总览
