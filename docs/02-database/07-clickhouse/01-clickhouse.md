# ClickHouse 核心原理与 OLAP 选型

> 考察频率：★★★☆☆  优先级：P1
> 关键词：列式存储、MergeTree、向量化执行、SIMD、OLAP

## ClickHouse 是什么

ClickHouse 是俄罗斯 Yandex 开源的**列式存储 OLAP 数据库**，专门处理**大规模数据分析查询**，单查询可在秒级扫描数十亿行数据。

**核心能力：**
- 10 亿行数据聚合查询 < 1 秒
- 相比 MySQL 快 100-1000 倍
- 支持 SQL 查询，迁移成本低

---

## 1. 列式存储原理

### 行存 vs 列存

```
行存（MySQL / MongoDB）：每行数据连续存储
┌─────┬───────┬─────────┬──────┐
│ R1A │ R1B   │ R1C     │ R1D  │
├─────┼───────┼─────────┼──────┤
│ R2A │ R2B   │ R2C     │ R2D  │
└─────┴───────┴─────────┴──────┘
读取 SELECT A,D → 需要跳过整行（浪费 IO）

列存（ClickHouse）：每列数据连续存储
┌─────┬─────┬─────┬─────┐ 列 A:  [R1A, R2A, R3A, ...]
│ R1A │ R2A │ R3A │ ... │ 列 B:  [R1B, R2B, R3B, ...]
└─────┴─────┴─────┴─────┘ 列 C:  [R1C, R2C, R3C, ...]
┌─────┬─────┬─────┬─────┐
│ R1B │ R2B │ R3B │ ... │
└─────┴─────┴─────┴─────┘
读取 SELECT A,D → 只读 A 列和 D 列（IO 极少）
```

**列存为什么快（OLAP 场景）：**
1. **IO 极少**：只读查询涉及的列，不读无关列
2. **压缩率高**：同一列数据类型一致，压缩效率高（10-100x）
3. **向量化友好**：批量处理同一列数据，利用 CPU 缓存和 SIMD 指令

### 数据组织：Part → Column → Mark

```
ClickHouse 数据存储层级：
DataPart（数据分区）
├── Column A (Column File)
│   └── [data] + [marks]  ← marks 记录偏移，用于二分查找
├── Column B (Column File)
│   └── [data] + [marks]
└── Primary.idx (稀疏索引)  ← 每 8192 行一条索引，快速定位

查询流程：
1. 用 Primary.idx 找到目标数据块（Mark 范围）
2. 读取对应 Mark 范围内的 Column Data
3. 解压 → 过滤 → 聚合
```

### 压缩编码

ClickHouse 对不同数据类型使用最优压缩算法：

| 数据类型 | 推荐压缩算法 | 原理 |
|---------|------------|------|
| 重复值多的整数 | `Delta` / `DoubleDelta` | 存差值，差值小 |
| 浮点数 | `Gorilla` | XOR 编码，相同位不存 |
| 字符串/通用 | `ZSTD` | 综合压缩率最高 |
| 低基数字符串 | `LowCardinality` | 字典编码 |

```sql
-- 建表时指定列压缩算法
CREATE TABLE events (
    event_id   UInt64 CODEC(ZSTD),
    user_id    UInt32 CODEC(Delta, ZSTD),
    device_id  String CODEC(LowCardinality, ZSTD),
    created_at DateTime CODEC(DoubleDelta, ZSTD)
) ENGINE = MergeTree()
ORDER BY (event_id, created_at);
```

---

## 2. MergeTree 引擎族（核心）

MergeTree 是 ClickHouse 最核心的表引擎，类似 MySQL 的 InnoDB。

### MergeTree 基本原理

```
MergeTree 数据组织：
PARTITION BY toYYYYMM(event_date)  -- 按月分区
ORDER BY (user_id, event_id)      -- 主键排序（决定数据物理顺序）
```

