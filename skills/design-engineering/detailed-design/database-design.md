# 数据库设计

设计数据存储结构，确保数据的完整性、一致性和性能。

## 表结构设计

### 设计原则

| 原则 | 描述 | 示例 |
|------|------|------|
| 规范化 | 避免数据冗余 | 第三范式 |
| 适度反规范化 | 查询性能优化 | 冗余常用字段 |
| 命名规范 | 统一命名风格 | snake_case |
| 主键设计 | 合理选择主键 | 自增ID或UUID |
| 软删除 | 保留删除记录 | deleted_at 字段 |

### 表结构模板

```sql
CREATE TABLE `orders` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `order_no` VARCHAR(32) NOT NULL COMMENT '订单号',
  `user_id` BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
  `status` TINYINT NOT NULL DEFAULT 0 COMMENT '订单状态：0-待支付，1-已支付，2-已发货，3-已完成，4-已取消',
  `total_amount` DECIMAL(12,2) NOT NULL DEFAULT 0.00 COMMENT '订单总金额',
  `pay_amount` DECIMAL(12,2) NOT NULL DEFAULT 0.00 COMMENT '实付金额',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '订单备注',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_at` DATETIME DEFAULT NULL COMMENT '删除时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_no` (`order_no`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_status` (`status`),
  KEY `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单表';
```

### 字段类型选择

| 数据类型 | 适用场景 | 示例 |
|----------|----------|------|
| BIGINT | 主键、外键、大数值 | id, user_id |
| INT | 数量、计数 | quantity, count |
| DECIMAL(M,N) | 金额、精确小数 | amount, price |
| VARCHAR(N) | 变长字符串 | name, description |
| TEXT | 长文本 | content, remark |
| TINYINT | 状态、类型枚举 | status, type |
| DATETIME | 日期时间 | created_at |
| TIMESTAMP | 时间戳（自动更新） | updated_at |
| JSON | JSON数据 | extra, metadata |

### 字段命名规范

| 类型 | 命名规范 | 示例 |
|------|----------|------|
| 主键 | id | id |
| 外键 | xxx_id | user_id, order_id |
| 状态 | xxx_status | order_status |
| 类型 | xxx_type | user_type |
| 时间 | xxx_at | created_at, paid_at |
| 金额 | xxx_amount | total_amount |
| 数量 | xxx_count/quantity | item_count |
| 标志 | is_xxx/has_xxx | is_deleted, has_paid |

## ER图设计

### ER图符号

```
┌─────────────┐
│   实体       │  矩形：实体（表）
├─────────────┤
│   属性       │  椭圆：属性（字段）
└─────────────┘

◇  菱形：关系

连线：
  ───  一对一
  1   一
  N   多
  M   多对多
```

### ER图示例

```
                    ┌─────────────┐
                    │    用户      │
                    ├─────────────┤
                    │ id (PK)     │
                    │ name        │
                    │ email       │
                    │ phone       │
                    └──────┬──────┘
                           │
                           │ 1
                           │
                           ◇ 下单
                           │
                           │ N
                           │
                    ┌──────┴──────┐
                    │    订单      │
                    ├─────────────┤
                    │ id (PK)     │
                    │ user_id(FK) │
                    │ order_no    │
                    │ status      │
                    │ total_amount│
                    └──────┬──────┘
                           │
                           │ 1
                           │
                           ◇ 包含
                           │
                           │ N
                           │
                    ┌──────┴──────┐
                    │   订单项     │
                    ├─────────────┤
                    │ id (PK)     │
                    │ order_id(FK)│
                    │ product_id  │
                    │ quantity    │
                    │ price       │
                    └─────────────┘
```

### 关系类型

| 关系类型 | 说明 | 实现方式 |
|----------|------|----------|
| 一对一 (1:1) | 一个实体对应一个实体 | 外键 + UNIQUE 约束 |
| 一对多 (1:N) | 一个实体对应多个实体 | 外键 |
| 多对多 (M:N) | 多个实体对应多个实体 | 中间表 |

