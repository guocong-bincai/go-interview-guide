# MySQL 慢查询优化：EXPLAIN 解读与索引失效场景

## 面试官考察意图

慢查询优化是 MySQL 面试高频考点。
初级只知道"加索引"，高级要能讲清楚 **EXPLAIN 每个字段的含义、哪些条件会导致索引失效、怎么通过慢查询日志定位问题、如何用覆盖索引避免回表**，并结合生产案例给出完整优化链路。

---

## 核心答案（30 秒版）

慢查询优化四步走：

```
① 定位：开启慢查询日志，抓出 >1s 的 SQL
② 分析：EXPLAIN 查看执行计划，判断是否走索引、是否回表
③ 优化：根据索引失效场景调整 SQL 或新建索引
④ 验证：上线后持续监控 query latency
```

---

## 深度展开

### 1. 慢查询定位

```sql
-- 查看慢查询是否开启
SHOW VARIABLES LIKE 'slow_query_log%';

-- 开启慢查询（临时）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录
SET GLOBAL slow_query_log_file = '/var/lib/mysql/slow.log';

-- 查看有多少条慢查询
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- 分析慢查询日志（pt-query-digest）
pt-query-digest /var/lib/mysql/slow.log
```

```go
// Go 中使用 go-sql-driver 配合 pprof 定位慢查询
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql"
    "net/http/pprof"
)

// 注册 pprof handler
http.Handle("/debug/pprof/", http.HandlerFunc(pprof.Handler))
```

### 2. EXPLAIN 深度解读

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 'paid' ORDER BY created_at DESC;
```

#### EXPLAIN 字段详解

| 字段 | 含义 | 判断标准 |
|------|------|----------|
| **id** | 查询中 SELECT 的顺序，id 越大越先执行 | id 相同从前往后，id 不同从大到小 |
| **select_type** | 查询类型 | SIMPLE（简单查询）、PRIMARY（最外层）、SUBQUERY（子查询）、DERIVED（衍生表） |
| **table** | 引用的表 |  |
| **type** | **访问类型**，最重要 | `const > eq_ref > ref > range > index > ALL`（ALL 需要优化） |
| **possible_keys** | 可能使用的索引 |  |
| **key** | **实际使用的索引** | NULL = 未走索引 |
| **key_len** | 索引使用的字节数 | 可判断复合索引使用了前几个字段 |
| **ref** | 与索引比较的列 | const（常量）、func、关联字段 |
| **rows** | 预估值扫描行数 | 越大越需要优化 |
| **Extra** | 额外信息 | Using filesort / Using temporary / Using index = 关键信号 |

#### type 字段详解（从好到差）

| type | 含义 | 说明 |
|------|------|------|
| `const` | 主键或唯一索引常量等值查询 | 最多 1 行，极优 |
| `eq_ref` | 关联查询时，使用主键或唯一索引 | 多表关联时极优 |
| `ref` | 普通索引等值查询 | 非唯一索引，命中多行 |
| `range` | 索引范围查询（ BETWEEN、IN、>、<） | 比全表扫描好 |
| `index` | 全索引扫描 | 比 ALL 快，但仍需优化 |
| `ALL` | **全表扫描** | 🔴 最差，必须优化 |

```sql
-- 典型问题：type=ALL，rows=百万级，需要优化
EXPLAIN SELECT * FROM orders WHERE created_at > '2026-01-01';
-- 结果：type=ALL（索引在 user_id 上，created_at 范围查走全表）

-- 优化：加 (user_id, created_at) 复合索引
ALTER TABLE orders ADD INDEX idx_user_time (user_id, created_at);
```

### 3. Extra 字段关键信号

| Extra 值 | 含义 | 处理方式 |
|----------|------|----------|
| **Using filesort** | 无法用索引排序，需额外排序 | 🔴 必须优化，加覆盖索引或调整 ORDER BY |
| **Using temporary** | 使用临时表存结果 | 🔴 常见于 GROUP BY、DISTINCT、UNION |
| **Using index** | **覆盖索引**，不回表 | ✅ 最好情况 |
| **Using index condition** | 索引下推（ICP） | ✅ 好，ICP 减少回表 |
| **Using where** | 服务层用 WHERE 过滤 | 部分数据需回表过滤 |
| **Backfiles index** | 紧凑索引扫描 | 比全索引扫描好 |

```sql
-- Using filesort 典型场景（ORDER BY 字段不在索引中）
EXPLAIN SELECT * FROM orders WHERE user_id = 100 ORDER BY amount;
-- Extra: Using filesort（user_id 有索引，但 amount 没有）

