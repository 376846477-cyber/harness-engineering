# 华为Java注释规范详细参考

## 规则2.1 Javadoc使用

- 公开的类、接口、方法、字段应使用Javadoc
- Javadoc注释以`/**`开头，以`*/`结尾
- 每个Javadoc块包含功能描述、@param、@return、@throws等标签

## 规则2.2 文件头注释

### 要求内容
- 版权声明
- 功能描述
- 作者
- 创建/修改日期

### 示例
```java
/**
 * Copyright (C) 华为技术有限公司 版权所有
 *
 * 功能：提供用户管理的核心服务，包括用户的增删改查等操作
 *
 * 作者：张三 2024-01-01
 * 修改：李四 2024-02-01 添加xxx方法
 */
public class UserService {}
```

## 规则2.3 方法头注释

### 要求内容
- 功能说明
- 参数说明（@param）
- 返回值说明（@return）
- 异常说明（@throws）

### 示例
```java
/**
 * 根据用户ID查询用户信息
 *
 * @param userId 用户ID，不能为空
 * @return 用户信息实体，查询不到返回null
 * @throws IllegalArgumentException userId为空时抛出
 * @throws UserNotFoundException 用户不存在时抛出
 */
public User getUserById(Long userId) {}
```

## 规则2.4 注释语言

- 使用中文注释
- 英文术语保持英文
- 保持注释与代码同步更新

## 规则2.5 代码内注释

- 解释**为什么**而不是**做什么**
- 复杂业务逻辑需要注释
- 关键算法需要注释
- TODO注释标注待完成事项

## 注释检查清单

1. ✅ 公开API有Javadoc
2. ✅ 文件头有版权和功能说明
3. ✅ 方法有参数和返回值说明
4. ✅ 使用中文注释
5. ✅ 注释与代码同步
6. ✅ 关键逻辑有注释说明