数据写入时：
1. 先写入一个 Part（内存 buffer，满后刷盘）
2. 后台合并（Merge）多个小 Part 成大 Part
3. 同 Partition 的 Part 合并，ORDER BY 相同的行会"去重合并"

### 关键概念

**主键（PRIMARY KEY）vs ORDER BY：**

```sql
-- MergeTree：主键用于索引，ORDER BY 决定数据物理顺序
-- 两者可以不同，但 ORDER BY 必须是主键的前缀
CREATE TABLE t (
    user_id  UInt32,
    order_id UInt32,
    amount   Float64,
    PRIMARY KEY (user_id)          -- 索引：快速定位 user_id
    ORDER BY (user_id, order_id)   -- 物理顺序：user_id 内按 order_id 排序
)
-- 重要：ORDER BY 才是实际决定数据排列顺序的
```

**分区（PARTITION BY）：**
- 数据按分区字段拆分目录（如 `/201901/`, `/201902/`）
- 不同分区的数据不会合并在一起
- 分区过多会导致小文件问题，建议按月分区

### ReplacingMergeTree（去重）

```sql
CREATE TABLE visits (
    user_id   UInt32,
    page_id   UInt32,
    ts        DateTime,
    version   UInt32  -- 版本号，用于去重时保留最新
) ENGINE = ReplacingMergeTree(version)
ORDER BY (user_id, page_id)
-- 同一 ORDER BY 键的多条数据，保留 version 最大的那条
-- 注意：去重在合并时才发生，不是写入时立即去重
```

**实战注意：**
```sql
-- 即使有 ReplacingMergeTree，也不要依赖"写入即去重"
-- 查询时必须用 GROUP BY 或 DISTINCT 确保结果正确
SELECT user_id, page_id, max(ts) as latest
FROM visits
GROUP BY user_id, page_id
```

### SummingMergeTree / AggregatingMergeTree

```sql
-- SummingMergeTree：预聚合（自动 SUM 相同 ORDER BY 键的值）
CREATE TABLE order_items (
    order_id   UInt32,
    product_id UInt32,
    amount     Float64
) ENGINE = SummingMergeTree(amount)
ORDER BY (order_id, product_id)
-- 合并时，同 order_id 的 amount 会自动相加

-- AggregatingMergeTree：自定义聚合（适合精确去重）
CREATE TABLE user_actions (
    user_id   UInt32,
    action    String,
    ts        DateTime
) ENGINE = AggregatingMergeTree()
ORDER BY (user_id, action)
```

---

## 3. 向量化执行

### SIMD 指令批量处理

ClickHouse 不逐行处理数据，而是**批量处理**：

```
传统行存处理（MySQL）：
for row in rows:
    process(row)  # 每行一次 CPU 指令调用

ClickHouse 向量化处理：
data = load_batch(column_a)      # 一次性加载 10000 行到内存
result = simd_process(data)       # 一条 SIMD 指令处理 256 行
```

**SIMD（Single Instruction Multiple Data）：**
- 一条 CPU 指令同时处理 256 位数据（如 8 个 Float32）
- 比逐行处理快 8-16 倍
- 充分利用 CPU L1/L2 缓存（批量数据在缓存中）

### 为什么 ClickHouse 单核性能远超 MySQL

| 优化点 | ClickHouse | MySQL |
|--------|-----------|-------|
| 数据组织 | 列存，只读需要的列 | 行存，读整行 |
| 压缩 | ZSTD/Gorilla 压缩，数据更小 | 无列级压缩 |
| 处理方式 | 向量化 + SIMD 批量 | 逐行处理 |
| 索引 | 稀疏索引（每 8192 行一跳）| 密集索引（每行一跳）|

---

## 4. 与其他 OLAP 选型对比

