# 分库分表：ShardingSphere 与分布式路由策略

## 面试官考察意图

分库分表是后端工程师面试区分度很高的考点。
初级只知道"数据量大就分表"，高级要能讲清楚 **分片键选择策略、分布式 ID 生成、跨分片查询解决方案、数据迁移方案**，并结合 ShardingSphere 等工具给出生产级实现。

---

## 核心答案（30 秒版）

分库分表解决的是**单机数据库容量和性能瓶颈**。

```
问题：单表超过 2000 万行、QPS 超过 1 万
解决：
  - 垂直拆分：按业务模块分库（订单库、用户库）
  - 水平拆分：按分片键分表（user_id % 4 拆成 4 张表）
  - 中间件：ShardingSphere、Vitess、MyCat

核心挑战：跨分片查询、分片键选择、分布式 ID、数据迁移
```

---

## 深度展开

### 1. 何时需要分库分表

| 指标 | 单机上限 | 信号 |
|------|----------|------|
| 数据量 | 2000~5000 万行（InnoDB） | 索引树高 > 3，查询变慢 |
| QPS | 5000~10000（普通服务器） | CPU 打满、连接数不足 |
| 磁盘 | 500GB~1TB | 存储接近上限 |
| 内存 | 64GB | 热点数据无法全部缓存 |

**分库分表顺序**：先考虑**缓存 + 读写分离**，再考虑**分库分表**（成本高、复杂度剧增）。

### 2. 分片策略

#### 2.1 按主键 ID 范围（Range）

```sql
-- 按 ID 范围分表
-- orders_0: id 1~500万
-- orders_1: id 500万~1000万
-- orders_2: id 1000万~1500万

-- 优点：新增表成本低，数据分布均匀
-- 缺点：热点数据集中在最新表，压力不均（写热点）
```

#### 2.2 按分片键 Hash（最常用）

```go
// Go 分片路由实现
func getTableName(userID int64, tableCount int) string {
    shardIndex := userID % tableCount
    return fmt.Sprintf("orders_%d", shardIndex)
}

// ShardingSphere YAML 配置示例
// sharding.yaml
// tables:
//   orders:
//     actualDataNodes: ds_${0..1}.orders_${0..3}
//     databaseStrategy:
//       standard:
//         shardingColumn: user_id
//         shardingAlgorithmName: user_id_mod
//     tableStrategy:
//       standard:
//         shardingColumn: order_id
//         shardingAlgorithmName: order_id_mod
```

#### 2.3 按时间分片

```sql
-- 按月分表：orders_202601, orders_202602
-- 适合日志、流水、消息等时间序列数据
-- 优点：自然过期清理，支持范围查询
-- 缺点：历史数据查询需跨多表

-- 热点问题：当前月份表压力集中
-- 解决：加预写表 + 异步归并
```

#### 2.4 复合分片（多分片键）

```go
// 复合分片：user_id + order_type
// sharding_key = hash(user_id) ^ hash(order_type) % table_count
func compositeSharding(userID int64, orderType string, tableCount int) int {
    h1 := hashCode(userID)
    h2 := hashCode(orderType)
    return (h1 ^ h2) & (tableCount - 1)  // tableCount 必须是 2 的幂次
}
```

### 3. 分片键选择原则

**好的分片键特征：**
- 查询频率高（80%+ 查询带此条件）
- 分布均匀（避免热点）
- 不频繁变更（变更需要数据迁移）

**常见错误分片键：**
| 分片键 | 问题 |
|--------|------|
| UUID | 完全随机，无法做范围查询 |
| 时间戳 | 热点集中在当前数据 |
| 性别 | 只有两个值，无法分片 |
| 状态（待支付/已支付） | 数据倾斜严重 |

```sql
-- 典型场景：订单系统
-- ✅ 好分片键：user_id（查询按用户查订单）
-- ❌ 坏分片键：status（按状态筛选极度不均）

-- 如果必须按状态查：加 ES / 单独建索引库
```

### 4. 分布式 ID 生成方案

分表后不能用自增主键，需要分布式 ID。

#### 方案对比

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| UUID | 随机 128 位 | 简单，不依赖第三方 | 无序、存储大、索引差 |
| 数据库自增 | 独立 ID 表 | 简单，连续 | 单点瓶颈，跨库不连续 |
| Snowflake | 时间戳 + 机器号 + 序列号 | 高性能，有序 | 依赖时钟（时钟回拨问题）|
| 百度 UidGenerator | 优化 Snowflake，减少时间依赖 | 高性能 | 实现复杂 |
| 滴滴 Leaf | Snowflake + 号段模式 | 双模式，高可用 | 依赖 ZooKeeper |
| Segment（号段模式） | 批量从数据库拿 ID 段 | 本地生成，极快 | ID 区间有空洞 |