## 索引设计

### 索引类型

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| 主键索引 | 唯一标识记录 | id |
| 唯一索引 | 字段值唯一 | order_no, email |
| 普通索引 | 加速查询 | status, created_at |
| 组合索引 | 多字段组合查询 | (user_id, status) |
| 全文索引 | 全文搜索 | content, description |

### 索引设计原则

```
索引设计原则：

1. 选择性高的字段优先
   选择性 = 不同值数量 / 总记录数
   选择性越高，索引效果越好

2. 遵循最左前缀原则
   组合索引 (a, b, c)
   支持：a, (a,b), (a,b,c)
   不支持：b, c, (b,c)

3. 覆盖索引优化
   查询字段都在索引中，无需回表

4. 避免过度索引
   索引占用存储，影响写入性能
```

### 索引示例

```sql
-- 唯一索引
CREATE UNIQUE INDEX uk_order_no ON orders(order_no);

-- 普通索引
CREATE INDEX idx_user_id ON orders(user_id);

-- 组合索引
CREATE INDEX idx_user_status ON orders(user_id, status);

-- 覆盖索引
CREATE INDEX idx_user_status_amount ON orders(user_id, status, total_amount);

-- 条件索引（部分索引）
CREATE INDEX idx_active_orders ON orders(created_at) 
WHERE status NOT IN (4);  -- 排除已取消订单
```

### 索引分析

```sql
-- 查看执行计划
EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND status = 1;

-- 分析索引使用情况
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1;

-- 查看表索引
SHOW INDEX FROM orders;

-- 分析索引选择性
SELECT 
  COUNT(DISTINCT user_id) / COUNT(*) as user_id_selectivity,
  COUNT(DISTINCT status) / COUNT(*) as status_selectivity
FROM orders;
```

## 关联关系设计

### 外键约束

```sql
-- 创建外键
ALTER TABLE orders
ADD CONSTRAINT fk_orders_user_id
FOREIGN KEY (user_id) REFERENCES users(id)
ON DELETE RESTRICT    -- 限制删除
ON UPDATE CASCADE;    -- 级联更新

-- 外键选项
-- RESTRICT: 限制（默认）
-- CASCADE: 级联
-- SET NULL: 设为NULL
-- NO ACTION: 无操作
```

### 中间表设计

```sql
-- 多对多关系：用户-角色
CREATE TABLE `user_roles` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `user_id` BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
  `role_id` BIGINT UNSIGNED NOT NULL COMMENT '角色ID',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_role` (`user_id`, `role_id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_role_id` (`role_id`),
  CONSTRAINT `fk_user_roles_user_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`) ON DELETE CASCADE,
  CONSTRAINT `fk_user_roles_role_id` FOREIGN KEY (`role_id`) REFERENCES `roles` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户角色关联表';
```

## 数据库设计文档模板

```markdown
# 数据库设计文档

## 概述

[数据库设计概述]

## ER图

[ER图]

## 表结构

### 表1：xxx表

| 字段名 | 类型 | 是否为空 | 默认值 | 说明 |
|--------|------|----------|--------|------|
| ... | ... | ... | ... | ... |

**索引：**
- PRIMARY KEY (id)
- UNIQUE KEY uk_xxx (xxx)
- KEY idx_xxx (xxx)

## 索引设计

| 索引名 | 类型 | 字段 | 说明 |
|--------|------|------|------|
| ... | ... | ... | ... |

## 数据字典

| 表名 | 说明 |
|------|------|
| ... | ... |
```

## 检查清单

- [ ] 表结构设计完整
- [ ] 字段类型选择合理
- [ ] 主键设计合理
- [ ] 外键关系正确
- [ ] 索引设计合理
- [ ] 命名规范统一
- [ ] 文档完整