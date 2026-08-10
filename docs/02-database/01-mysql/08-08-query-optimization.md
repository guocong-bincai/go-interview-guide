# MySQL 查询优化与慢查询分析

> 考察频率：★★★★☆  难度：★★★☆☆
> 重点：能用 EXPLAIN 分析执行计划，了解索引失效场景，知道常见的优化手段

## 1. 慢查询日志

### 开启慢查询

```sql
-- 查看慢查询是否开启
SHOW VARIABLES LIKE 'slow_query_log';

-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录

-- 开启慢查询记录所有未使用索引的查询
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

```properties
# my.cnf 配置
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
```

### pt-query-digest 分析

```bash
# 安装 percona-toolkit
# 分析慢查询日志
pt-query-digest /var/log/mysql/slow.log

# 输出示例
# Query 1: 0.00 QPS, 0.05x concat 0.00/0.00/0.00 %
#   95% of answered below 8ms
#   "SELECT * FROM orders WHERE user_id = ?"
```

---

## 2. EXPLAIN 执行计划

### 基本用法

```sql
EXPLAIN SELECT * FROM users WHERE id = 1;

-- 5.6+ 支持查看执行计划（不执行）
EXPLAIN FORMAT=JSON SELECT ...

-- 5.7+ 支持 EXPLAIN 查看子查询
EXPLAIN FOR CONNECTION connection_id;
```

### 字段解读

| 字段 | 说明 | 关注点 |
|------|------|--------|
| **id** | SELECT 序号 | id 越大越先执行 |
| **select_type** | 查询类型 | SIMPLE/PRIMARY/SUBQUERY/DERIVED/UNION |
| **table** | 表名 | 显示哪张表 |
| **type** | 访问类型 | const > eq_ref > ref > range > index > ALL |
| **possible_keys** | 可用索引 | 有哪些索引可用 |
| **key** | 实际使用 | 实际用了哪个索引 |
| **key_len** | 索引长度 | 越短越好 |
| **ref** | 索引比较 | const/func/字段 |
| **rows** | 扫描行数 | 越少越好 |
| **Extra** | 额外信息 | Using filesort/Using temporary |

### type 详解

```
性能从好到差：
const    — 主键/唯一索引等值查询，最多一行
eq_ref   — JOIN 中，被驱动表的索引是主键或唯一索引
ref      — 索引等值查询，返回匹配的行
range    — 索引范围查询（>, <, BETWEEN, IN, LIKE）
index    — 全索引扫描，比 ALL 快（覆盖索引）
ALL      — 全表扫描，最差
```

### Extra 详解

| 值 | 含义 | 优化方向 |
|---|------|---------|
| Using index | 覆盖索引，无需回表 | 好 |
| Using where | 需要在存储引擎后过滤 | 检查是否索引覆盖 |
| Using filesort | 额外排序，无法用索引 | 必须优化 |
| Using temporary | 用了临时表 | 必须优化 |
| Using index condition | 索引下推 | 好 |
| Using join buffer | NLJ 被转成 Block NLJ | 考虑加索引 |

---

## 3. 索引失效场景

### 不走索引的典型场景

```sql
-- 1. 函数/运算
SELECT * FROM orders WHERE YEAR(create_time) = 2024;  -- 不走索引
SELECT * FROM orders WHERE id + 1 = 10;               -- 不走索引

-- 优化：改成范围查询
SELECT * FROM orders WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';

-- 2. 类型转换（字段类型不一致）
-- 如果 user_id 是 VARCHAR
SELECT * FROM orders WHERE user_id = 123;   -- 不走索引（隐式转成数字）
SELECT * FROM orders WHERE user_id = '123';  -- 走索引

-- 3. LIKE 开头是通配符
SELECT * FROM users WHERE name LIKE '%张%';  -- 不走索引
SELECT * FROM users WHERE name LIKE '张%';   -- 走索引（后缀匹配）

-- 4. OR 有一边不带索引
SELECT * FROM users WHERE id = 1 OR phone = '13800000000';  -- id有索引，phone没有，OR导致全表
-- 优化：拆成 UNION
SELECT * FROM users WHERE id = 1
UNION ALL
SELECT * FROM users WHERE phone = '13800000000' AND id IS NULL;  -- id IS NULL利用索引

-- 5. NOT IN / NOT EXISTS / <>
SELECT * FROM users WHERE id NOT IN (1, 2, 3);  -- 不走索引
SELECT * FROM users WHERE id <> 1;               -- 不走索引
-- 优化：用 > 和 < 组合，或放到子查询

