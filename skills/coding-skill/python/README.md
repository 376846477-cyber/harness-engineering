## 项目简介

**Clean Code Python Skills** 是一套完整的Claude Code技能（Skills）集合，专门用于在Python代码开发过程中应用编码规范。

本项目将Clean Code指导书的核心内容转换为可自动触发的Skill，帮助开发者在编写Python代码时自动遵循编码规范，提升代码质量。

## 规范来源

- **Clean Code指导书** (IPD_MPD_SWD_W375913)
- **PEP 8 -- Style Guide for Python Code**
- **PEP 257 -- Docstring Conventions**
- **Python安全编程规范**

## 包含的Skills

本项目包含 **15个** Python编码规范Skill：

### 基础规范（5个）

| Skill                       | 描述                                           |
| --------------------------- | ---------------------------------------------- |
| python-naming             | 命名规范 - 模块名、类名、函数名、变量名、常量 |
| python-formatting         | 排版格式规范 - 缩进、空行、行宽、import排序   |
| python-comments           | 注释规范 - Docstring、文件头、代码注释         |
| python-exceptions         | 异常处理规范 - try/except、自定义异常、上下文  |
| python-concurrency        | 并发规范 - threading、asyncio、multiprocessing |

### 安全规范（3个）

| Skill                                | 描述                                              |
| ------------------------------------ | ------------------------------------------------- |
| python-security-input              | 输入校验与安全 - SQL注入、命令注入、模板注入     |
| python-security-crypto             | 加密安全 - 密码哈希、加密算法、SSL/TLS           |
| python-security-serialization      | 序列化安全 - pickle安全、反序列化防护、YAML安全  |

### Clean Code特征（7个）

| Skill                         | 描述                                | 优先级 |
| ----------------------------- | ----------------------------------- | ------ |
| python-readability          | 可读性 - Pythonic、名称传递信息     | 高     |
| python-maintainability      | 可维护性 - 抽象、依赖、耦合         | 高     |
| python-reliability          | 可靠性 - 防御式编程、容错、自愈     | 中     |
| python-testability          | 可测试性 - pytest、mock、隔离       | 中     |
| python-performance          | 高效性 - 算法、生成器、异步         | 中     |
| python-portability          | 可移植性 - 跨平台、编码、路径处理   | 低     |
| python-clean-code           | 总览 - 七个特征概览及索引           | 汇总   |

## 快速入门

### 安装：复制到Claude Code Skills目录

`ash
cp -r python-* ~/.claude/skills/
`

**Skills目录路径**：

- Windows: C:\Users\<用户名>\.claude\skills\
- macOS/Linux: ~/.claude/skills/

### 手动触发

`
/python-naming
/python-readability
/python-security-input
`

### 自动触发

- 提到"命名"、"snake_case" → 触发 python-naming
- 提到"可读"、"Pythonic" → 触发 python-readability
- 提到"SQL注入"、"安全" → 触发 python-security-input
- 提到"性能"、"优化" → 触发 python-performance

### 优秀实践

可以将CLAUDE.md放入项目根目录，指导模型在开发阶段选择到对应的SKILL。

## 如何反馈

欢迎提交Issue和Pull Request来改进本技能集。