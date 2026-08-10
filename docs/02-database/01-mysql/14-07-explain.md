[🏠 首页](../../../README.md) · [🗤️ 数据库](../../README.md) · [🐬 MySQL](../README.md)

---

# EXPLAIN 输出字段逐一解读与实战案例

## 面试官考察意图

MySQL 优化面试必问 EXPLAIN。
初级候选人能说出 type/rows/key 几个字段，高级候选人能**逐字解读 12 个输出字段、辨别 Optimizer Trace 与 EXPLAIN 的区别、结合真实 SQL 说出调优思路**。
这道题是高频追问题海，一旦回答不准确（如把 ref 误说成 const、不知道 Using filesort 的真实代价），立刻暴露知识深度不足。

---

## 核心答案（30 秒版）

MySQL 8.0 `EXPLAIN` 输出 12 个字段，核心关注 6 个：

| 字段 | 关注点 |
|------|--------|
| `type` | **ALL（全表）→ const（主键）**，尽量避免 ALL |
| `key` | 实际用到的索引，NULL = 没用索引 |
| `rows` | 预估扫描行数，越少越好（但可能被统计误差欺骗）|
| `Extra` | `Using index`(覆盖) ✅ / `Using filesort`(糟糕) ❌ / `Using temporary` ❌ |
| `possible_keys` | 可选索引，与 key 对比判断是否用错索引 |
| `filtered` | 过滤后剩余百分比，配合 rows 算真实扫描量 |

**最常见的问题 SQL：type=ALL 且 Extra 包含 Using filesort。**

---

## 深度展开

### 1. EXPLAIN 输出字段逐字解读

MySQL 8.0 支持两种输出格式：
- `EXPLAIN ANALYZE`（执行计划 + 实际运行时（开启统计））
- `EXPLAIN FORMAT=TREE/JSON`（树形/JSON 结构）

我们以标准表格式为准。

#### id：执行序号

同一 SELECT 中的多个子查询按 id 降序执行（先执行 id 大的）。

```sql
EXPLAIN SELECT * FROM (SELECT id FROM user WHERE id = 1) t JOIN order o ON t.id = o.user_id;
-- id=2 的子查询先执行，id=1 的外层后执行
-- id 相同则按从上到下顺序
```

| 值 | 含义 |
|----|------|
| 数字 | 该行的执行顺序，id 越大优先级越高 |
| NULL | 结果行是 UNION 的 TOTAL 行 |

#### select_type：查询类型

| 值 | 含义 |
|----|------|
| SIMPLE | 简单 SELECT，无 UNION 或子查询 |
| PRIMARY | 最外层 SELECT |
| SUBQUERY | 子查询（在 SELECT 列表中） |
| DERIVED | 派生表（FROM 子句中的子查询）|
| UNION | UNION 第二个及之后的 SELECT |
| UNION RESULT | UNION 结果集 |

```sql
EXPLAIN SELECT * FROM user u
WHERE u.id IN (SELECT user_id FROM order WHERE status = 1);  -- SUBQUERY

EXPLAIN SELECT * FROM user UNION SELECT * FROM user_backup;
-- row1: SIMPLE (user)
-- row2: UNION
-- row3: UNION RESULT
```

#### table：涉及的表

- 表名
- `<derived N>`：第 N 行的派生表
- `<union M,N>`：UNION 结果集

#### partitions：分区信息

如果表按年份分区，`partitions` 列显示实际扫描了哪些分区：

```sql
EXPLAIN SELECT * FROM events PARTITION (p2024)
WHERE created_at > '2024-01-01';
-- partitions = p2024（只扫描该分区）
```

#### type：访问类型（最核心字段之一）

按性能从优到劣排序：

```
system > const > eq_ref > ref > ref_or_null > range > index > ALL
 ↑最优                                            ↑最差
```

