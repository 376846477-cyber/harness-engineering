---
name: python-clean-code
description: Clean Code总览 - Python编码规范的七个核心特征概览及索引。适用于所有Python代码开发场景：(1) 了解Clean Code整体框架；(2) 根据具体特征快速查找对应规范；(3) 代码审查时综合检查。触发关键词：Python、Clean Code、规范、特征、可读、可维护、安全、可靠、可测试、高效、可移植
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

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 可读性优先于性能优化，过早优化是万恶之源 |
| 规则1.2 | 安全性不可妥协，宁可牺牲性能也要保证安全 |
| 规则2.1 | 通过抽象隔离变化，依赖抽象而非实现 |
| 规则2.2 | 高内聚低耦合，单一职责原则 |
| 规则3.1 | 永远不信任外部输入，使用白名单校验 |
| 规则3.2 | 敏感数据不硬编码，使用环境变量或密钥管理 |
| 规则4.1 | 防御式编程，类型提示+参数校验 |
| 规则4.2 | 资源使用with语句管理，确保正确释放 |
| 规则5.1 | 依赖注入，减少全局状态 |
| 规则5.2 | 结果和过程可观测，日志+指标监控 |
| 规则6.1 | 测量后优化，使用profiler定位瓶颈 |
| 规则6.2 | 选择正确的算法和数据结构 |
| 规则7.1 | 使用pathlib处理路径，明确指定字符编码 |
| 规则7.2 | 优先使用标准库和跨平台库 |

## 错误/正确对比示例

**错误示例** - 违反多特征：
```python
def get_data(x):
    d = {}
    f = open("data.json")
    try:
        d = json.load(f)
    except:
        pass
    return d.get(x)
```

**正确示例** - 符合Clean Code各特征：
```python
from pathlib import Path
from typing import Any, Optional
import logging

logger = logging.getLogger(__name__)

def get_data(key: str, file_path: Path = Path("data.json")) -> Optional[Any]:
    if not key:
        raise ValueError("key不能为空")
    try:
        with file_path.open("r", encoding="utf-8") as f:
            data: dict = json.load(f)
    except FileNotFoundError:
        logger.warning(f"数据文件不存在: {file_path}")
        return None
    except json.JSONDecodeError as e:
        logger.error(f"数据文件格式错误: {e}")
        return None
    return data.get(key)
```

## 规范速查表

| 场景 | 使用Skill |
|-----|----------|
| 命名检查 | `python-naming` |
| 格式排版 | `python-formatting` |
| 注释规范 | `python-comments` |
| 异常处理 | `python-exceptions` |
| 并发编程 | `python-concurrency` |
| 输入安全 | `python-security-input` |
| 加密安全 | `python-security-crypto` |
| 序列化安全 | `python-security-serialization` |
| 可读性 | `python-readability` |
| 可维护性 | `python-maintainability` |
| 可靠性 | `python-reliability` |
| 可测试性 | `python-testability` |
| 高效性 | `python-performance` |
| 可移植性 | `python-portability` |

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
- [ ] 命名是否清晰表达意图？
- [ ] 输入是否经过校验？
- [ ] 资源是否使用with语句管理？