-- 6. IS NULL / IS NOT NULL
SELECT * FROM users WHERE name IS NULL;  -- 可以走索引，但不推荐（MySQL倾向于全表）
-- 优化：给 NULL 设默认值（如 'UNKNOWN'），改成 = 判断

-- 7. 复合索引不遵循最左前缀
-- 索引 idx(a, b, c)
SELECT * FROM users WHERE b = 1;   -- 不走 a，不符合最左前缀
SELECT * FROM users WHERE a = 1 AND c = 1;  -- 只用上 a，c 用不到
```

### 索引生效的条件

```sql
-- 复合索引最左前缀原则
ALTER TABLE users ADD INDEX idx_name_age_dept(name, age, dept);

-- 这些SQL能用上索引
SELECT * FROM users WHERE name = '张三';                 -- ✅ 用上 name
SELECT * FROM users WHERE name = '张三' AND age = 30;  -- ✅ 用上 name, age
SELECT * FROM users WHERE name = '张三' AND age = 30 AND dept = '技术';  -- ✅ 全用

-- 这些SQL用不上索引
SELECT * FROM users WHERE age = 30;                     -- ❌ 没有 name
SELECT * FROM users WHERE dept = '技术';               -- ❌ 没有 name
```

---

## 4. 常见优化手段

### 4.1 避免 SELECT *

```sql
-- 慢：传输大量不需要的字段
SELECT * FROM orders WHERE user_id = 1;

-- 快：只查需要的字段
SELECT id, status, amount FROM orders WHERE user_id = 1;
```

### 4.2 大表分页优化

```sql
-- 慢：OFFSET 大时，MySQL 先扫描到第10000行再返回
SELECT * FROM orders LIMIT 10000, 10;

-- 优化1：游标分页（上一页最后ID）
SELECT * FROM orders WHERE id > 10000 ORDER BY id LIMIT 10;

-- 优化2：延迟关联
SELECT * FROM orders o
INNER JOIN (
    SELECT id FROM orders WHERE user_id = 1 ORDER BY id LIMIT 10000, 10
) t ON o.id = t.id;

-- 优化3：覆盖索引 + 游标
SELECT * FROM orders WHERE id IN (
    SELECT id FROM orders WHERE user_id = 1 ORDER BY id LIMIT 10000, 10
);
```

### 4.3 JOIN 优化

```sql
-- 小表驱动大表（MySQL 自动优化，但显式写清楚更好）
SELECT * FROM orders o INNER JOIN users u ON o.user_id = u.id;
-- MySQL 会自动用小表（orders < users）作为驱动表

-- 优化：确保 JOIN 字段有索引
-- orders.user_id 和 users.id 都应该有索引

-- 多表 JOIN：数据量控制在合理范围
-- 避免 5 个以上表 JOIN
```

### 4.4 COUNT(*) 优化

```sql
-- 慢：COUNT(*) 需要全表扫描
SELECT COUNT(*) FROM orders WHERE status = 1;

-- 优化1：用统计表代替实时 COUNT
-- 订单状态变更时同步更新统计表 order_stats

-- 优化2：用 EXPLAIN 估算（不精确但快）
EXPLAIN SELECT * FROM orders;

-- 优化3：用 Covering Index
ALTER TABLE orders ADD INDEX idx_status(status);
SELECT COUNT(*) FROM orders WHERE status = 1;  -- 直接读索引，不回表

-- 优化4：分区表
SELECT COUNT(*) FROM orders PARTITION (p2024) WHERE status = 1;
```

### 4.5 批量插入优化

```sql
-- 慢：逐条插入
INSERT INTO orders (id, user_id, amount) VALUES (1, 1, 100);
INSERT INTO orders (id, user_id, amount) VALUES (2, 2, 200);

-- 快：批量插入
INSERT INTO orders (id, user_id, amount) VALUES
    (1, 1, 100),
    (2, 2, 200),
    (3, 3, 300);

-- 更快：事务包裹
START TRANSACTION;
INSERT INTO orders (id, user_id, amount) VALUES ...;
INSERT INTO orders (id, user_id, amount) VALUES ...;
COMMIT;

-- 极限优化：LOAD DATA INFILE（快10倍）
LOAD DATA INFILE '/tmp/orders.csv'
INTO TABLE orders FIELDS TERMINATED BY ',';
```

---

## 5. Go + MySQL 优化实践

### 使用PrepareStatement

```go
import (
    "database/sql"
)

