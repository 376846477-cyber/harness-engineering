# 项目规范

本项目使用Clean Code规范进行Python开发。

## 自动应用的规范

在编写Python代码时，请自动应用以下编码规范：

1. **可读性** - 代码应Pythonic，名称传递信息，格式一致，注释作为补充
2. **可维护性** - 通过抽象隔离变化，依赖抽象而非实现，高内聚低耦合
3. **可靠性** - 防御式编程，资源正确管理，容错和自愈机制
4. **可测试性** - 依赖注入，高内聚低耦合，结果可观测
5. **高效性** - 选择高效算法，合理使用数据结构，善用生成器和异步
6. **可移植性** - 使用标准库，明确指定编码，使用pathlib处理路径
7. **安全性** - 输入校验，敏感数据保护，使用安全函数
8. **命名规范** - 函数/变量snake_case，类名PascalCase，常量UPPER_SNAKE_CASE
9. **格式规范** - 4空格缩进，遵循PEP 8
10. **注释规范** - Docstring文档注释，注释与代码同步

## 详细规范

如需查看特定规范的详细内容，可以使用以下命令：
- /python-clean-code - 总览所有特征
- /python-readability - 可读性规范
- /python-maintainability - 可维护性规范
- /python-reliability - 可靠性规范
- /python-testability - 可测试性规范
- /python-performance - 高效性规范
- /python-portability - 可移植性规范
- /python-naming - 命名规范
- /python-formatting - 格式规范
- /python-comments - 注释规范
- /python-exceptions - 异常处理规范
- /python-concurrency - 并发规范
- /python-security-input - 输入安全规范
- /python-security-crypto - 加密安全规范
- /python-security-serialization - 序列化安全规范

## 代码审查检查清单

完成代码后，检查以下要点：
- [ ] 名称是否清晰传递信息？
- [ ] 是否遵循单一职责原则？
- [ ] 是否有适当的安全防护？
- [ ] 异常是否正确处理？
- [ ] 资源是否正确释放（with语句）？
- [ ] 代码是否易于测试？