| 维度 | ClickHouse | TiFlash | Apache Doris | Elasticsearch |
|------|-----------|---------|-------------|--------------|
| **写入延迟** | 中（批量好）| 低（实时）| 中 | 低 |
| **查询性能** | 极高 | 高 | 高 | 中（聚合弱）|
| **SQL 兼容** | 高 | 高 | 高 | 低（DSL）|
| **实时性** | 秒级 | 秒级 | 秒级 | 秒级 |
| **生态成熟度** | 高 | 高（TiDB 生态）| 中 | 高 |
| **运维复杂度** | 高（调优复杂）| 中 | 低 | 中 |
| **最佳场景** | 固定报表、大数据量聚合 | MySQL 实时分析 | 快速查询、统一分析 | 全文检索+分析 |
| **不支持** | 高并发点查、频繁更新 | 极大量明细查询 | 超大规模数据 | 复杂 JOIN |

**选型建议：**
- **日志分析 + 固定报表**：ClickHouse
- **MySQL 数据实时分析**：TiFlash
- **快速 BI 看板**：Apache Doris
- **搜索 + 分析混合**：Elasticsearch

---

## 5. 典型使用场景

### 场景 1：日志分析（替代 ELK 中 ES 的分析角色）

```
架构：Kafka → ClickHouse → Grafana
                ↓
         用户行为分析：漏斗、留存、路径

优势：
- 比 ES 的 SQL 支持更完整
- 查询速度更快（聚合场景）
- 存储成本更低（ZSTD 压缩）
```

### 场景 2：监控指标存储

```
架构：Prometheus → Thanos → ClickHouse
                ↓
         查询：过去 30 天的 P99 延迟

优势：
- 支持时间序列的预聚合（SummingMergeTree）
- 比 Prometheus 长期存储成本低
- SQL 查询比 PromQL 更灵活
```

### 场景 3：用户行为分析

```sql
-- 漏斗分析：统计每一步的转化率
SELECT
    step1_users,
    step2_users,
    step3_users,
    round(step2_users / step1_users * 100, 2) as conv_1to2,
    round(step3_users / step2_users * 100, 2) as conv_2to3
FROM (
    SELECT
        uniqExact(user_id) as step1_users,
        sum(step = 2) as step2_users,
        sum(step = 3) as step3_users
    FROM user_funnel
    WHERE date = today()
);
```

### 场景 4：广告效果统计

```sql
-- 实时 CTR/CVR 计算
SELECT
    ad_group_id,
    sum(clicks) as total_clicks,
    sum(conversions) as total_conversions,
    round(sum(clicks) / sum(impressions) * 100, 4) as ctr,
    round(sum(conversions) / sum(clicks) * 100, 4) as cvr
FROM ad_stats
WHERE date >= today() - 7
GROUP BY ad_group_id
ORDER BY cvr DESC
LIMIT 100
```

---

## 6. Go 接入实战

### clickhouse-go v2 SDK

```go
import (
    "github.com/ClickHouse/clickhouse-go/v2"
    "github.com/ClickHouse/clickhouse-go/v2/lib/driver"
)

func newClickHouseClient() (driver.Client, error) {
    opts := &clickhouse.Options{
        Addr: []string{"localhost:9000"},
        Auth: clickhouse.Auth{
            Database: "default",
            Username: "default",
            Password: "",
        },
        Settings: clickhouse.Settings{
            "max_execution_time": 60,     // 最大查询时间 60s
            "max_block_size":    10000,   // 每次传输的数据块大小
            "log_level":         1,       // 日志级别
        },
        Debug: false,
    }
    return clickhouse.Open(opts)
}
```

### 批量写入最佳实践

```go
func batchInsert(coll driver.Client) error {
    ctx := context.Background()
    batch, err := coll.PrepareBatch(ctx, "INSERT INTO events")
    if err != nil {
        return err
    }

    for i := 0; i < 10000; i++ {
        // 逐条追加到 batch，内部缓冲
        err := batch.Append(
            uint64(i),
            "event_name",
            time.Now(),
            rand.Float64(),
        )
        if err != nil {
            return err
        }

        // 每 1000 条 flush 一次（控制内存）
        if i > 0 && i%1000 == 0 {
            if err := batch.Send(); err != nil {
                return err
            }
        }
    }
    return batch.Send()
}
```