// PrepareStatement 可以复用执行计划，减少解析开销
stmt, err := db.Prepare("SELECT * FROM users WHERE id = ?")
if err != nil {
    panic(err)
}
defer stmt.Close()

for i := 1; i <= 100; i++ {
    row := stmt.QueryRow(i)
    var name string
    row.Scan(&name)
    // ...
}
```

### 连接池配置

```go
import (
    _ "github.com/go-sql-driver/mysql"
)

db, err := sql.Open("mysql", "user:pass@tcp(host:3306)/dbname?parseTime=true")

// 关键配置
db.SetMaxOpenConns(50)              // 最大并发连接数
db.SetMaxIdleConns(10)              // 空闲连接数
db.SetConnMaxLifetime(time.Hour)    // 连接生命周期
db.SetConnMaxIdleTime(10 * time.Minute)  // 空闲超时

// 监控连接池状态
stats := db.Stats()
fmt.Printf("OpenConnections: %d, InUse: %d, Idle: %d\n",
    stats.OpenConnections, stats.InUse, stats.Idle)
```

### 避免 N+1 查询

```go
// 慢：N+1 查询
type Order struct {
    ID     int64
    UserID int64
    User   User  // 需要单独查询
}

func badQuery(db *sql.DB) {
    rows, _ := db.Query("SELECT id, user_id FROM orders LIMIT 100")
    for rows.Next() {
        var o Order
        rows.Scan(&o.ID, &o.UserID)
        // 每个订单再查一次用户表！
        db.QueryRow("SELECT * FROM users WHERE id = ?", o.UserID).Scan(&o.User)
    }
}

// 好：JOIN 查询
func goodQuery(db *sql.DB) []Order {
    rows, _ := db.Query(`
        SELECT o.id, o.user_id, u.name, u.email
        FROM orders o
        JOIN users u ON o.user_id = u.id
        LIMIT 100
    `)
    defer rows.Close()
    // ...
}
```

---

## 6. 常见慢查询案例

### 案例1：未加索引

```sql
-- 发现慢查询
SELECT * FROM orders WHERE user_id = 123 AND status = 1;

-- EXPLAIN 分析
EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 1;
-- type: ALL (全表扫描)
-- key: NULL (没用索引)

-- 加索引
ALTER TABLE orders ADD INDEX idx_user_status(user_id, status);
```

### 案例2：索引有但MySQL不选

```sql
-- 有索引但不走
SELECT * FROM orders WHERE user_id = 123 AND DATE(create_time) = '2024-01-01';

-- 原因：DATE() 函数导致索引失效

-- 优化：改成范围查询
SELECT * FROM orders
WHERE user_id = 123
  AND create_time >= '2024-01-01 00:00:00'
  AND create_time < '2024-01-02 00:00:00';
```

### 案例3：模糊匹配

```sql
-- 前缀通配符不走索引
SELECT * FROM users WHERE name LIKE '%张三%';

-- 优化1：ES 全文搜索
-- 优化2：分词 + 前缀索引
ALTER TABLE users ADD COLUMN name_pinyin VARCHAR(100);
UPDATE users SET name_pinyin = PINYIN(name);

-- 优化3：只保留后缀通配符
SELECT * FROM users WHERE name LIKE '张三%';  -- 走索引
```

---

## 面试话术

**Q：MySQL 执行计划 type 是 ALL 怎么优化？**

> type=ALL 是全表扫描，先看 where 条件有没有索引，如果字段没有索引就加索引；如果有索引但 MySQL 不选，看是不是统计信息不准（ANALYZE TABLE），或者索引顺序不对（复合索引最左前缀），或者数据量太小 MySQL 认为全表扫描更快。

**Q：索引失效怎么排查？**

> 先用 EXPLAIN 看执行计划，关注 Extra 字段有没有 Using filesort/Using temporary，然后检查：字段有没有经过函数运算、类型是否一致、OR 两边是否都有索引、LIKE 是否前缀通配符、是否 NOT IN。对于 OR 失效，改成 UNION；对于函数失效，改成范围查询。

**Q：分页很深怎么优化？**

> 禁止跳页查询，用游标分页代替 OFFSET 分页。比如 `WHERE id > last_id ORDER BY id LIMIT N`，这样无论翻到第几页，查询复杂度都是 O(N)，而不是 O(OFFSET+N)。或者用延迟关联，先查索引列再 JOIN 回原表。