```go
// Snowflake Go 实现
type Snowflake struct {
    mu        sync.Mutex
    timestamp int64
    machineID int64
    sequence  int64
}

const (
    twepoch     = int64(1609459200000) // 2021-01-01
    machineBits = 5
    seqBits     = 12
)

func (s *Snowflake) NextID() int64 {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    now := time.Now().UnixNano()/1000000 - twepoch
    
    if now == s.timestamp {
        s.sequence = (s.sequence + 1) & ((1 << seqBits) - 1)
        if s.sequence == 0 {
            for now <= s.timestamp {
                now = time.Now().UnixNano()/1000000 - twepoch
            }
        }
    } else {
        s.sequence = 0
    }
    s.timestamp = now
    
    return (now << (machineBits + seqBits)) |
           (s.machineID << seqBits) |
           s.sequence
}
```

### 5. 跨分片查询解决方案

#### 5.1 聚合函数（COUNT/SUM/AVG）

```go
// 并行查询所有分片，结果在应用层聚合
func countOrdersByUser(userID int64, tables []string) (int64, error) {
    var total int64
    var mu sync.Mutex
    var wg sync.WaitGroup
    errChan := make(chan error, len(tables))
    
    for _, tbl := range tables {
        wg.Add(1)
        go func(t string) {
            defer wg.Done()
            var cnt int64
            err := db.QueryRowContext(ctx, "SELECT COUNT(*) FROM "+t+" WHERE user_id = ?", userID).Scan(&cnt)
            if err != nil {
                errChan <- err
                return
            }
            mu.Lock()
            total += cnt
            mu.Unlock()
        }(tbl)
    }
    wg.Wait()
    close(errChan)
    
    for err := range errChan {
        return 0, err
    }
    return total, nil
}
```

#### 5.2 分页查询

```go
// 跨分片深度分页：禁止使用 OFFSET
// 正确方案：游标分页（基于 ID）

// ❌ 错误：SELECT * FROM orders ORDER BY id LIMIT 1000000, 10
// 每个分片都要扫描 1000010 行

// ✅ 正确：先查 ID，再拿数据
func paginateOrders(userID int64, lastID int64, pageSize int) ([]Order, error) {
    // 1. 从所有分片并行获取下一页的 ID 列表
    var allIDs []int64
    for _, tbl := range tables {
        rows, err := db.QueryContext(ctx,
            "SELECT id FROM "+tbl+" WHERE user_id = ? AND id > ? ORDER BY id LIMIT ?",
            userID, lastID, pageSize)
        if err != nil {
            return nil, err
        }
        for rows.Next() {
            var id int64
            rows.Scan(&id)
            allIDs = append(allIDs, id)
        }
        rows.Close()
    }
    
    // 2. 排序并取前 pageSize 个
    sort.Slice(allIDs, func(i, j int) bool { return allIDs[i] < allIDs[j] })
    if len(allIDs) > pageSize {
        allIDs = allIDs[:pageSize]
    }
    
    // 3. 根据 ID 列表查完整数据（IN 查询，注意分片限制）
    return getOrdersByIDs(ctx, allIDs)
}
```

#### 5.3 排序查询

```go
// 全局排序（按金额）：需要从所有分片取数据，在应用层排序
// ⚠️ 性能差，数据量大时不可行
// 解决：把排序字段冗余到 ES，用 ES 做全局排序

type Order struct { Amount float64 } // 排序字段

func globalSort(tables []string, limit int) ([]Order, error) {
    all := []Order{}
    for _, tbl := range tables {
        rows, _ := db.QueryContext(ctx, "SELECT amount FROM "+tbl+" ORDER BY amount DESC LIMIT 1000")
        for rows.Next() {
            var o Order
            rows.Scan(&o.Amount)
            all = append(all, o)
        }
    }
    sort.Slice(all, func(i, j int) bool { return all[i].Amount > all[j].Amount })
    if len(all) > limit {
        all = all[:limit]
    }
    return all, nil
}
```

### 6. ShardingSphere 实战配置