| type | 含义 | 何时出现 |
|------|------|---------|
| `system` | 表只有一行，InnoDB 不出现（MyISAM 静态表）| 系统表 |
| `const` | 主键/唯一索引等值查找，最多匹配 1 行 | `WHERE pk = X` |
| `eq_ref` | JOIN 时，主键/唯一索引与另一表等值匹配，每行只返回 1 条 | `JOIN ... ON pk = X` |
| `ref` | 非唯一索引等值查找，返回所有匹配的行 | `WHERE idx_col = X` |
| `ref_or_null` | ref + 额外扫描 NULL 值 | `WHERE col = X OR col IS NULL` |
| `range` | 索引范围扫描（BETWEEN、\>、\<、IN、LIKE 前缀）| `WHERE id > 10` |
| `index` | 全索引扫描（不用排序，但 I/O 仍多）| `SELECT idx_col FROM t` |
| `ALL` | **全表扫描**，必须优化 | 无 WHERE 条件 |

**实战经验：**
- `const` 和 `eq_ref` 是最优解，通常是主键/唯一索引访问
- `ref` 正常（普通索引等值查询），可接受
- `range` 可接受（范围查询有索引支持）
- `ALL` 出现 → 必须优化（加索引、改写法）
- `index` 出现 → 比 ALL 好但仍然昂贵（扫描整个索引树）

```sql
-- const: 主键等值
EXPLAIN SELECT * FROM user WHERE id = 100;  -- const

-- eq_ref: JOIN 时主键等值
EXPLAIN SELECT * FROM order o JOIN user u ON o.user_id = u.id;  -- eq_ref

-- ref: 普通索引等值
EXPLAIN SELECT * FROM user WHERE name = 'Alice';  -- ref (name 有普通索引)

-- range: 范围查询
EXPLAIN SELECT * FROM user WHERE id BETWEEN 100 AND 200;  -- range

-- ALL: 全表扫描
EXPLAIN SELECT * FROM user WHERE email = 'a@b.com';  -- email 无索引
```

#### possible_keys：可用的索引列表

MySQL 优化器认为可能使用的索引（候选索引）。
**可能为空**（没有可用索引）或**包含多个索引**（需要和 key 列对比判断最终用哪个）。

#### key：实际使用的索引

真正被优化器选中的索引。如果为 NULL → 没走索引。

**常见误区：possible_keys 有值，key 是 NULL**
→ 说明优化器分析了候选索引，但最终选择不走索引（通常因为统计信息不准 or 索引选择代价更高）。

```sql
EXPLAIN SELECT * FROM user WHERE name = 'Alice' AND age = 20;
-- possible_keys: idx_name, idx_age, idx_name_age
-- key: idx_name_age（选择了联合索引而非单列索引）
```

#### key_len：索引使用的字节数

用于判断联合索引使用了前几列：

```
计算规则（InnoDB）：
- int: 4 字节
- bigint: 8 字节
- varchar(N) utf8mb4: 3*N + 2 字节（变长 + 长度前缀）
- char(N) utf8mb4: 4*N 字节
- null: 1 字节
```

```sql
-- 联合索引 INDEX(name, age, city)
-- name varchar(30) utf8mb4: 3*30 + 2 = 92 字节 → 只用了 name
-- key_len=92: 用到 name 列（第一列）
-- key_len=96: 用到 name + age（age int: 4 字节）
-- key_len=96+92=188: 三列都用上了

EXPLAIN SELECT * FROM user WHERE name = 'Alice';
-- key_len = 92（只用 name）

EXPLAIN SELECT * FROM user WHERE name = 'Alice' AND age = 20;
-- key_len = 96（name + age）
```

**通过 key_len 可以推断联合索引的使用程度**，如果只用了前缀列但查询需要后续列，要警惕索引失效。

#### ref：与索引比较的列

显示索引列与什么值比较：

| 值 | 含义 |
|----|------|
| `const` | 与常量比较，如 `WHERE id = 1` |
| `func` | 使用了函数或表达式 |
| `db.t.col` | 与其他表的列关联 |
| `NULL` | system/ALL/全表扫描时 |

