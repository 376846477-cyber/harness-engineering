# 入参与出参设计

设计清晰的接口参数和返回值结构。

## 入参设计

### 设计原则

| 原则 | 描述 | 示例 |
|------|------|------|
| 必填最小化 | 只要求必要信息 | 注册只需手机号+密码 |
| 语义清晰 | 参数名自解释 | `userId` 而非 `id` |
| 类型安全 | 使用明确类型 | 枚举而非字符串 |
| 默认合理 | 提供合理默认值 | `pageSize` 默认 20 |
| 验证前置 | 明确验证规则 | `@Min(1) @Max(100)` |

### 参数类型选择

| 类型 | 适用场景 | 示例 |
|------|----------|------|
| Path | 资源标识 | `/users/{userId}` |
| Query | 过滤、分页 | `?status=active&page=1` |
| Header | 元数据、认证 | `Authorization: Bearer xxx` |
| Body | 复杂数据 | 创建/更新请求体 |

### 请求体设计模式

**模式1：扁平结构**

```typescript
// 适用于简单场景
interface LoginRequest {
  username: string;
  password: string;
}
```

**模式2：嵌套结构**

```typescript
// 适用于复杂数据
interface CreateOrderRequest {
  items: OrderItem[];        // 商品列表
  shipping: ShippingInfo;    // 配送信息
  payment: PaymentInfo;      // 支付信息
}
```

**模式3：包装结构**

```typescript
// 适用于需要元数据的场景
interface ApiRequest<T> {
  data: T;                   // 业务数据
  metadata: RequestMetadata; // 元数据
}
```

### 参数验证

**验证规则定义：**

```typescript
interface CreateOrderRequest {
  // 商品列表
  @Required()
  @Size(min: 1, max: 100)
  items: OrderItem[];
  
  // 收货地址ID
  @Required()
  @Pattern(/^[a-zA-Z0-9]{16}$/)
  addressId: string;
  
  // 优惠券码
  @Optional()
  @Pattern(/^[A-Z0-9]{8,16}$/)
  couponCode?: string;
  
  // 订单备注
  @Optional()
  @Size(max: 200)
  remark?: string;
}
```

**验证规则模板：**

| 规则 | 说明 | 示例 |
|------|------|------|
| @Required | 必填 | `@Required()` |
| @Optional | 可选 | `@Optional()` |
| @Size | 长度/大小限制 | `@Size(min: 1, max: 100)` |
| @Pattern | 正则匹配 | `@Pattern(/^[a-z]+$/)` |
| @Min/@Max | 数值范围 | `@Min(0) @Max(100)` |
| @Email | 邮箱格式 | `@Email()` |
| @Phone | 手机号格式 | `@Phone()` |

## 出参设计

### 设计原则

| 原则 | 描述 | 示例 |
|------|------|------|
| 结构扁平 | 避免过深嵌套 | 最多 3 层 |
| 信息完整 | 返回足够信息 | 包含关联实体摘要 |
| 一致性 | 相同概念相同结构 | 用户信息结构一致 |
| 可扩展 | 预留扩展空间 | 使用对象而非基本类型 |

### 响应体设计模式

**模式1：统一响应结构**

```typescript
interface ApiResponse<T> {
  code: number;              // 业务状态码
  message: string;           // 状态信息
  data: T;                   // 业务数据
  traceId: string;           // 追踪ID
  timestamp: number;         // 时间戳
}
```

**模式2：分页响应**

```typescript
interface PageResponse<T> {
  code: number;
  message: string;
  data: {
    list: T[];               // 数据列表
    total: number;           // 总数
    page: number;            // 当前页
    pageSize: number;        // 每页大小
    hasMore: boolean;        // 是否有更多
  };
  traceId: string;
  timestamp: number;
}
```

**模式3：带摘要的响应**

```typescript
interface OrderDetailResponse {
  orderId: string;
  orderNo: string;
  status: OrderStatus;
  
  // 关联信息摘要（避免额外查询）
  user: UserSummary;
  items: OrderItemSummary[];
  address: AddressSummary;
  
  // 时间信息
  createdAt: string;
  updatedAt: string;
}

interface UserSummary {
  userId: string;
  nickname: string;
  avatar: string;
}

interface OrderItemSummary {
  productId: string;
  productName: string;
  productImage: string;
  quantity: number;
  price: number;
}
```

### 字段命名规范

| 类型 | 命名规范 | 示例 |
|------|----------|------|
| ID | xxxId | userId, orderId |
| 名称 | xxxName | userName, productName |
| 时间 | xxxAt | createdAt, updatedAt |
| 状态 | xxxStatus | orderStatus |
| 数量 | xxxCount | itemCount |
| 金额 | xxxAmount | totalAmount |
| 标志 | isXxx/hasXxx | isActive, hasDiscount |

### 空值处理

| 场景 | 推荐处理方式 |
|------|-------------|
| 可选字段未设置 | 返回 `null` 或不返回该字段 |
| 列表为空 | 返回 `[]` 而非 `null` |
| 对象为空 | 返回 `null` 或空对象 `{}` |
| 数值为空 | 返回 `null` 或默认值 `0` |

```typescript
// 推荐：明确区分空值
interface UserResponse {
  nickname: string;          // 必有值
  avatar: string | null;     // 可能为空
  bio?: string;              // 可选字段
  tags: string[];            // 空时返回 []
}
```

## 版本兼容

### 字段新增

```
v1 响应：
{
  "userId": "u001",
  "nickname": "张三"
}

v2 响应（新增字段）：
{
  "userId": "u001",
  "nickname": "张三",
  "avatar": "https://...",    // 新增字段
  "vipLevel": 2               // 新增字段
}

兼容性：✓ 旧客户端忽略新字段
```

### 字段删除

```
v1 响应：
{
  "userId": "u001",
  "oldField": "xxx"    // 计划删除
}

v2 响应（标记废弃）：
{
  "userId": "u001",
  "oldField": null,    // 标记为废弃，返回 null
  "newField": "xxx"    // 替代字段
}

v3 响应（删除字段）：
{
  "userId": "u001",
  "newField": "xxx"    // oldField 已删除
}

兼容性：需要版本升级
```

### 字段类型变更

```
v1 响应：
{
  "id": 123           // 数字类型
}

v2 响应（类型变更）：
{
  "id": "123"         // 字符串类型
}

兼容性：✗ 需要版本升级
```

## 设计检查清单

### 入参检查

- [ ] 必填字段最小化
- [ ] 参数命名语义清晰
- [ ] 使用明确类型（避免 any）
- [ ] 提供合理的默认值
- [ ] 验证规则完整

### 出参检查

- [ ] 结构扁平（≤3 层）
- [ ] 信息完整（避免额外查询）
- [ ] 相同概念结构一致
- [ ] 空值处理明确
- [ ] 字段命名规范

### 兼容性检查

- [ ] 新增字段向后兼容
- [ ] 删除字段有过渡期
- [ ] 类型变更有版本升级
- [ ] 文档已更新