-- 优化：加 (user_id, amount) 复合索引，索引本身就是有序的
ALTER TABLE orders ADD INDEX idx_user_amount (user_id, amount);

-- 再查：Extra: Using index condition，直接从索引拿数据，无需 filesort
```

### 4. 索引失效的 10 个场景

#### 场景 1：索引列参与运算

```sql
-- ❌ 失效：YEAR() 导致索引失效
SELECT * FROM orders WHERE YEAR(created_at) = 2026;

-- ✅ 优化：范围查询，保留索引
SELECT * FROM orders WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';
```

#### 场景 2：隐式类型转换

```sql
-- ❌ user_id 是 bigint，传入字符串 '100'
SELECT * FROM orders WHERE user_id = '100';  -- 字符串转 bigint，索引失效

-- ✅ 优化：使用正确类型
SELECT * FROM orders WHERE user_id = 100;
```

#### 场景 3：LIKE 左边通配符

```sql
-- ❌ 失效：'%xx' 前缀通配符无法用 B+ 树
SELECT * FROM users WHERE name LIKE '%bincai%';

-- ✅ 优化：右通配符可以走索引（但MySQL 8.0 ICP 会有改善）
SELECT * FROM users WHERE name LIKE 'bincai%';

-- ✅ 更好的方案：使用全文索引
ALTER TABLE users ADD FULLTEXT INDEX ft_name (name);
SELECT * FROM users WHERE MATCH(name) AGAINST('bincai');
```

#### 场景 4：复合索引不遵循最左前缀

```sql
-- 复合索引 INDEX idx(a, b, c)
-- ✅ 走索引：a=1, a=1 AND b=2, a=1 AND b=2 AND c=3
-- ❌ 失效：b=2, c=3, b=2 AND c=3（跳过 a）

SELECT * FROM orders WHERE b = 2;  -- 索引失效
SELECT * FROM orders WHERE a = 1 AND c = 3;  -- 只用了 a，c 走 filesort
```

#### 场景 5：OR 连接不同列

```sql
-- ❌ 失效：OR 两边有一个不走索引就全表扫描
SELECT * FROM orders WHERE user_id = 100 OR phone = '13800001111';

-- ✅ 优化：拆成 UNION，强制走索引
SELECT * FROM orders WHERE user_id = 100
UNION ALL
SELECT * FROM orders WHERE phone = '13800001111' AND user_id IS NULL;  -- 覆盖索引技巧
```

#### 场景 6：IN 包含大量值

```sql
-- ❌ 可能失效：IN 太多值，MySQL 认为全表扫描更快
SELECT * FROM orders WHERE user_id IN (1,2,3,...,10000);

-- ✅ 优化：分段查询，或用 JOIN 改写
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE status=1);
```

#### 场景 7：NOT NULL / IS NOT NULL

```sql
-- ❌ MySQL 优化器有时不走索引
SELECT * FROM users WHERE email IS NOT NULL;

-- ✅ 优化：用默认值替代 NULL，或加复合索引
ALTER TABLE users ADD INDEX idx_email_status (email, status);
```

#### 场景 8：DISTINCT + ORDER BY

```sql
-- ❌ 可能产生 Using temporary + filesort
SELECT DISTINCT user_id FROM orders ORDER BY created_at;

-- ✅ 优化：去除不必要的 DISTINCT，或用 GROUP BY
SELECT user_id FROM orders GROUP BY user_id;
```

#### 场景 9：UNION 的陷阱

```sql
-- ❌ 第二个 SELECT 可能产生临时表
(SELECT id, name FROM users WHERE status=1)
UNION
(SELECT id, name FROM users WHERE status=2) ORDER BY name;

-- ✅ 优化：用 UNION ALL + 外部排序（延迟排序）
(SELECT id, name FROM users WHERE status=1)
UNION ALL
(SELECT id, name FROM users WHERE status=2);
-- 外部再加一层 SELECT 完成排序
```

#### 场景 10：COUNT(*) 的优化

```sql
-- ❌ 全表 COUNT(*)
SELECT COUNT(*) FROM orders WHERE status = 'paid';