```sql
EXPLAIN SELECT * FROM user u JOIN order o ON u.id = o.user_id;
-- ref: const（o.user_id 与常量 u.id 比较）

EXPLAIN SELECT * FROM user WHERE YEAR(created_at) = 2024;
-- ref: func（对索引列使用函数，索引失效）
```

#### rows：预估扫描行数

**优化器基于统计信息（InnoDB 的 Cardinality）估算的扫描行数**，不是精确值。
与 filtered 相乘得到实际处理的行数：

```
实际处理行数 ≈ rows × filtered / 100
```

**常见陷阱：rows 被低估**
统计信息陈旧（ANALYZE TABLE 未执行）、直方图缺失 → 优化器估算错误 → 选择错误的执行计划。

```sql
-- 某表有 1000 万行，rows=1000000 说明优化器估算错了（实际应接近全表）
EXPLAIN SELECT * FROM user WHERE status = 1;
-- 如果 status=1 实际只有 10 行（区分度高），优化器选择了全表扫描
```

#### filtered：过滤后剩余百分比

表示通过 WHERE 条件过滤后剩余数据占总数据的百分比。
配合 rows 估算最终结果量：

```
预估返回行数 = rows × filtered / 100
```

```sql
EXPLAIN SELECT * FROM user WHERE name = 'Alice' AND age > 20;
-- rows = 10000（扫描了 1 万行）
-- filtered = 5（只有 5% 通过过滤）
-- 实际结果 ≈ 500 行
```

#### Extra：额外执行信息（关键！）

这是最容易暴露问题的字段。

##### ✅ 好的 Extra

| 值 | 含义 |
|----|------|
| `Using index` | **覆盖索引**，不需要回表，性能最优 |
| `Using index condition` | **索引下推（ICP）**，在索引层过滤条件，减少回表 |

```sql
-- 覆盖索引
EXPLAIN SELECT name, age FROM user WHERE name = 'Alice';
-- Extra: Using index

-- 索引下推
EXPLAIN SELECT * FROM user WHERE name LIKE 'A%' AND age > 20;
-- Extra: Using index condition
```

##### ⚠️ 需要关注的 Extra

| 值 | 含义 | 优化方向 |
|----|------|---------|
| `Using where` | 服务层用 WHERE 过滤（正常，需要回表）| 正常 |
| `Using index for skip scan` | 跳过扫描，索引不连续场景 | 考虑覆盖索引 |
| `Using join buffer` | 使用了 join buffer（小表驱动大表）| 考虑增大 join_buffer_size |

##### ❌ 糟糕的 Extra

| 值 | 含义 | 后果 |
|----|------|------|
| `Using filesort` | **无法用索引排序，需要额外排序操作** | P99 延迟飙升 |
| `Using temporary` | **使用了临时表存储中间结果** | 内存/磁盘溢出 |
| `Using filesort, Using temporary` | **最差组合**，临时表 + 排序 | 性能灾难 |

```sql
-- Using filesort：ORDER BY 字段没有索引
EXPLAIN SELECT * FROM user WHERE name = 'Alice' ORDER BY age;
-- Extra: Using filesort（age 无索引，无法利用 (name, age) 联合索引的顺序）

-- 优化：建联合索引 (name, age)
ALTER TABLE user ADD INDEX idx_name_age(name, age);

-- Using temporary：GROUP BY / DISTINCT 无法利用索引
EXPLAIN SELECT name, COUNT(*) FROM user GROUP BY name;
-- Extra: Using temporary; Using filesort
-- name 字段无索引，GROUP BY 只能建哈希临时表后排序

-- 优化：建 name 索引
ALTER TABLE user ADD INDEX idx_name(name);
```

---

### 2. 实战案例

#### 案例 1：联合索引列顺序选错导致回表爆炸

**背景**：订单表 `orders(id, user_id, status, created_at)`，联合索引 `idx_user_status(user_id, status)`。

```sql
-- 查询语句
SELECT * FROM orders
WHERE user_id = 100 AND status = 1
ORDER BY created_at DESC
LIMIT 10;
```

