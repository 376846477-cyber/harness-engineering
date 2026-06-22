## 项目简介

**Clean Code Java Skills** 是一套完整的Claude Code技能（Skills）集合，专门用于在Java代码开发过程中应用编码规范。

本项目将Clean Code指导书的核心内容转换为可自动触发的Skill，帮助开发者在编写Java代码时自动遵循编码规范，提升代码质量。

## 规范来源

- **Clean Code指导书** (IPD_MPD_SWD_W375913)
- **Java语言通用编程规范**
- **安全编程规范**

## 包含的Skills

本项目包含 **15个** Java编码规范Skill：

### 基础规范（5个）

| Skill                     | 描述                                       |
| ------------------------- | ------------------------------------------ |
| `java-naming`      | 命名规范 - 类名、方法名、变量名、常量等    |
| `java-formatting`  | 排版格式规范 - 缩进、大括号、换行等        |
| `java-comments`    | 注释规范 - Javadoc、文件头、代码注释       |
| `java-exceptions`  | 异常处理规范 - try-catch、抛出、自定义异常 |
| `java-concurrency` | 多线程并发规范 - 线程同步、锁、ThreadLocal |

### 安全规范（3个）

| Skill                                | 描述                                         |
| ------------------------------------ | -------------------------------------------- |
| `java-security-input`         | 输入校验与安全 - SQL注入、命令注入、文件安全 |
| `java-security-crypto`        | 加密安全 - 密码存储、加密算法、SSL/TLS       |
| `java-security-serialization` | 序列化安全 - Serializable、反序列化防护      |

### Clean Code特征（7个）

| Skill                         | 描述                              | 优先级 |
| ----------------------------- | --------------------------------- | ------ |
| `java-readability`     | 可读性 - 名称传递信息、格式、注释 | 高     |
| `java-maintainability` | 可维护性 - 抽象、依赖、耦合       | 高     |
| `java-reliability`     | 可靠性 - 防御式编程、容错、自愈   | 中     |
| `java-testability`     | 可测试性 - 隔离、可控、可观测     | 中     |
| `java-performance`     | 高效性 - 算法、内存、并行         | 中     |
| `java-portability`     | 可移植性 - 平台无关、跨平台       | 低     |
| `java-clean-code`      | 总览 - 七个特征概览及索引         | 汇总   |

## 快速入门

### 安装：复制到Claude Code Skills目录

```bash
# 将 java-* 目录复制到 Claude Code 的 skills 目录
cp -r java-* ~/.claude/skills/
```

**Skills目录路径**：

- Windows: `C:\Users\<用户名>\.claude\skills\`
- macOS/Linux: `~/.claude/skills/`

### 手动触发

在对话中直接输入技能名称：

```
/java-naming
/java-readability
/java-security-input
```

### 自动触发

当对话中包含相关关键词时，Skill会自动加载：

- 提到"命名"、"camelCase" → 触发 `java-naming`
- 提到"可读"、"易读" → 触发 `java-readability`
- 提到"SQL注入"、"安全" → 触发 `java-security-input`
- 提到"性能"、"优化" → 触发 `java-performance`

### 优秀实践

可以将CLAUDE.md放入项目根目录，指导模型在开发阶段选择到对应的SKILL。

## 如何反馈

欢迎提交Issue和Pull Request来改进本技能集。