-- ✅ 优化：利用索引覆盖
ALTER TABLE orders ADD INDEX idx_status (status);
SELECT COUNT(*) FROM orders WHERE status = 'paid';  -- 走覆盖索引，不回表
```

### 5. 覆盖索引（Covering Index）

覆盖索引：查询的所有字段都在索引树中，**无需回表**，性能最优。

```sql
-- 订单表
-- 索引：(user_id, status, created_at)

-- ✅ 走覆盖索引，不回表
EXPLAIN SELECT user_id, status, created_at FROM orders
WHERE user_id = 100 AND status = 'paid';

-- Extra: Using index（覆盖索引）

-- ❌ 需要回表：created_at 不在索引中
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 'paid';
```

### 6. 慢查询优化实战案例

#### 案例 1：分页深度偏移

```sql
-- ❌ 深分页：偏移量大时性能极差
SELECT * FROM orders LIMIT 1000000, 10;  -- 扫描 1000010 行

-- ✅ 优化 1：延迟关联
SELECT * FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 10
) t ON o.id = t.id;

-- ✅ 优化 2：游标分页（推荐）
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;

-- ✅ 优化 3：上一页ID缓存
-- 首次：SELECT * FROM orders ORDER BY id LIMIT 10;
-- 下一页：SELECT * FROM orders WHERE id > {last_id} LIMIT 10;
```

#### 案例 2：JOIN 顺序错误

```sql
-- ❌ 小表驱动大表错误（orders 1000万，users 10万）
SELECT * FROM orders o LEFT JOIN users u ON o.user_id = u.id WHERE u.city = '北京';

-- ✅ 优化：小表（users）在前，驱动大表（orders）
SELECT * FROM users u LEFT JOIN orders o ON o.user_id = u.id WHERE u.city = '北京';

-- EXPLAIN 检查：前表 rows 应该远小于后表
```

#### 案例 3：索引下推（ICP）优化

```sql
-- MySQL 5.6+ 特性：在索引遍历过程中过滤 WHERE 条件
SELECT * FROM orders WHERE user_id = 100 AND created_at > '2026-01-01';

-- 索引：(user_id, created_at)
-- 不开启 ICP：先在索引树找到 user_id=100 的所有记录，再回表过滤 created_at
-- 开启 ICP：在索引树遍历时就过滤 created_at，减少回表次数

-- 验证：EXPLAIN 的 Extra 有 "Using index condition"（ICP）
```

### 7. 生产监控建议

```go
// Go 应用层慢查询打点
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    queryDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "mysql_query_duration_seconds",
            Help:    "MySQL query duration",
            Buckets: []float64{.001, .005, .01, .025, .05, .1, .5, 1},
        },
        []string{"query_name"},
    )
)

func getOrders(userID int64) ([]Order, error) {
    timer := prometheus.NewTimer(queryDuration.WithLabelValues("get_orders"))
    defer timer.ObserveDuration()
    
    rows, err := db.QueryContext(ctx, "SELECT * FROM orders WHERE user_id = ?", userID)
    // ...
}
```

---

## 面试追问

**Q：怎么判断一个 SQL 是否需要优化？**
> 通常以 100ms 为基准线，超过 100ms 的查询需要评估优化必要性。还要看 QPS — 一个 50ms 的查询如果 QPS 是 10 万，合计影响时间达 5000s，同样需要优化。

**Q：加了索引反而更慢是怎么回事？**
> 可能是数据量太小（MySQL 优化器认为全表扫描更快），或者选了错误的索引（force index 强制指定），或者索引 cardinality 太低（区分度差，如性别字段）。

**Q：线上有慢查询，怎么快速止血？**
> ① kill 掉当前慢查询；② 通过 EXPLAIN 分析；③ 紧急加索引（pt-online-schema-change）；④ 上线 SQL 限流。

---

## 相关知识点

- 索引原理 → `01-index.md`
- 事务与锁 → `02-transaction.md`
- 分库分表 → `05-sharding.md`
- 主从复制延迟 → `06-replication.md`
