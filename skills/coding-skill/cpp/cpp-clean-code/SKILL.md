---
name: cpp-clean-code
description: Clean Code总览 - C++编码规范的七个核心特征概览及索引。适用于所有C++代码开发场景：(1) 了解Clean Code整体框架；(2) 根据具体特征快速查找对应规范；(3) 代码审查时综合检查。触发关键词：C++、Clean Code、规范、特征、可读、可维护、安全、可靠、可测试、高效、可移植
---

# Clean Code总览

基于Clean Code指导书

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

## 规范速查表

| 场景 | 使用Skill |
|-----|----------|
| 命名检查 | `cpp-naming` |
| 格式排版 | `cpp-formatting` |
| 注释规范 | `cpp-comments` |
| 异常处理 | `cpp-exceptions` |
| 并发编程 | `cpp-concurrency` |
| 输入安全 | `cpp-security-input` |
| 加密安全 | `cpp-security-crypto` |
| 序列化安全 | `cpp-security-serialization` |
| 可读性 | `cpp-readability` |
| 可维护性 | `cpp-maintainability` |
| 可靠性 | `cpp-reliability` |
| 可测试性 | `cpp-testability` |
| 高效性 | `cpp-performance` |
| 可移植性 | `cpp-portability` |

## 特征权衡

建议优先级：**功能正确 > 可读 > 可维护 > 安全 > 可靠 > 可测试 > 高效 > 可移植**

## 详细规范

见 [references/clean-code.md](references/clean-code.md)

## 检查清单

- [ ] 代码是否易于阅读和理解？
- [ ] 修改是否影响局部化？
- [ ] 是否有安全漏洞？
- [ ] 是否有容错和自愈机制？
- [ ] 代码是否易于测试？
- [ ] 是否有不必要的性能浪费？
- [ ] 是否依赖平台特定的API？