```sql
EXPLAIN SELECT * FROM orders
WHERE user_id = 100 AND status = 1
ORDER BY created_at DESC
LIMIT 10\G
```

**EXPLAIN 输出：**

```
type: ref
key: idx_user_status
rows: 50000       ← 命中 5 万行
Extra: Using index condition; Using filesort
```

**问题分析：**
- `Using index condition` = 索引下推，减少了回表次数（好）
- `Using filesort` = `created_at` 没有索引，排序在内存/磁盘中完成（坏）
- 5 万次回表 + 5 万行排序 = 接口 P99 从 5ms 飙升到 2s

**优化方案：**

```sql
-- 改写查询：延迟关联 + 只排序主键
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders
    WHERE user_id = 100 AND status = 1
    ORDER BY created_at DESC
    LIMIT 10
) t ON o.id = t.id;

-- 或者新建覆盖索引
ALTER TABLE orders ADD INDEX idx_user_status_created(user_id, status, created_at);
-- 让 ORDER BY 直接从索引中获取有序结果，无需 filesort
```

#### 案例 2：深分页导致回表次数爆炸

**背景**：`SELECT * FROM products WHERE category = 'electronics' ORDER BY id LIMIT 100000, 10`

```sql
EXPLAIN SELECT * FROM products
WHERE category = 'electronics'
ORDER BY id LIMIT 100000, 10\G
```

**EXPLAIN 输出：**

```
type: ref
possible_keys: idx_category
key: idx_category
rows: 1000000      ← 扫描 100 万行
Extra: Using index condition; Using filesort
```

**问题分析：**
- `category` 索引中，`id` 不是有序的（只有 category 有序），所以有 filesort
- `LIMIT 100000, 10` 需要跳过前 10 万行，只取 10 行
- 即使走了索引，也要先查出 10 万零 10 条记录，再丢弃前 10 万条

**优化方案 1：游标分页（最优）**

```sql
-- 记住上次最后一个 id
SELECT * FROM products
WHERE category = 'electronics' AND id > :last_id
ORDER BY id LIMIT 10;
-- type: range, key: PRIMARY, rows: 10（精确扫描）
```

**优化方案 2：延迟关联**

```sql
SELECT p.* FROM products p
INNER JOIN (
    SELECT id FROM products
    WHERE category = 'electronics'
    ORDER BY id LIMIT 100000, 10
) t ON p.id = t.id;
-- 内层只扫主键索引树（覆盖索引），外层根据主键回表取完整数据
```

#### 案例 3：OR 导致全表扫描

```sql
EXPLAIN SELECT * FROM user WHERE name = 'Alice' OR phone = '13800000000'\G
```

**EXPLAIN 输出：**

```
type: ALL
key: NULL
rows: 100000
Extra: Using where; Using filesort
```

**问题分析：**
- `phone` 字段有索引，但 `name` 无索引
- MySQL 5.7 及之前：OR 条件下，只要有一个字段无索引 → 全表扫描
- MySQL 8.0：优化器分析成本后仍可能选择全表扫描

**优化方案：拆分为 UNION ALL**

```sql
SELECT * FROM user WHERE name = 'Alice'
UNION ALL
SELECT * FROM user WHERE phone = '13800000000' AND name != 'Alice';

-- 给 name 也加索引
ALTER TABLE user ADD INDEX idx_name(name);
```

#### 案例 4：MySQL 8.0 EXPLAIN ANALYZE（实际执行分析）

MySQL 8.0 新增 `EXPLAIN ANALYZE`，输出实际运行时间和预估时间对比：

```sql
EXPLAIN ANALYZE
SELECT o.* FROM orders o
INNER JOIN user u ON o.user_id = u.id
WHERE u.city = 'Beijing'\G
```

**输出示例：**

```
-> Nested loop inner join  (cost=12345.00 rows=1000)
    (actual time=0.021..50.123 rows=980 loops=1)
    -> Index lookup on u using idx_city (city='Beijing')
        (cost=1000.00 rows=1000)
        (actual time=0.010..0.500 rows=980 loops=1)
    -> Index lookup on o using idx_user_id (user_id=u.id)
        (cost=10.00 rows=1)
        (actual time=0.005..0.010 rows=1 loops=980)
```

