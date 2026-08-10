[🏠 首页](../../../README.md) · [🗤️ 数据库](../../README.md) · [🐬 MySQL](../README.md)

---

# MySQL 索引原理与优化

## 面试官考察意图

索引是 MySQL 面试最高频的考点之一。
初级只知道"加索引会快"，高级要能讲清楚 **B+ 树的结构优势、聚簇索引与二级索引的区别、索引失效的所有场景、覆盖索引的价值**，并能结合慢查询给出优化方案。

---

## 核心答案（30 秒版）

MySQL InnoDB 使用 **B+ 树**作为索引结构。
B+ 树的特点：所有数据存在叶子节点，叶子节点通过双向链表连接，非常适合**范围查询**；树高通常只有 3~4 层，磁盘 I/O 次数可控。

索引分两类：
- **聚簇索引（主键索引）**：叶子节点存整行数据
- **二级索引（普通索引）**：叶子节点存主键值，查到后需**回表**查完整数据

---

## 深度展开

### 1. 为什么是 B+ 树，不是其他结构？

| 结构 | 问题 |
|------|------|
| 哈希表 | 不支持范围查询，不支持排序 |
| 二叉搜索树 | 最坏退化成链表，树高 O(n)，磁盘 I/O 多 |
| B 树 | 非叶节点存数据，一页能存的键更少，树更高；不支持高效范围扫描 |
| **B+ 树** | 非叶节点只存键，分支因子大（1页=16KB，可存数百个键），树高 3~4；叶子双链表支持范围扫描 |

**B+ 树高度估算：**

```
InnoDB 页大小：16KB
主键 bigint(8B) + 指针(6B) = 14B/条
每页可存：16 * 1024 / 14 ≈ 1170 个键

- 3层树：1170 × 1170 × 16 ≈ 2190万行
- 4层树：1170³ × 16 ≈ 256亿行

结论：千万级数据树高不超过 3，每次查询磁盘 I/O ≤ 3 次
```

### 2. 聚簇索引 vs 二级索引

```
聚簇索引（主键索引）
非叶节点：[主键值 | 指向下一层的指针]
叶子节点：[主键值 | 完整行数据] ← 数据和索引在一起

二级索引（如 INDEX(name)）
非叶节点：[name 值 | 指针]
叶子节点：[name 值 | 主键值] ← 只存主键，不存完整数据

查询 SELECT * FROM t WHERE name='Alice'：
  1. 走 name 索引，定位到叶子节点，拿到主键 id=42
  2. 拿 id=42 回聚簇索引查完整数据（回表）
  3. 若只 SELECT id, name → 叶子节点已有，无需回表（覆盖索引）
```

**回表的代价：**
每次回表是一次随机 I/O，大量回表（如全表扫描走二级索引）可能比直接全表扫描还慢，这也是优化器有时选择不走索引的原因。

### 3. 联合索引与最左前缀原则

联合索引 `INDEX(a, b, c)` 的存储是按 **(a, b, c) 字典序排列**：

```
(1, 1, 1)
(1, 1, 2)
(1, 2, 1)
(2, 1, 1)
...
```

**最左前缀原则**：查询条件必须包含索引最左列，才能走索引：

| 查询条件 | 是否走索引 | 原因 |
|----------|-----------|------|
| `WHERE a=1` | ✅ 走 | 用到 a |
| `WHERE a=1 AND b=2` | ✅ 走 | 用到 a, b |
| `WHERE a=1 AND b=2 AND c=3` | ✅ 全走 | 用到 a, b, c |
| `WHERE b=2` | ❌ 不走 | 没有最左列 a |
| `WHERE a=1 AND c=3` | ✅ 部分走 | 用到 a，c 无法用（b 断了） |
| `WHERE a>1 AND b=2` | ✅ 部分走 | a 范围查询后 b 无法用索引 |

**索引下推（ICP，Index Condition Pushdown，MySQL 5.6+）**

```sql
-- 联合索引 (name, age)，查询：
SELECT * FROM t WHERE name LIKE 'A%' AND age > 20;

-- 没有 ICP：引擎层只用 name 过滤，age 条件在 server 层过滤（需回表后才判断）
-- 有 ICP：  引擎层同时用 name + age 过滤，减少回表次数
```

### 4. 覆盖索引

查询字段**全部在索引中**，无需回表，称为覆盖索引，`EXPLAIN` 的 `Extra` 列显示 `Using index`：

```sql
-- 表：user(id PK, name, age, email, ...)
-- 索引：INDEX idx_name_age(name, age)

-- 覆盖索引（只查 name, age，索引叶子节点已有）
SELECT name, age FROM user WHERE name = 'Alice';

-- 无法覆盖（需要 email，回表）
SELECT name, age, email FROM user WHERE name = 'Alice';

-- 覆盖索引优化：如果高频查询需要 email，考虑联合索引加入 email
-- INDEX idx_name_age_email(name, age, email)  -- 以空间换时间
```

### 5. 索引失效的所有场景

