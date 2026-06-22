# RESTful API 设计规范

设计符合 REST 风格的 HTTP API。

## RESTful 核心原则

### 1. 资源导向

以资源为中心设计 API，而不是以操作为中心。

```
不好的设计（面向操作）：
POST /createUser
POST /deleteUser
POST /updateUser
GET /getUser

好的设计（面向资源）：
POST   /users        # 创建用户
DELETE /users/{id}   # 删除用户
PUT    /users/{id}   # 更新用户
GET    /users/{id}   # 获取用户
```

### 2. 统一接口

使用标准的 HTTP 方法和状态码。

### 3. 无状态

每个请求包含所有必要信息，不依赖服务器会话状态。

### 4. 分层系统

客户端不需要知道是连接到最终服务器还是中间层。

## URL 设计规范

### 资源命名

| 规则 | 示例 | 说明 |
|------|------|------|
| 使用名词 | `/users`, `/orders` | 而非 `/getUsers` |
| 使用复数 | `/users` 而非 `/user` | 保持一致性 |
| 使用小写 | `/order-items` | 而非 `/OrderItems` |
| 使用连字符 | `/order-items` | 而非 `/orderItems` |
| 避免动词 | `/users` | 而非 `/getUsers` |

### URL 层级

```
资源层级表达：

/users                           # 用户集合
/users/{userId}                  # 单个用户
/users/{userId}/orders           # 用户的订单集合
/users/{userId}/orders/{orderId} # 用户的单个订单

规则：
• 层级不超过 3 层
• 使用 ID 定位具体资源
• 避免过深的嵌套
```

### 查询参数

```
过滤：
GET /users?status=active
GET /orders?status=pending&userId=u001

分页：
GET /users?page=1&pageSize=20
GET /users?offset=0&limit=20

排序：
GET /users?sort=createdAt:desc
GET /users?sort=name:asc,createdAt:desc

字段选择：
GET /users?fields=id,name,email

搜索：
GET /users?q=张三
```

## HTTP 方法语义

### 方法定义

| 方法 | 语义 | 幂等性 | 安全性 | 请求体 | 响应体 |
|------|------|--------|--------|--------|--------|
| GET | 获取资源 | 是 | 是 | 否 | 是 |
| POST | 创建资源 | 否 | 否 | 是 | 是 |
| PUT | 全量更新 | 是 | 否 | 是 | 是/否 |
| PATCH | 部分更新 | 否 | 否 | 是 | 是/否 |
| DELETE | 删除资源 | 是 | 否 | 否 | 是/否 |

### 使用示例

```
获取用户列表：
GET /users
Response: { "users": [...], "total": 100 }

获取单个用户：
GET /users/u001
Response: { "id": "u001", "name": "张三" }

创建用户：
POST /users
Request: { "name": "张三", "email": "zhangsan@example.com" }
Response: { "id": "u001", "name": "张三" }

全量更新用户：
PUT /users/u001
Request: { "name": "李四", "email": "lisi@example.com" }
Response: { "id": "u001", "name": "李四" }

部分更新用户：
PATCH /users/u001
Request: { "name": "李四" }
Response: { "id": "u001", "name": "李四" }

删除用户：
DELETE /users/u001
Response: 204 No Content
```

## HTTP 状态码

### 成功响应

| 状态码 | 含义 | 使用场景 |
|--------|------|----------|
| 200 OK | 成功 | GET/PUT/PATCH/DELETE 成功 |
| 201 Created | 已创建 | POST 创建资源成功 |
| 202 Accepted | 已接受 | 异步任务已接受 |
| 204 No Content | 无内容 | DELETE 成功或 PUT 不返回内容 |

### 客户端错误

