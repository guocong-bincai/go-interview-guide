# ClickHouse 存储设计：排序键 / 分区键选型、稀疏索引与物化视图

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：MergeTree、ORDER BY、PARTITION BY、稀疏索引、granule、物化视图、TTL、去重

## 面试官考察意图

面试官想确认你是「把 ClickHouse 当 MySQL 用」还是真的理解列存 + 有序存储的取舍：

1. "建表时 `ORDER BY` 和 `PRIMARY KEY` 是什么关系？" → 引出稀疏索引与主键语义
2. "分区键怎么选？分区多了会怎样？" → 引出分区数量与 Merge 压力
3. "实时报表怎么做到只查增量？" → 引出物化视图与预聚合

---

## 核心答案（30 秒版）

| 问题 | 一句话回答 |
|------|-----------|
| `ORDER BY` 和 `PRIMARY KEY` 关系？ | CK 的 `PRIMARY KEY` 必须**是 `ORDER BY` 的前缀**（缺省等于 `ORDER BY`），它只是**稀疏索引**，**不保证唯一** |
| 稀疏索引怎么工作？ | 每 8192 行（一个 granule）取第一行的排序键值建一条索引 → 索引很小，能常驻内存，靠二分定位 granule |
| 分区键怎么选？ | 用**低基数的时间维度**（如 `toYYYYMM(ts)`），分区数控制在几十~几百；分区过多导致后台 Merge 与元数据压力 |
| 怎么预聚合？ | `MATERIALIZED VIEW` + `AggregatingMergeTree` / `SummingMergeTree`，写入即增量聚合 |
| 主键去重？ | 普通 `MergeTree` **不去重**；要按主键去重需 `ReplacingMergeTree`（合并时去重，非实时） |

---

## 深度展开

### 1. 排序键（ORDER BY）：ClickHouse 的「聚簇索引」

```sql
CREATE TABLE events (
    ts          DateTime,
    user_id     UInt64,
    event       LowCardinality(String),
    amount      Decimal(18, 2)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)       -- 按月分区
ORDER BY (event, ts)            -- 排序键 = 稀疏索引依据
SETTINGS index_granularity = 8192;
```

- 数据在磁盘上**按 `ORDER BY` 物理有序**，同一批数据的 granule 内有序 → 范围查询只需读少量 granule。
- **查询过滤要「最左前缀」用排序键**：`WHERE event = 'pay' AND ts > ...` 能裁剪；只写 `WHERE ts > ...` 则要扫全部 granule（除非 `event` 基数极低、优化器仍能跳过 granule）。
- **基数是关键**：`ORDER BY` 应该把**高基数**列放前面（如 `user_id`、`ts`），低基数放前面会让每个值覆盖太多行、granule 裁剪失效。

```sql
-- 对比：把低基数的 event 放前面，ts 范围查询几乎无法裁剪 granule
ORDER BY (event, ts)   -- ❌ 若 event 只有几种值，每个值横跨海量 granule
ORDER BY (ts, user_id) -- ✅ 时间范围查询能精准裁剪
```

### 2. 稀疏索引与 granule

```
磁盘布局（按 ORDER BY 有序）：
  granule 0 (8192 行) | granule 1 (8192 行) | granule 2 ...
        ↑                     ↑
  索引记录(键值, granule 起始位置) —— 每个 granule 只存一条 → "稀疏"

查询 WHERE ts BETWEEN t1 AND t2：
  1. 在稀疏索引上二分，定位到覆盖 [t1, t2] 的 granule 区间
  2. 只读取这些 granule 的列数据（列存按列读取，未涉及列不读）
  3. 向量化执行 + SIMD 批量计算
```

### 3. 物化视图：实时预聚合

```sql
-- 明细表
CREATE TABLE events (...) ENGINE = MergeTree ...;

-- 物化视图：每当 events 有新数据写入，自动把增量聚合结果写入目标表
CREATE MATERIALIZED VIEW mv_daily
ENGINE = SummingMergeTree
ORDER BY (event, day)
AS
SELECT
    event,
    toDate(ts) AS day,
    count()           AS cnt,
    sum(amount)       AS total
FROM events
GROUP BY event, day;
```

**要点：**

- 物化视图是**写入触发的增量计算**（类似触发器），只处理新写入的数据块，**不含历史数据**——所以「表里已有数据 + 新建 MV」需要 `INSERT INTO ... SELECT` 手工回填历史。
- MV 查询时必须用 `FINAL` 或 `sum(cnt)` 聚合，因为 `SummingMergeTree` 的合并是异步的，同一 `ORDER BY` 键可能有多行未合并。
- 大宽表场景可用 `AggregatingMergeTree` 配合 `-State`/`-Merge` 函数做精确去重（`uniqState`/`uniqMerge`）。

### 4. 生产踩坑

| 坑 | 现象 | 处理 |
|----|------|------|
| 分区过多 | 每秒建分区 / 按用户 ID 分区 → Merge 卡死 | 按月/按天分区，分区数控制在千级以内 |
| 把小表当 MySQL 用 | 逐行 `ALTER UPDATE` 极慢 | CK 更新/删除是异步 mutation，尽量用 `ReplacingMergeTree` + 查询去重 |
| 高频点查 | `WHERE id = ?` 单行查询慢 | CK 适合**批量扫描**；点查仍用 MySQL/Redis |
| 缺少 `LowCardinality` | 枚举列存 String 占用大 | 低基数列用 `LowCardinality(String)`，字典编码压缩 |
| 不做 TTL | 存储无限膨胀 | `TTL ts + INTERVAL 90 DAY TO VOLUME 'cold'` 冷热分层 |

---

## 高频追问

### Q1：ClickHouse 为什么快？

列式存储（只读需要的列）+ 数据按排序键物理有序（granule 裁剪）+ 稀疏索引常驻内存 + 向量化执行（SIMD 批量处理）+ 数据高度压缩 + 无事务无锁的 append-only 写入。它的快是「批处理扫描」的快，不是「点查事务」的快。

### Q2：ClickHouse 的 `PRIMARY KEY` 是主键吗？

不是唯一约束。它只是稀疏索引的别名，且必须是 `ORDER BY` 的前缀。**允许重复**。要「按主键去重」必须用 `ReplacingMergeTree(version)`，并且去重只在后台 merge 时发生，查询时需 `FINAL` 或 `GROUP BY` 兜底。

### Q3：ClickHouse vs MySQL vs ES 怎么选？

- **MySQL**：OLTP，点查、事务、强一致。
- **ClickHouse**：OLAP，海量数据的聚合/明细大宽表扫描，写入一次读多次。
- **ES**：全文检索 + 聚合，近实时、支持复杂条件过滤。
- 常见组合：MySQL 做事实源 → binlog/Kafka → ClickHouse 做报表 → ES 做搜索。

---

## 延伸阅读

- [ClickHouse Docs - MergeTree engine](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree)
- [ClickHouse Docs - Materialized views](https://clickhouse.com/docs/en/sql-reference/statements/create/view#materialized-view)
- [ClickHouse Docs - Sparse primary index](https://clickhouse.com/docs/en/guides/best-practices/sparse-primary-indexes)
