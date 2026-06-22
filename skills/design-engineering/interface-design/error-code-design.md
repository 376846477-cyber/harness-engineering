# 错误码设计

设计统一的错误码体系，确保错误信息清晰、可追踪。

## 错误码结构

### 编码规则

```
错误码格式：[服务码][模块码][错误类型][错误序号]

示例：
  10001 = 用户服务(1) + 用户模块(0) + 参数错误(0) + 序号(01)
  20101 = 订单服务(2) + 订单模块(0) + 业务错误(1) + 序号(01)
```

### 错误码分类

| 范围 | 类型 | HTTP 状态码 | 说明 |
|------|------|-------------|------|
| 00000 | 成功 | 200 | 请求成功 |
| 1xxxx | 参数错误 | 400 | 请求参数不合法 |
| 2xxxx | 业务错误 | 400 | 业务规则校验失败 |
| 3xxxx | 权限错误 | 401/403 | 认证或授权失败 |
| 4xxxx | 资源错误 | 404 | 资源不存在 |
| 5xxxx | 系统错误 | 500 | 系统内部错误 |

## 错误响应结构

### 标准错误响应

```typescript
interface ErrorResponse {
  code: number;              // 业务错误码
  message: string;           // 用户友好信息
  details?: ErrorDetail[];   // 详细错误信息
  traceId: string;           // 追踪ID
  timestamp: number;         // 时间戳
}

interface ErrorDetail {
  field: string;             // 错误字段
  value: any;                // 错误值
  reason: string;            // 错误原因
}
```

### 错误响应示例

```json
{
  "code": 10101,
  "message": "参数验证失败",
  "details": [
    {
      "field": "email",
      "value": "invalid-email",
      "reason": "邮箱格式不正确"
    },
    {
      "field": "password",
      "value": "",
      "reason": "密码不能为空"
    }
  ],
  "traceId": "abc123def456",
  "timestamp": 1704067200000
}
```

## 错误码定义

### 通用错误码

| 错误码 | 错误名称 | HTTP 状态码 | 说明 |
|--------|----------|-------------|------|
| 0 | SUCCESS | 200 | 成功 |
| 10001 | INVALID_PARAMETER | 400 | 参数格式错误 |
| 10002 | MISSING_PARAMETER | 400 | 缺少必填参数 |
| 10003 | PARAMETER_OUT_OF_RANGE | 400 | 参数超出范围 |
| 30001 | UNAUTHORIZED | 401 | 未登录或令牌过期 |
| 30002 | FORBIDDEN | 403 | 无权限访问 |
| 40001 | NOT_FOUND | 404 | 资源不存在 |
| 50001 | INTERNAL_ERROR | 500 | 系统内部错误 |
| 50002 | SERVICE_UNAVAILABLE | 503 | 服务不可用 |
| 50003 | RATE_LIMIT_EXCEEDED | 429 | 请求过于频繁 |

### 业务错误码示例

**用户模块（1xxxx）**

| 错误码 | 错误名称 | 说明 |
|--------|----------|------|
| 10101 | EMAIL_FORMAT_ERROR | 邮箱格式错误 |
| 10102 | PASSWORD_TOO_WEAK | 密码强度不足 |
| 10103 | EMAIL_ALREADY_EXISTS | 邮箱已被注册 |
| 10104 | USER_NOT_FOUND | 用户不存在 |
| 10105 | PASSWORD_INCORRECT | 密码错误 |

**订单模块（2xxxx）**

| 错误码 | 错误名称 | 说明 |
|--------|----------|------|
| 20101 | OUT_OF_STOCK | 库存不足 |
| 20102 | PRODUCT_NOT_AVAILABLE | 商品已下架 |
| 20103 | ORDER_NOT_FOUND | 订单不存在 |
| 20104 | ORDER_STATUS_INVALID | 订单状态不允许此操作 |
| 20105 | COUPON_EXPIRED | 优惠券已过期 |
| 20106 | COUPON_NOT_AVAILABLE | 优惠券不可用 |

## 错误处理最佳实践

### 错误信息设计

```
错误信息层次：

1. 用户友好信息（message）
   - 面向最终用户
   - 简洁明了
   - 提供解决建议

2. 技术信息（details）
   - 面向开发者
   - 包含具体字段和原因
   - 便于调试

3. 日志信息
   - 面向运维
   - 包含完整堆栈
   - 便于排查问题
```

### 错误信息示例

```typescript
// 好的错误信息
{
  "code": 20101,
  "message": "商品库存不足，请减少购买数量或选择其他商品",
  "details": [
    {
      "field": "items[0].quantity",
      "value": 10,
      "reason": "商品「iPhone 15 Pro」库存仅剩 5 件"
    }
  ]
}

// 不好的错误信息
{
  "code": 20101,
  "message": "库存不足"  // 信息不足，用户不知道怎么办
}
```

### 错误处理策略

| 错误类型 | 处理策略 | 用户提示 |
|----------|----------|----------|
| 参数错误 | 直接返回，不记录日志 | 提示具体字段和原因 |
| 业务错误 | 记录业务日志 | 提示具体原因和建议 |
| 权限错误 | 记录安全日志 | 提示无权限，引导登录 |
| 系统错误 | 记录错误日志+告警 | 提示系统繁忙，稍后重试 |
| 第三方错误 | 记录错误日志 | 提示服务暂时不可用 |

## 错误码文档模板

```markdown
# 错误码文档

## 概述

[错误码体系说明]

## 通用错误码

| 错误码 | 错误名称 | HTTP 状态码 | 说明 | 解决方案 |
|--------|----------|-------------|------|----------|
| ... | ... | ... | ... | ... |

## 业务错误码

### [模块名称]

| 错误码 | 错误名称 | HTTP 状态码 | 说明 | 解决方案 |
|--------|----------|-------------|------|----------|
| ... | ... | ... | ... | ... |

## 错误响应示例

### 参数错误

```json
{
  "code": 10001,
  "message": "参数验证失败",
  "details": [...]
}
```

### 业务错误

```json
{
  "code": 20101,
  "message": "库存不足",
  "details": [...]
}
```
```

## 错误码管理

### 错误码分配流程

```
1. 新增错误码申请
   ├── 填写错误码申请表
   ├── 技术评审
   └── 分配错误码

2. 错误码变更
   ├── 评估影响范围
   ├── 通知相关方
   └── 更新文档

3. 错误码废弃
   ├── 标记为废弃
   ├── 保留过渡期
   └── 最终删除
```

### 错误码注册表

```typescript
// 错误码定义文件
export const ErrorCodes = {
  // 通用错误
  SUCCESS: { code: 0, message: '成功' },
  INVALID_PARAMETER: { code: 10001, message: '参数格式错误' },
  
  // 用户模块
  USER_NOT_FOUND: { code: 10104, message: '用户不存在' },
  PASSWORD_INCORRECT: { code: 10105, message: '密码错误' },
  
  // 订单模块
  OUT_OF_STOCK: { code: 20101, message: '库存不足' },
  ORDER_NOT_FOUND: { code: 20103, message: '订单不存在' },
} as const;

// 使用示例
throw new BusinessException(ErrorCodes.OUT_OF_STOCK, {
  details: [{ field: 'quantity', reason: '库存仅剩 5 件' }]
});
```

## 检查清单

- [ ] 错误码编码规则明确
- [ ] 错误码分类清晰
- [ ] 错误响应结构统一
- [ ] 错误信息用户友好
- [ ] 错误码文档完整
- [ ] 错误码注册表维护