| 状态码 | 含义 | 使用场景 |
|--------|------|----------|
| 400 Bad Request | 参数错误 | 请求参数不合法 |
| 401 Unauthorized | 未认证 | 缺少认证信息 |
| 403 Forbidden | 禁止访问 | 无权限访问 |
| 404 Not Found | 未找到 | 资源不存在 |
| 405 Method Not Allowed | 方法不允许 | 不支持的 HTTP 方法 |
| 409 Conflict | 冲突 | 资源冲突（如重复创建） |
| 422 Unprocessable Entity | 无法处理 | 语义错误 |
| 429 Too Many Requests | 限流 | 请求过于频繁 |

### 服务端错误

| 状态码 | 含义 | 使用场景 |
|--------|------|----------|
| 500 Internal Server Error | 服务器错误 | 系统内部错误 |
| 502 Bad Gateway | 网关错误 | 上游服务错误 |
| 503 Service Unavailable | 服务不可用 | 服务维护或过载 |
| 504 Gateway Timeout | 网关超时 | 上游服务超时 |

## 请求头与响应头

### 常用请求头

| 请求头 | 说明 | 示例 |
|--------|------|------|
| Authorization | 认证信息 | Bearer xxx |
| Content-Type | 内容类型 | application/json |
| Accept | 接受类型 | application/json |
| Accept-Language | 语言偏好 | zh-CN |
| If-Modified-Since | 缓存检查 | Wed, 21 Oct 2024 07:28:00 GMT |
| If-None-Match | 缓存检查 | "etag-value" |

### 常用响应头

| 响应头 | 说明 | 示例 |
|--------|------|------|
| Content-Type | 内容类型 | application/json |
| Content-Length | 内容长度 | 1234 |
| Cache-Control | 缓存控制 | max-age=3600 |
| ETag | 资源版本 | "etag-value" |
| Last-Modified | 最后修改时间 | Wed, 21 Oct 2024 07:28:00 GMT |
| X-Request-Id | 请求追踪ID | abc123 |
| X-RateLimit-Limit | 限流上限 | 1000 |
| X-RateLimit-Remaining | 剩余配额 | 999 |

## 版本管理

### 版本策略

**URL 版本（推荐）**

```
/api/v1/users
/api/v2/users
```

**Header 版本**

```
Accept: application/vnd.myapi.v1+json
Accept: application/vnd.myapi.v2+json
```

**查询参数版本**

```
/api/users?version=1
/api/users?version=2
```

### 版本兼容规则

| 变更类型 | 兼容性 | 是否需要升级大版本 |
|----------|--------|-------------------|
| 新增接口 | 向后兼容 | 否 |
| 新增可选字段 | 向后兼容 | 否 |
| 新增响应字段 | 向后兼容 | 否 |
| 删除接口 | 不兼容 | 是 |
| 删除字段 | 不兼容 | 是 |
| 修改字段类型 | 不兼容 | 是 |
| 修改字段语义 | 不兼容 | 是 |

## 分页设计

### 分页参数

```
方式1：页码分页
GET /users?page=1&pageSize=20

方式2：偏移分页
GET /users?offset=0&limit=20

方式3：游标分页（大数据量推荐）
GET /users?cursor=abc123&limit=20
```

### 分页响应

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "list": [...],
    "pagination": {
      "total": 100,
      "page": 1,
      "pageSize": 20,
      "totalPages": 5,
      "hasMore": true
    }
  }
}
```

## HATEOAS

超媒体作为应用状态引擎，返回可执行的操作链接。

```json
{
  "id": "order001",
  "status": "pending",
  "_links": {
    "self": { "href": "/api/v1/orders/order001" },
    "pay": { "href": "/api/v1/orders/order001/pay", "method": "POST" },
    "cancel": { "href": "/api/v1/orders/order001/cancel", "method": "POST" }
  }
}
```

## 检查清单

- [ ] URL 使用名词复数
- [ ] HTTP 方法语义正确
- [ ] 状态码使用恰当
- [ ] 请求/响应头规范
- [ ] 版本管理策略明确
- [ ] 分页设计合理
- [ ] 错误处理统一