### 连接池配置

```go
opts := &clickhouse.Options{
    Addr: []string{"localhost:9000", "localhost:9001", "localhost:9002"},
    Auth: clickhouse.Auth{Database: "events", Username: "default"},
    Settings: clickhouse.Settings{
        "max_open_connections": 100,  // 最大并发连接数
        "max_idle_connections": 10,   // 最小空闲连接
        "connection_timeout":    5,    // 连接超时（秒）
        "send_timeout":          300,  // 发送超时
        "receive_timeout":       300, // 接收超时
    },
}
```

### 查询示例

```go
func queryTopUsers(cli driver.Client) ([]TopUser, error) {
    ctx := context.Background()
    rows, err := cli.Query(ctx, `
        SELECT user_id,
               count()     as event_count,
               uniq(event_id) as unique_events
        FROM events
        WHERE created_at >= now() - INTERVAL 7 DAY
        GROUP BY user_id
        ORDER BY event_count DESC
        LIMIT 10
    `)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var results []TopUser
    for rows.Next() {
        var u TopUser
        if err := rows.Scan(&u.UserID, &u.EventCount, &u.UniqueEvents); err != nil {
            return nil, err
        }
        results = append(results, u)
    }
    return results, nil
}
```

---

## 高频追问

### Q1：ClickHouse 为什么不适合高并发点查？

**原因：稀疏索引 + 列存的查询开销：**

1. **稀疏索引**：每 8192 行才一条索引，点查需要遍历大量 Mark 范围
2. **列存特性**：点查时需要解压多个列，CPU 解压开销大
3. **无行级缓存**：ClickHouse 主要优化扫描吞吐量，不是点查延迟

**解决方案：**
- 用 `CLICKHOUSE_GO` 做预聚合，避免实时聚合
- 对高频点查字段加主键索引
- 用 Redis 缓存热点查询结果

### Q2：ReplacingMergeTree 删除数据为什么不是立即生效？

```sql
-- ReplacingMergeTree 在合并时才去重，不是写入时立即去重
INSERT INTO dedup_test VALUES (1, 'A', 1);
INSERT INTO dedup_test VALUES (1, 'B', 2);  -- 同 ORDER BY 键，version 更大

-- 此时查询：可能返回两条（合并还没发生）
SELECT * FROM dedup_test;
-- 结果可能是两条，也可能是合并后的一条（不确定）

-- 正确做法：用 GROUP BY 或 FINAL 关键字
SELECT * FROM dedup_test FINAL;  -- FINAL 会强制等待合并完成
```

**原因：** ClickHouse 的 Merge 操作在后台异步执行，无法实时保证去重。

### Q3：Kafka → ClickHouse 的实时数据管道怎么设计？

```
方案 1（简单）：Kafka → clickhouse-go → ClickHouse
  缺点：延迟高（每个事件一次写入）

方案 2（推荐）：Kafka → Flink/Spark Streaming → ClickHouse
  优点：批量写入，吞吐量大，exactly-once 语义

方案 3（最高实时）：Kafka → MaterializedMySQL → ClickHouse
  优点：MySQL 变更实时同步到 ClickHouse

具体 Kafka + ClickHouse 写入代码：
  consumer := kafka.NewConsumer(...)  // 消费 Kafka
  batch, _ := clickhouse.PrepareBatch("INSERT INTO events")

  for {
      msg, _ := consumer.ReadMessage()  // 批量消费
      batch.Append(msg.UserID, msg.Event, msg.TS)
      if batch.RowCount() >= 1000 {
          batch.Send()  // 每 1000 条批量写入
      }
  }
```
