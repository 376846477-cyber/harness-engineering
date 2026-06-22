---
name: python-performance
description: Python编码规范 - 高效性规范。适用于所有Python代码开发场景：(1) 代码编写时考虑性能；(2) 代码审查时检查性能；(3) 算法、生成器、异步优化。触发关键词：Python、高效、performance、性能、优化、算法、生成器、异步、GIL
---

# Python高效性规范

基于Clean Code指导书

## 核心原则

在不影响其他代码质量属性的前提下，养成编写高效代码的习惯，**不要过早做性能优化**。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 选择高效的数据结构 |
| 1.2 | 使用生成器处理大数据 |
| 1.3 | 减少冗余或无效计算 |
| 1.4 | 合理使用异步编程 |
| 2.1 | 减少内存占用 |
| 3.1 | 减少IO次数 |
| 4.1 | 利用内置函数和标准库 |

## 数据结构选择

```python
# 使用set进行O(1)查找
valid_ids = set(user_ids)
if target_id in valid_ids:  # O(1)
    process(target_id)

# 使用dict而非列表查找
user_map = {u.id: u for u in users}
user = user_map.get(target_id)  # O(1)

# 使用collections优化
from collections import Counter, defaultdict
counts = Counter(items)
groups = defaultdict(list)
```

## 生成器

```python
# 使用生成器处理大数据
def process_large_file(path):
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            yield process_line(line)

# 生成器表达式
total = sum(item.price for item in items if item.is_valid())
```

## 详细规范

见 [references/performance.md](references/performance.md)

## 检查清单

- [ ] 是否使用了合适的数据结构？
- [ ] 大数据是否使用生成器？
- [ ] 是否有重复计算可以缓存？
- [ ] IO操作是否使用异步？
- [ ] 是否利用了内置函数？
