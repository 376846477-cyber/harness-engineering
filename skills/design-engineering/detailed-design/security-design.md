# 安全设计

设计系统的安全机制，保护数据和系统安全。

## 鉴权机制

### 认证方式

| 方式 | 适用场景 | 优缺点 |
|------|----------|--------|
| Session + Cookie | 传统Web应用 | 简单，但不适合分布式 |
| JWT | 前后端分离、微服务 | 无状态，但无法主动失效 |
| OAuth 2.0 | 第三方登录 | 标准化，但实现复杂 |
| API Key | 服务间调用 | 简单，但安全性较低 |

### JWT 认证设计

```
认证流程：

1. 用户登录
   POST /auth/login
   { username, password }
   ↓
2. 验证凭据
   ↓
3. 生成 JWT
   Header: { alg: "RS256", typ: "JWT" }
   Payload: { userId, roles, exp, iat }
   Signature: RSASHA256(header + payload, privateKey)
   ↓
4. 返回 Token
   { accessToken, refreshToken, expiresIn }
   ↓
5. 后续请求携带 Token
   Authorization: Bearer <accessToken>
   ↓
6. 验证 Token
   - 签名验证
   - 过期时间验证
   - 黑名单检查
   ↓
7. 提取用户信息
   { userId, roles }
```

### Token 设计

```typescript
interface JwtPayload {
  userId: string;           // 用户ID
  username: string;         // 用户名
  roles: string[];          // 角色列表
  permissions: string[];    // 权限列表
  iat: number;              // 签发时间
  exp: number;              // 过期时间
  jti: string;              // Token ID（用于黑名单）
}

interface TokenResponse {
  accessToken: string;      // 访问令牌（短期，如 15 分钟）
  refreshToken: string;     // 刷新令牌（长期，如 7 天）
  expiresIn: number;        // 过期时间（秒）
  tokenType: string;        // 令牌类型（Bearer）
}
```

### 刷新机制

```
Token 刷新流程：

1. Access Token 过期
   ↓
2. 使用 Refresh Token 刷新
   POST /auth/refresh
   { refreshToken }
   ↓
3. 验证 Refresh Token
   - 签名验证
   - 过期时间验证
   - 是否在黑名单
   ↓
4. 生成新的 Access Token
   ↓
5. 返回新的 Token
   { accessToken, refreshToken, expiresIn }

注意：
- Refresh Token 只能使用一次
- 使用后旧的 Refresh Token 加入黑名单
- 支持并发刷新处理
```

## 接口权限

### RBAC 模型

```
RBAC（基于角色的访问控制）：

用户 ──关联──→ 角色 ──关联──→ 权限

示例：
用户: 张三
角色: 订单管理员
权限: 
  - order:create
  - order:read
  - order:update
  - order:export
```

### 权限模型设计

```sql
-- 用户表
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  username VARCHAR(50) NOT NULL,
  password VARCHAR(255) NOT NULL,
  ...
);

-- 角色表
CREATE TABLE roles (
  id BIGINT PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  code VARCHAR(50) NOT NULL UNIQUE,
  description VARCHAR(255),
  ...
);

-- 权限表
CREATE TABLE permissions (
  id BIGINT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  code VARCHAR(100) NOT NULL UNIQUE,
  resource VARCHAR(100) NOT NULL,
  action VARCHAR(50) NOT NULL,
  ...
);

-- 用户角色关联表
CREATE TABLE user_roles (
  user_id BIGINT NOT NULL,
  role_id BIGINT NOT NULL,
  PRIMARY KEY (user_id, role_id)
);

-- 角色权限关联表
CREATE TABLE role_permissions (
  role_id BIGINT NOT NULL,
  permission_id BIGINT NOT NULL,
  PRIMARY KEY (role_id, permission_id)
);
```

### 权限检查

```typescript
// 权限注解
@RequirePermission('order:create')
async createOrder(request: CreateOrderRequest) {
  // ...
}

// 权限检查中间件
async function checkPermission(userId: string, permission: string): Promise<boolean> {
  const userPermissions = await getUserPermissions(userId);
  return userPermissions.includes(permission);
}

// 权限检查逻辑
async function getUserPermissions(userId: string): Promise<string[]> {
  const roles = await getUserRoles(userId);
  const permissions = await getRolePermissions(roles.map(r => r.id));
  return permissions.map(p => p.code);
}
```

### 权限命名规范

```
权限格式：resource:action

资源：
- user: 用户
- order: 订单
- product: 商品
- report: 报表

操作：
- create: 创建
- read: 读取
- update: 更新
- delete: 删除
- export: 导出
- import: 导入

示例：
- user:create: 创建用户
- order:read: 查看订单
- order:export: 导出订单
- report:view: 查看报表
```

## 数据权限

### 数据权限模型

```
数据权限层级：

1. 全部数据
   - 管理员可以访问所有数据

2. 部门数据
   - 只能访问本部门数据

3. 个人数据
   - 只能访问自己的数据

4. 自定义规则
   - 根据业务规则过滤
```

### 数据权限实现