**解读：**
- `cost=12345` vs `actual time`：成本估算 vs 实际耗时
- 如果差距大（10 倍以上）→ 统计信息可能不准确
- actual time 给出首次返回（0.021ms）和总时间（50.123ms）
- `rows=980` 是实际返回行数，与预估 rows=1000 相近 → 统计信息准确

---

### 3. EXPLAIN FORMAT=TREE：更直观的执行计划

MySQL 8.0+ 支持树形输出：

```sql
EXPLAIN FORMAT=TREE
SELECT * FROM orders WHERE user_id = 100\G
```

```
-> Index lookup on orders using idx_user_id (user_id=100)
    (cost=0.35 rows=1)
```

对于复杂 JOIN，TREE 格式比表格更直观：

```
-> Nested loop inner join
    -> Index lookup on o using idx_status (status=1)
        (cost=5000 rows=50000)
    -> Index lookup on u using PRIMARY (id=o.user_id)
        (cost=0.25 rows=1)
```

---

## 高频追问

**Q：rows 和实际返回行数不符怎么办？**

执行 `ANALYZE TABLE tbl` 重新收集统计信息。
如果仍然不准，检查 `innodb_stats_auto_recalc` 是否被禁用，或手动设置 `STATISTICS`。

**Q：Using filesort 一定慢吗？**

不一定：
- 结果集小（< 10MB），内存排序很快（`sort_buffer_size` 默认 256KB）
- `EXPLAIN` 看不到内存 vs 磁盘排序的区分
- 当 `sort_buffer_size` 不够时会走 `max_sort_length` 磁盘文件排序
- 经验：filesort 的 P99 延迟在 10 万行以内通常 < 5ms，超过 100 万行可能 > 100ms

**Q：如何判断一个索引是否值得加？**

1. 看查询频率（高频查询优先加索引）
2. 看 `SELECTIVITY = COUNT(DISTINCT col) / COUNT(*)`（区分度）
3. 看写入比例（每多一个索引，INSERT 慢约 10%）
4. 看覆盖程度（联合索引尽量覆盖高频查询字段）

**Q：EXPLAIN 和 Optimizer Trace 的区别？**

| | EXPLAIN | Optimizer Trace |
|---|---|---|
| 何时用 | 快速查看执行计划 | 理解优化器决策过程 |
| 内容 | 最终执行计划 | 所有候选计划 + 成本比较 |
| 开销 | 低 | 高（记录大量中间状态）|

```sql
SET optimizer_trace = 'enabled=on';
SELECT ...;
SELECT * FROM information_schema.OPTIMIZER_TRACE;
SET optimizer_trace = 'enabled=off';
```

**Q：有没有办法直接在生产环境小流量验证索引？**

```sql
-- 使用 optimizer_switch 关闭某些优化器特性对比
SET SESSION optimizer_switch = 'index_condition_pushdown=off';

-- 或用 SQL hint 强制使用某索引
SELECT * FROM user USE INDEX (idx_name) WHERE name = 'Alice';

-- 强制不走某索引
SELECT * FROM user IGNORE INDEX (idx_name) WHERE name = 'Alice';
```

---

## 延伸阅读

- [MySQL 8.0 EXPLAIN 输出格式官方文档](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [MySQL 8.0 EXPLAIN ANALYZE 官方文档](https://dev.mysql.com/doc/refman/8.0/en/explain-analyze.html)
- [USE The Index, Luke - EXPLAIN 详解](https://use-the-index-luke.com/sql/explain-plan)
- [《高性能 MySQL》第 4 章：高性能索引策略](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)
- [MySQL optimizer trace 使用](https://dev.mysql.com/doc/refman/8.0/en/optimizer-trace.html)

---

**[← 上一篇：主从复制 →](./06-replication.md)** · **[下一篇：返回目录](../README.md)**