```sql
-- 假设索引：INDEX(name), INDEX(age), INDEX(name, age)

-- ❌ 1. 对索引列做函数运算
WHERE YEAR(created_at) = 2024
-- ✅ 改为：WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- ❌ 2. 隐式类型转换（name 是 varchar，传入数字）
WHERE name = 123
-- ✅ 改为：WHERE name = '123'

-- ❌ 3. LIKE 以通配符开头
WHERE name LIKE '%Alice%'
-- ✅ 前缀匹配可以走索引：WHERE name LIKE 'Alice%'

-- ❌ 4. OR 连接非索引列（有一个列无索引，整个 OR 不走）
WHERE name = 'Alice' OR email = 'a@b.com'  -- email 无索引
-- ✅ 改为 UNION ALL

-- ❌ 5. NOT IN / NOT EXISTS（有时候走，取决于选择性，但通常不走）
WHERE id NOT IN (1, 2, 3)

-- ❌ 6. 联合索引不满足最左前缀
WHERE age = 20  -- 联合索引 (name, age)，跳过了 name

-- ❌ 7. 区分度太低，优化器放弃（如 status 只有 0/1）
WHERE status = 1  -- 全表 50% 的数据都是 status=1，全表扫描更快

-- ❌ 8. 范围查询后的联合索引列失效
WHERE name > 'A' AND age = 20  -- age 无法用索引
```

### 6. EXPLAIN 关键字段解读

```sql
EXPLAIN SELECT * FROM user WHERE name = 'Alice'\G
```

| 字段 | 重点关注值 |
|------|-----------|
| `type` | **system > const > eq_ref > ref > range > index > ALL**，ALL 是全表扫描要避免 |
| `key` | 实际使用的索引，NULL 表示没走索引 |
| `rows` | 预估扫描行数，越小越好 |
| `Extra` | `Using index`（覆盖索引）✅ `Using filesort`（额外排序）⚠️ `Using temporary`（临时表）❌ |

```
type 级别说明：
- const：主键/唯一索引等值查询，最多1行（最优）
- eq_ref：join 时每条记录用主键/唯一索引匹配
- ref：非唯一索引等值查询
- range：索引范围扫描（BETWEEN, >, <, IN）
- index：扫描整个索引树（比 ALL 好，但仍需优化）
- ALL：全表扫描（最差）
```

### 7. 生产经验

**场景 1：分页深度查询慢**

```sql
-- 慢：OFFSET 100000，需要扫描并丢弃 10 万行
SELECT * FROM orders ORDER BY id LIMIT 100000, 10;

-- 优化：游标分页（记住上次的 id）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 10;

-- 或：延迟关联
SELECT * FROM orders
JOIN (SELECT id FROM orders ORDER BY id LIMIT 100000, 10) t
USING(id);
```

**场景 2：ORDER BY 触发 filesort**

```sql
-- 慢：name 索引存在，但 ORDER BY age 需要 filesort
SELECT * FROM user WHERE name = 'Alice' ORDER BY age;

-- 优化：建联合索引，让 ORDER BY 用上索引
ALTER TABLE user ADD INDEX idx_name_age(name, age);
-- WHERE name='Alice' ORDER BY age → 索引已按 (name, age) 排好序
```

**场景 3：写多读少时索引过多拖慢写入**

```
经验：单表索引数量建议不超过 5 个
每次 INSERT/UPDATE/DELETE 都需要维护所有索引的 B+ 树
定期用 sys.schema_unused_indexes 查出未使用索引并删除
```

---

## 高频追问

**Q：主键为什么推荐自增整数，而不用 UUID？**

UUID 是随机值，插入时需要在 B+ 树中间位置插入，导致**页分裂**（将一个数据页拆成两个），产生大量碎片，写入性能差。自增主键总是追加到末尾，不触发页分裂。

**Q：InnoDB 和 MyISAM 索引的区别？**

| | InnoDB | MyISAM |
|---|---|---|
| 索引类型 | 聚簇索引 | 非聚簇索引 |
| 数据存储 | 和主键索引在一起 | 单独的 .MYD 文件 |
| 事务 | 支持 | 不支持 |
| 外键 | 支持 | 不支持 |

**Q：什么情况下不建议加索引？**

- 区分度低的列（如 boolean、status）
- 频繁更新的列（维护索引开销大）
- 小表（全表扫描比走索引更快）
- 已有联合索引的前缀列（冗余索引）

**Q：索引长度对性能有影响吗？**

对于字符串字段，可以只对前 N 个字符建索引（前缀索引）：

```sql
-- 对 email 前 10 个字符建索引，减少索引大小
ALTER TABLE user ADD INDEX idx_email(email(10));

-- 代价：无法做覆盖索引（叶子节点只有前缀，不是完整值）
```

---

## 延伸阅读

- [MySQL 官方文档：How MySQL Uses Indexes](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html)
- [《高性能 MySQL》第 5 章：索引优化](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)
- [Use The Index, Luke](https://use-the-index-luke.com/)（索引优化专项网站）

---

**[← 上一篇：返回目录](../README.md)** · **[下一篇：事务、隔离级别与 MVCC →](./02-transaction.md)**