```typescript
interface DataPermission {
  type: 'all' | 'department' | 'self' | 'custom';
  departmentIds?: string[];  // 部门ID列表
  customRule?: string;       // 自定义规则
}

// 数据权限过滤
async function applyDataPermission(
  query: SelectQuery,
  userId: string,
  resource: string
): Promise<SelectQuery> {
  const permission = await getDataPermission(userId, resource);
  
  switch (permission.type) {
    case 'all':
      return query;
      
    case 'department':
      const deptIds = await getUserDepartmentIds(userId);
      return query.where('department_id', 'IN', deptIds);
      
    case 'self':
      return query.where('user_id', '=', userId);
      
    case 'custom':
      return applyCustomRule(query, permission.customRule);
  }
}
```

### 数据权限 SQL 示例

```sql
-- 全部数据
SELECT * FROM orders;

-- 部门数据
SELECT * FROM orders 
WHERE department_id IN (SELECT department_id FROM user_departments WHERE user_id = ?);

-- 个人数据
SELECT * FROM orders 
WHERE user_id = ?;

-- 自定义规则（如：订单金额小于10000或状态为已完成）
SELECT * FROM orders 
WHERE (total_amount < 10000 OR status = 'completed')
AND user_id = ?;
```

## 敏感数据处理

### 数据分类

| 级别 | 类型 | 示例 | 处理方式 |
|------|------|------|----------|
| 公开 | 可公开数据 | 商品名称 | 无需特殊处理 |
| 内部 | 内部使用数据 | 订单号 | 访问控制 |
| 敏感 | 个人隐私数据 | 手机号、邮箱 | 脱敏、加密 |
| 机密 | 高敏感数据 | 密码、身份证号 | 加密存储、脱敏展示 |

### 数据脱敏

```typescript
// 脱敏规则
const maskingRules = {
  phone: (value: string) => value.replace(/(\d{3})\d{4}(\d{4})/, '$1****$2'),
  email: (value: string) => value.replace(/(.{2}).*(@.*)/, '$1***$2'),
  idCard: (value: string) => value.replace(/(.{6}).*(.{4})/, '$1********$2'),
  bankCard: (value: string) => value.replace(/(.{4}).*(.{4})/, '$1****$2'),
  name: (value: string) => value.charAt(0) + '*'.repeat(value.length - 1),
};

// 脱敏示例
maskingRules.phone('13812345678');    // 138****5678
maskingRules.email('test@example.com'); // te***@example.com
maskingRules.idCard('110101199001011234'); // 110101********1234
```

### 数据加密

```typescript
// AES 加密
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const algorithm = 'aes-256-gcm';
const key = Buffer.from(process.env.ENCRYPTION_KEY, 'hex');

function encrypt(plaintext: string): { ciphertext: string; iv: string; tag: string } {
  const iv = randomBytes(12);
  const cipher = createCipheriv(algorithm, key, iv);
  let encrypted = cipher.update(plaintext, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const tag = cipher.getAuthTag();
  
  return {
    ciphertext: encrypted,
    iv: iv.toString('hex'),
    tag: tag.toString('hex'),
  };
}

function decrypt(encrypted: { ciphertext: string; iv: string; tag: string }): string {
  const decipher = createDecipheriv(algorithm, key, Buffer.from(encrypted.iv, 'hex'));
  decipher.setAuthTag(Buffer.from(encrypted.tag, 'hex'));
  let decrypted = decipher.update(encrypted.ciphertext, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

### 密码存储

```typescript
import { hash, compare } from 'bcrypt';

// 密码哈希
async function hashPassword(password: string): Promise<string> {
  const saltRounds = 12;
  return hash(password, saltRounds);
}

// 密码验证
async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return compare(password, hash);
}
```

### 敏感操作审计

```typescript
interface AuditLog {
  id: string;
  userId: string;
  action: string;           // 操作类型
  resource: string;         // 资源类型
  resourceId: string;       // 资源ID
  oldValue?: string;        // 原值（脱敏）
  newValue?: string;        // 新值（脱敏）
  ipAddress: string;        // IP地址
  userAgent: string;        // 用户代理
  timestamp: Date;          // 操作时间
}

// 记录审计日志
async function logAudit(auditLog: AuditLog) {
  await saveAuditLog({
    ...auditLog,
    oldValue: auditLog.oldValue ? maskSensitiveData(auditLog.oldValue) : undefined,
    newValue: auditLog.newValue ? maskSensitiveData(auditLog.newValue) : undefined,
  });
}
```

## 安全检查清单

### 认证安全

- [ ] 使用 HTTPS
- [ ] 密码加密存储（bcrypt）
- [ ] Token 有过期时间
- [ ] 支持 Token 刷新
- [ ] 支持 Token 黑名单
- [ ] 登录失败限制

### 授权安全

- [ ] 权限最小化原则
- [ ] 默认拒绝访问
- [ ] 权限检查在服务端
- [ ] 敏感操作二次确认

### 数据安全

- [ ] 敏感数据加密存储
- [ ] 敏感数据脱敏展示
- [ ] 数据传输加密
- [ ] 敏感操作审计日志

### 接口安全

- [ ] 参数验证
- [ ] SQL 注入防护
- [ ] XSS 防护
- [ ] CSRF 防护
- [ ] 接口限流