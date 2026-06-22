---
name: java-performance
description: Java编码规范 - 高效性规范。适用于所有Java代码开发场景：(1) 代码编写时考虑性能；(2) 代码审查时检查性能；(3) 算法、内存、并行优化；(4) 资源利用率提升。触发关键词：Java、高效、performance、性能、优化、算法、内存、缓存、并行
---

# Java高效性规范

基于Clean Code指导书第2章"高效"特征

## 核心原则

软件性能影响客户体验和硬件成本，性能问题甚至会导致功能受损。在不影响其他代码质量属性（可维护性、可移植性等）及对编码效率影响不大的前提下，养成编写高效代码的习惯，不要浪费资源，可以节省大量后期优化和问题处理的时间。**但开发阶段不提倡追求极致的性能，不要过早做性能优化。**

提升性能主要是提升系统硬件资源的利用率，常见硬件资源有**CPU、内存（含总线）、硬盘（含IO）、网络带宽**等，其中最核心的就是CPU和内存。

## 核心规则速查

| 规则 | 要求 |
|-----|-----|
| 1.1 | 选择高效的算法（Hash、二分、红黑树） |
| 1.2 | 空间换时间（合理使用缓存） |
| 1.3 | 减少CPU外设访问开销 |
| 1.4 | 减少冗余或无效计算（延迟加载） |
| 1.5 | 合理使用并行编程 |
| 2.1 | 减少内存浪费（合理选择数据结构） |
| 2.2 | 合理的内存生命周期管理 |
| 2.3 | 减少内存碎片（对象复用） |
| 3.1 | 减少IO次数（批量操作） |
| 3.2 | 异步IO提高吞吐量 |
| 4.1 | 减少数据库查询次数（JOIN、批量） |
| 4.2 | 正确使用索引 |

## 数据结构选择

```java
// ✅ 使用HashSet进行O(1)查找
Set<String> userIds = new HashSet<>(userList.stream()
    .map(User::getId)
    .collect(Collectors.toSet()));
boolean exists = userIds.contains(targetId);

// ❌ 列表遍历查找 - O(n)
for (User user : users) {
    if (user.getId().equals(targetId)) { return user; }
}

// ✅ 指定集合初始容量
List<User> users = new ArrayList<>(1000);
Set<String> uniqueNames = new HashSet<>(nameList);

// ✅ 使用基本类型避免装箱拆箱
int[] ids = new int[1000];  // vs Integer[]
```

## 循环优化

```java
// ✅ 避免重复计算
public void processItems(List<Item> items) {
    int size = items.size();  // 预先计算
    for (int i = 0; i < size; i++) {
        process(items.get(i));
    }
}

// ❌ 循环中重复调用
for (int i = 0; i < items.size(); i++) { }
```

## 详细规范

见 [references/performance.md](references/performance.md)

## 检查清单

- [ ] 是否使用了合适的数据结构（HashSet vs List）？
- [ ] 是否有重复计算可以缓存？
- [ ] 循环中是否有重复调用方法？
- [ ] 是否可以使用并行处理？
- [ ] 集合是否指定了初始容量？
- [ ] 是否使用了基本类型避免装箱？
- [ ] 资源是否正确释放（try-with-resources）？
- [ ] 数据库操作是否使用批量处理？