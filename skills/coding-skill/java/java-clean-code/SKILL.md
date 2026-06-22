---
name: java-clean-code
description: Clean Code总览 - Java编码规范的七个核心特征概览及索引。适用于所有Java代码开发场景：(1) 了解Clean Code整体框架；(2) 根据具体特征快速查找对应规范；(3) 代码审查时综合检查。触发关键词：Java、Clean Code、规范、特征、可读、可维护、安全、可靠、可测试、高效、可移植
---

# Clean Code总览

基于Clean Code指导书（IPD_MPD_SWD_W375913）

## 概述

Clean Code是指在满足功能正确的前提下，把具有以下**七个特征**的代码称为Clean Code：

| 特征 | 定义 | 优先级 |
|-----|------|-------|
| **可读** | 易于阅读 | 高 |
| **可维护** | 可修改、可扩展、可复用 | 高 |
| **安全** | 防御恶意威胁 | 高 |
| **可靠** | 正常运行概率 | 中 |
| **可测试** | 易于发现和定位故障 | 中 |
| **高效** | 资源利用率高 | 中 |
| **可移植** | 跨平台运行 | 低 |

## 特征速查

### 1. 可读（Readability）
- 传递足够多的有用信息
- 减少信息失真（误解）
- 减少信息干扰
- 降低信息理解难度

### 2. 可维护（Maintainability）
- 修改/扩展影响局部化
- 通过抽象隔离变化
- 向着稳定的方向依赖

### 3. 安全（Security）
- 减少攻击面
- 输入校验及转义
- 敏感数据保护

### 4. 可靠（Reliability）
- 预防错误
- 容错
- 故障恢复（自愈）

### 5. 可测试（Testability）
- 可隔离
- 可控制
- 可观测
- 可定位

### 6. 高效（Performance）
- CPU资源利用
- 内存优化
- IO优化
- 数据库优化

### 7. 可移植（Portability）
- 避免平台相关API
- 使用标准API
- 明确指定字符集

## 规范速查表

| 场景 | 使用Skill |
|-----|----------|
| 命名检查 | `java-naming` |
| 格式排版 | `java-formatting` |
| 注释规范 | `java-comments` |
| 异常处理 | `java-exceptions` |
| 多线程并发 | `java-concurrency` |
| 输入安全 | `java-security-input` |
| 加密安全 | `java-security-crypto` |
| 序列化安全 | `java-security-serialization` |
| 可读性 | `java-readability` |
| 可维护性 | `java-maintainability` |
| 可靠性 | `java-reliability` |
| 可测试性 | `java-testability` |
| 高效性 | `java-performance` |
| 可移植性 | `java-portability` |

## 特征权衡

建议优先级：**功能正确 > 可读 > 可维护 > 安全 > 可靠 > 可测试 > 高效 > 可移植**

## 详细规范

见 [references/clean-code.md](references/clean-code.md)

## 检查清单

- [ ] 代码是否易于阅读和理解？
- [ ] 修改是否影响局部化（最小化影响范围）？
- [ ] 是否有安全漏洞（注入、加密等）？
- [ ] 是否有容错和自愈机制？
- [ ] 代码是否易于测试？
- [ ] 是否有不必要的性能浪费？
- [ ] 是否依赖平台特定的API？