```yaml
# ShardingSphere 分库分表配置（5.0+ 版本）
schemaName: sharding_db

dataSources:
  ds_0:
    url: jdbc:mysql://localhost:3306/ds_0?serverTimezone=UTC
    username: root
    password: 
    connectionPoolClassName: com.zaxxer.hikari.HikariDataSource
  ds_1:
    url: jdbc:mysql://localhost:3306/ds_1?serverTimezone=UTC
    username: root
    password: 

rules:
  - !SHARDING
    tables:
      orders:
        actualDataNodes: ds_${0..1}.orders_${0..3}
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: user_id_database_mod
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: order_id_table_mod
        keyGenerateStrategy:
          column: order_id
          keyGeneratorName: snowflake
    
    shardingAlgorithms:
      user_id_database_mod:
        type: INLINE
        props:
          algorithm-expression: ds_${user_id % 2}
      order_id_table_mod:
        type: INLINE
        props:
          algorithm-expression: orders_${order_id % 4}
    
    keyGenerators:
      snowflake:
        type: SNOWFLAKE
        props:
          worker-id: 1
```

```go
// Go 连接 ShardingSphere Proxy
import (
    "github.com/go-sql-driver/mysql"
)

func init() {
    // ShardingSphere Proxy 3307 端口
    dsn := "root:@tcp(localhost:3307)/sharding_db?parseTime=true"
    db, _ = sql.Open("mysql", dsn)
    db.SetMaxOpenConns(100)
}
```

### 7. 数据迁移方案

#### 阶段一：双写（旧库 + 新库）

```go
// 应用层双写：写旧库，同时写新库
func createOrder(order *Order) error {
    // 1. 写旧库
    if err := writeOldDB(order); err != nil {
        return err
    }
    
    // 2. 写新库（分片表）
    shard := getTableName(order.UserID, 4)
    if err := writeNewDB(shard, order); err != nil {
        // 异步重试，避免阻塞主流程
        go func() {
            retryWriteToNewDB(order, shard)
        }()
    }
    
    return nil
}
```

#### 阶段二：数据同步（binlog 增量订阅）

```go
// 使用 canal / Debezium 订阅 binlog，增量同步到分库分表
// 典型工具：
// - canal（阿里）：解析 MySQL binlog
// - Debezium（Red Hat）：支持 MySQL/PostgreSQL/MongoDB
// - Maxwell：轻量级 binlog 订阅
// - Canal Adapter：同步到 Kafka/ES/HBase

// 同步流程：
// MySQL → Canal → Kafka → Sync Service → ShardingDB
```

#### 阶段三：数据校验与切读

```go
// 数据校验：抽样比对
func verifyMigration(oldDB, newDB *sql.DB, sampleRate float64) error {
    rows, _ := oldDB.QueryContext(ctx, 
        "SELECT COUNT(*) FROM orders WHERE created_at > '迁移时间点'")
    var total int64
    rows.Scan(&total)
    
    sampleSize := int64(float64(total) * sampleRate)
    // 随机抽样校验
    
    // 切读：先读新库，降级读旧库
    // 通过配置中心动态切换读来源
    return nil
}
```

#### 阶段四：灰度切读

```
步骤：
① 5% 流量读新库 → 监控错误率
② 20% 流量读新库 → 继续监控
③ 50% 流量读新库 → 核心指标对比
④ 100% 切读 → 关闭旧库
```

### 8. 分库分表后的限制

| 限制 | 说明 | 解决方案 |
|------|------|----------|
| 跨分片 JOIN | 无法直接 JOIN | 业务层拆解，或冗余字段，或 ES |
| 事务 | 无法跨库事务 | 分布式事务（TCC/Saga），或最终一致性 |
| 自增主键 | 跨库不连续 | 分布式 ID（Snowflake/号段） |
| 全局唯一索引 | 无法建跨库唯一索引 | 业务保证，或全局唯一 ID |
| 深分页 | 跨分片性能差 | 游标分页，禁止 OFFSET |

---

## 面试追问

**Q：分库分表后如何做分页查询？**
> 禁止使用 OFFSET，深分页必须用游标（基于 ID）。先从各分片并行查 ID，排序后取目标页，再查完整数据。数据量极大时考虑 ES。

**Q：分片键选错了怎么办？**
> 如果是热点分片键导致数据不均，可以做**二次分片**（加一层路由层）；如果是分片键变更，那只能做数据迁移，成本很高，所以分片键选择一定要慎重。

**Q：什么场景不适合分库分表？**
> 数据量没到瓶颈（<2000万）、QPS 不是瓶颈（<5000）、可以通过缓存解决读性能问题的场景。分库分表引入的复杂度是指数级的，能不分就不分。

---

## 相关知识点

- 索引原理 → `01-index.md`
- 慢查询优化 → `04-slow-query.md`
- 分布式事务 → `../../03-distributed/02-transactions/`
- 主从复制 → `06-replication.md`
