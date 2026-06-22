# 接口定义规范

定义清晰的接口契约，确保调用者和实现者有共同的理解。

## 接口契约要素

### 完整接口契约

一个完整的接口契约包含以下要素：

```
┌─────────────────────────────────────────────────────────────┐
│                       接口契约                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 类型签名                                                │
│     • 方法名和参数类型                                      │
│     • 返回值类型                                            │
│                                                             │
│  2. 前置条件（Preconditions）                               │
│     • 调用前必须满足的条件                                  │
│     • 调用者责任                                            │
│                                                             │
│  3. 后置条件（Postconditions）                              │
│     • 调用后保证的状态                                      │
│     • 实现者责任                                            │
│                                                             │
│  4. 不变量（Invariants）                                    │
│     • 始终保持为真的条件                                    │
│                                                             │
│  5. 错误模式                                                │
│     • 可能抛出的错误                                        │
│     • 错误的含义和处理方式                                  │
│                                                             │
│  6. 性能特征                                                │
│     • 时间复杂度                                            │
│     • 空间复杂度                                            │
│     • 响应时间要求                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 契约示例

```typescript
/**
 * 订单服务接口
 */
interface OrderService {
  /**
   * 创建订单
   * 
   * @param request - 订单创建请求
   * @returns 订单ID和订单号
   * 
   * @precondition
   *   - 用户已登录
   *   - 购物车不为空
   *   - 收货地址有效
   * 
   * @postcondition
   *   - 订单已持久化
   *   - 库存已锁定
   *   - 订单状态为 PENDING
   * 
   * @invariant
   *   - 订单号唯一
   *   - 订单金额 = 商品金额之和 - 优惠金额
   * 
   * @throws
   *   - OutOfStockError: 库存不足
   *   - ProductNotAvailableError: 商品已下架
   *   - InvalidAddressError: 地址无效
   * 
   * @performance
   *   - 平均响应时间 < 500ms
   *   - 支持 1000 QPS
   */
  createOrder(request: CreateOrderRequest): Promise<CreateOrderResponse>;
}
```

## 接口设计原则

### 1. 最小惊讶原则

接口行为应符合直觉预期。

```
好的设计：
  user.getName() → 返回用户名称
  list.size() → 返回列表大小

不好的设计：
  user.getName() → 修改用户名称并返回
  list.size() → 清空列表并返回之前的大小
```

### 2. 单一职责原则

每个接口方法只做一件事。

```
好的设计：
  createOrder() → 只创建订单
  payOrder() → 只支付订单
  cancelOrder() → 只取消订单

不好的设计：
  createAndPayOrder() → 创建并支付（职责混合）
```

### 3. 接口隔离原则

不应强迫客户依赖它们不使用的方法。

```
不好的设计：
interface UserService {
  createUser();
  updateUser();
  deleteUser();
  exportUsers();    // 管理后台专用
  batchImport();    // 管理后台专用
}

好的设计：
interface UserService {
  createUser();
  updateUser();
  deleteUser();
}

interface UserManagement {
  exportUsers();
  batchImport();
}
```

### 4. 依赖倒置原则

高层模块依赖抽象，不依赖具体实现。

```
不好的设计：
class OrderService {
  private mysqlOrderRepo: MySQLOrderRepository;  // 直接依赖具体实现
}

好的设计：
class OrderService {
  private orderRepo: OrderRepository;  // 依赖抽象接口
}
```

## 接口粒度

### 细粒度 vs 粗粒度

| 粒度 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| 细粒度 | 灵活、可组合 | 调用次数多、网络开销大 | 内部服务、领域服务 |
| 粗粒度 | 调用简单、网络开销小 | 灵活性差 | 外部 API、跨服务调用 |

### 粒度选择指南

```
判断标准：

1. 调用者是谁？
   - 内部服务 → 偏细粒度
   - 外部系统 → 偏粗粒度
   - 前端应用 → 中等粒度

2. 网络开销重要吗？
   - 局部调用 → 细粒度可接受
   - 跨网络 → 粗粒度更好

3. 需要灵活性吗？
   - 需要灵活组合 → 细粒度
   - 固定流程 → 粗粒度
```

## 接口命名规范

### 方法命名

| 操作类型 | 命名前缀 | 示例 |
|----------|----------|------|
| 查询单个 | get/find | getUser, findById |
| 查询列表 | list/search | listUsers, searchOrders |
| 创建 | create | createUser |
| 更新 | update | updateUser |
| 删除 | delete/remove | deleteUser |
| 判断 | is/has/can | isActive, hasPermission |
| 计算 | calculate | calculateTotal |
| 验证 | validate/check | validateEmail |

### 参数命名

| 类型 | 命名规范 | 示例 |
|------|----------|------|
| ID | xxxId | userId, orderId |
| 名称 | xxxName | userName, productName |
| 类型 | xxxType | userType, orderType |
| 状态 | xxxStatus | orderStatus |
| 时间 | xxxAt/xxxTime | createdAt, expireTime |
| 数量 | xxxCount/quantity | itemCount |
| 列表 | xxxs/xxxList | users, orderList |

## 接口文档模板

```markdown
# [接口名称]

## 概述

[简要描述接口的功能和用途]

## 接口定义

### 方法签名

```
[方法签名]
```

### 参数说明

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| ... | ... | ... | ... |

### 返回值

| 字段名 | 类型 | 说明 |
|--------|------|------|
| ... | ... | ... |

## 契约

### 前置条件

- [条件1]
- [条件2]

### 后置条件

- [条件1]
- [条件2]

### 不变量

- [不变量1]

### 错误

| 错误类型 | 触发条件 | 处理建议 |
|----------|----------|----------|
| ... | ... | ... |

## 示例

### 正常调用

```
[示例代码]
```

### 错误处理

```
[错误处理示例]
```

## 注意事项

- [注意事项1]
- [注意事项2]
```
