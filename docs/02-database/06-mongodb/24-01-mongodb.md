# MongoDB 核心原理与 Go 实战

> 考察频率：★★★☆☆  优先级：P1
> 关键词：BSON、WiredTiger、副本集、分片、MongoDB 事务、聚合管道

## 为什么选择 MongoDB

MongoDB 是面向文档的 NoSQL 数据库，用**JSON 格式（BSON）存储数据**，天然适合：
- 数据结构频繁变化（无 Schema 约束）
- 嵌套结构多（如订单包含商品列表）
- 需要快速迭代的产品初期

---

## 1. 文档模型 vs 关系型

### BSON 格式

BSON（Binary JSON）是 MongoDB 的二进制序列化格式，相比 JSON 多了几种高效类型：

| BSON 类型 | 说明 | 示例 |
|-----------|------|------|
| `ObjectId` | 12 字节全局唯一 ID | `ObjectId("507f1f77bcf86cd799439011")` |
| `Date` | 64 位有符号整数，毫秒时间戳 | `ISODate("2026-05-06T00:00:00Z")` |
| `Decimal128` | 高精度金融计算 | `Decimal128("123.45")` |
| `Binary` | 二进制数据 | 图片、文件 |
| `DBRef` | 跨文档引用 | 类似外键 |

```go
// Go 中的 BSON 文档
type Order struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    UserID    int64              `bson:"user_id"`
    Items     []OrderItem        `bson:"items"`      // 嵌套文档
    Status    string             `bson:"status"`
    CreatedAt time.Time          `bson:"created_at"`
}

type OrderItem struct {
    ProductID int64   `bson:"product_id"`
    Name      string  `bson:"name"`
    Price     float64 `bson:"price"`
    Quantity  int     `bson:"quantity"`
}
```

### 嵌套文档 vs 关联表：什么时候选哪个

**内嵌文档（Denormalization，反范式）** 适用：
- 数据总是一起读取（1:1 或 1:few）
- 数据不需要单独更新
- 嵌套层级 ≤ 2 层

```go
// ✅ 内嵌：用户文档中包含地址数组（一般用户地址不会太多）
{
  "_id": ObjectId("..."),
  "name": "张三",
  "addresses": [
    { "city": "北京", "detail": "朝阳区xxx" },
    { "city": "上海", "detail": "浦东新区xxx" }
  ]
}
```

**引用文档（Normalization，正规化）** 适用：
- 数据需要独立更新（如商品信息变更要同步到所有订单）
- 一对多且"多"的数量不确定
- 需要做关联聚合查询

```go
// ✅ 引用：订单引用用户 ID，而不是复制用户信息
{
  "_id": ObjectId("..."),
  "user_id": 12345,       // 引用，而非内嵌
  "items": [...],
  "total": 299.00
}

// 查询时用 $lookup 做关联
db.orders.aggregate([
  { $match: { user_id: 12345 } },
  { $lookup: {
      from: "users",
      localField: "user_id",
      foreignField: "_id",
      as: "user_info"
  }}
])
```

### MongoDB 无 Schema 的优势与风险

**优势：**
- 快速迭代：新字段无需 `ALTER TABLE`
- 灵活存储：不同文档可以有不同字段

**风险：**
- 数据质量难以保证（缺少约束）
- 查询结果不稳定（字段可能不存在）
- 索引维护困难

---

## 2. 索引

### 单字段索引与复合索引

```go
// 单字段索引
db.users.createIndex({ "age": 1 })

// 复合索引（ESR 规则：Equality → Sort → Range）
// 先等值查询，再排序，最后范围查询
db.orders.createIndex({ "user_id": 1, "created_at": -1, "status": 1 })
// 最佳查询顺序：
// 1. { user_id: 123 }        → 等值
// 2. { user_id: 123, created_at: { $gte: ... } }  → 排序
// 3. { user_id: 123, created_at: { $gte: ... }, status: "paid" } → 范围
```

### ESR 规则详解

复合索引字段顺序决定查询能否命中索引：

| 查询条件 | 能否使用索引 | 原因 |
|---------|------------|------|
| `{a: 1, b: 1, c: 1}` | ✅ 全用 | 完全顺序匹配 |
| `{a: 1, c: 1}` | ✅ 用 a | 跳过 b 不影响 a 生效 |
| `{b: 1, c: 1}` | ❌ 用不上 | b 不是最左前缀 |
| `{a: 1, b: { $gt: 5 }}` | ✅ 用 a | b 是范围但不影响 a |

### 多键索引（数组字段）

```go
// 对数组字段建索引，数组中每个元素都建立索引项
db.orders.createIndex({ "items.product_id": 1 })

// 查询订单中包含某商品的所有订单
db.orders.find({ "items.product_id": 10086 })
```

### 地理空间索引

```go
// 2dsphere 索引：用于经纬度查询
db.stores.createIndex({ "location": "2dsphere" })

// 查询附近 5km 内的商家（按距离排序）
db.stores.find({
  "location": {
    $nearSphere: {
      $geometry: { type: "Point", coordinates: [116.397, 39.908] },
      $maxDistance: 5000  // 米
    }
  }
}).limit(10)
```

### `explain()` 分析执行计划

```go
// 查看查询计划
db.orders.find({ "user_id": 12345, "status": "paid" }).explain("executionStats")

// 关键字段解读：
// "stage": "IXSCAN" → 使用了索引扫描（好）
// "stage": "COLLSCAN" → 全表扫描（需要优化）
// "nReturned": 5     → 返回了 5 条文档
// "totalDocsExamined": 100000 → 扫描了 10 万条（效率低）
// "winningPlan.indexName": "user_id_1_status_1" → 命中哪个索引
```

---

## 3. 聚合管道（Aggregation Pipeline）

聚合管道是 MongoDB 最强大的查询工具，类似函数式编程的流水线。

```go
// MongoDB Go 驱动聚合示例
pipeline := mongo.Pipeline{
    {{Key: "$match", Value: bson.D{
        {Key: "created_at", Value: bson.D{{Key: "$gte", Value: startDate}}},
        {Key: "status", Value: "paid"},
    }}},
    {{Key: "$unwind", Value: "$items"}},                              // 展开订单明细
    {{Key: "$group", Value: bson.D{                                   // 按商品分组
        {Key: "_id", Value: "$items.product_id"},
        {Key: "total_amount", Value: bson.D{{Key: "$sum", Value: bson.D{
            {Key: "$multiply", Value: []interface{}{"$items.price", "$items.quantity"}},
        }}}},
        {Key: "order_count", Value: bson.D{{Key: "$sum", Value: 1}}},
    }}},
    {{Key: "$sort", Value: bson.D{{Key: "total_amount", Value: -1}}}}, // 按金额降序
    {{Key: "$limit", Value: 10}},                                       // 取前 10
}}

cursor, err := coll.Aggregate(ctx, pipeline)
```

### 与 MySQL 对比

| MongoDB 聚合 | MySQL 等价 |
|-------------|-----------|
| `$match` | `WHERE` |
| `$group` | `GROUP BY` |
| `$sort` | `ORDER BY` |
| `$project` | `SELECT col1, col2` |
| `$limit` | `LIMIT` |
| `$unwind` | （无直接等价，需要JOIN/子查询模拟） |
| `$lookup` | `LEFT JOIN` |

---

## 4. WiredTiger 存储引擎

### MVCC 多版本并发控制

WiredTiger 使用**文档级锁**（而非 MySQL 的行锁/表锁），并发性能更高：

```go
// 写操作：获取文档级别写锁（其他写操作可并发修改不同文档）
db.orders.updateOne(
    { "_id": ObjectId("...") },  // 只需锁定这一篇文档
    { "$set": { "status": "paid" } }
)
```

### CheckPoint 机制

```
写入路径：
WiredTiger Write Batch → Journal（预写日志，持久化）→ CheckPoint（定期将内存数据刷到磁盘）

CheckPoint 触发条件（满足任一）：
1. 距离上次 CheckPoint 已过 60 秒
2. Journal 文件达到 2GB
3. 执行 fsync 或关闭数据库
```

### 压缩算法

```go
// 创建集合时指定压缩算法
db.createCollection("logs", {
    storageEngine: {
        wiredTiger: {
            configString: "block_compressor=zstd"  // zstd / snappy / zlib
        }
    }
})
```

---

## 5. 副本集与分片集群

### 副本集（Replica Set）

```
副本集拓扑：
┌─────────────────────────────────────────┐
│           Primary（主节点）              │  ← 接收所有写操作
│   "votes":1, "priority":5, "hidden":false│
├─────────────────────────────────────────┤
│  Secondary-1    Secondary-2   Arbiter   │
│  votes:1        votes:1       votes:1   │
│  priority:3     priority:3   priority:0│  ← Arbiter 不存数据，只投票
└─────────────────────────────────────────┘

选举机制（Raft 变体）：
1. 心跳超时（默认 10s）→ Secondary 认为 Primary 宕机
2. 满足 electionTimeoutMin ~ electionTimeoutMax 随机等待
3. 发起选举：有投票权的节点中获多数票者成为新 Primary
```

**Go 读取副本集数据：**

```go
// 设置读写分离读（从 Secondary 读，分担 Primary 压力）
opts := options.Client().ApplyURI("mongodb://host1:27017,host2:27017/?replicaSet=rs0")
opts.SetReadPreference(readpref.Secondary())

client, err := mongo.Connect(ctx, opts)
coll := client.Database("shop").Collection("orders")

// 读偏好模式：
// - primary（默认）：始终读 Primary
// - secondaryPreferred：优先 Secondary，必要时读 Primary
// - nearest：读延迟最低的节点
```

### 分片集群（Sharded Cluster）

```
分片集群架构：
┌──────────┐
│  mongos  │  ← 路由节点，不存储数据，根据配置计算路由
│ (路由层) │
└────┬─────┘
     │
┌────┴────┬──────────┐
▼         ▼          ▼
┌────────┐┌────────┐┌────────┐
│ Shard1 ││ Shard2 ││ Shard3 │  ← 分片节点，存储实际数据
│(0~33%) ││(34~66%)││(67~100%)│
└────────┘└────────┘└────────┘
     ↑
┌─────────────────────────────────┐
│       Config Server（配置节点）  │  ← 存储集群元数据（分片信息）
│   副本集模式（3节点）            │
└─────────────────────────────────┘
```

### 分片键选择原则

```go
// ❌ 错误分片键示例
{ "_id": "ObjectId" }  // ObjectId 递增 → 所有新写入都路由到最后一个分片（热分片）

// ✅ 正确分片键示例
{ "user_id": "hashed" }  // 哈希分片：写入均匀分布
{ "region_id": 1, "created_at": 1 }  // 范围分片：适合按时间范围查询
```

**分片键三原则：**
1. **高基数**：字段值种类要多（如 user_id，不要选性别）
2. **写均匀**：避免写入集中在某几个分片（哈希分片 > 范围分片）
3. **查询覆盖**：常用查询条件包含分片键，避免广播查询（scatter-gather）

---

## 6. MongoDB 4.x+ 多文档事务

### 事务 API

```go
// Go 中使用多文档事务
session, err := client.StartSession()
defer session.EndSession(ctx)

_, err = session.WithTransaction(ctx, func(sessCtx mongo.SessionContext) (interface{}, error) {
    // 事务内操作：扣库存
    _, err = collInventory.UpdateOne(sessCtx,
        bson.M{"product_id": 10086, "stock": bson.M{"$gte": 1}},
        bson.M{"$inc": bson.M{"stock": -1}},
    )
    if err != nil {
        return nil, err  // 事务自动 abort
    }

    // 事务内操作：创建订单
    _, err = collOrders.InsertOne(sessCtx, order)
    return nil, err
})
```

### MongoDB 事务 vs MySQL 事务

| 对比维度 | MongoDB 多文档事务 | MySQL 事务 |
|---------|------------------|-----------|
| 隔离级别 | 可重复读（Snapshot）| 可重复读（InnoDB）|
| 锁粒度 | 文档级 | 行级 |
| 回滚方式 | 写冲突时 abort | 任意时刻可 rollback |
| 性能影响 | 较大（跨分片事务更重）| 较小 |
| 跨集合能力 | ✅ 支持 | ✅ 支持（JOIN 跨表）|
| 跨分片能力 | ⚠️ 慢（需 mongos 协调）| N/A |

---

## 7. Go 实战（mongo-go-driver）

### 连接池配置

```go
import (
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
    "go.mongodb.org/mongo-driver/mongo/readpref"
)

func newMongoClient(ctx context.Context) (*mongo.Client, error) {
    opts := options.Client().
        ApplyURI("mongodb://localhost:27017").
        SetMaxPoolSize(100).              // 最大连接数
        SetMinPoolSize(10).               // 最小连接数
        SetMaxConnIdleTime(5 * time.Minute). // 空闲连接超时
        SetServerSelectionTimeout(3 * time.Second).
        SetConnectTimeout(10 * time.Second)

    client, err := mongo.Connect(ctx, opts)
    if err != nil {
        return nil, fmt.Errorf("mongo connect failed: %w", err)
    }

    // 验证连接
    if err := client.Ping(ctx, readpref.Primary()); err != nil {
        return nil, fmt.Errorf("mongo ping failed: %w", err)
    }
    return client, nil
}
```

### CRUD 操作示例

```go
// 插入
res, err := coll.InsertOne(ctx, Order{
    UserID:    12345,
    Items:     items,
    Status:    "pending",
    CreatedAt: time.Now(),
})

// 查询
var order Order
err := coll.FindOne(ctx, bson.M{"_id": res.InsertedID}).Decode(&order)

// 更新（原子操作：库存充足才扣减）
result, err := coll.UpdateOne(ctx,
    bson.M{"_id": orderID, "items.product_id": pid, "items.stock": bson.M{"$gte": qty}},
    bson.M{"$inc": bson.M{"items.$.stock": -qty}},
)

// 删除
_, err = coll.DeleteOne(ctx, bson.M{"_id": orderID})
```

### Context 超时控制

```go
// 所有操作都支持 context 超时控制
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// 超时后自动关闭游标和连接
cursor, err := coll.Find(ctx, filter)
```

---

## 高频追问

### Q1：MongoDB 的事务和 MySQL 的事务有什么区别？

**核心区别在于锁粒度和一致性模型：**

- **MongoDB** 使用文档级锁（乐观并发），事务默认在快照隔离级别运行，冲突时 abort 并重试；而 **MySQL InnoDB** 使用行锁（悲观并发），支持 `SELECT ... FOR UPDATE` 强制加锁。
- MongoDB 事务是**两阶段提交**（2PC）变体，跨分片事务需要 mongos 协调，性能损耗显著；MySQL 单节点事务是单阶段提交，开销小得多。
- **实战建议**：能用单文档原子操作解决的，不用多文档事务（如订单+库存扣减，可用 `FindOneAndUpdate` 原子更新）。

### Q2：为什么 MongoDB 默认不支持 JOIN，而是用嵌套文档代替？

MongoDB 设计哲学是**数据就近存储**，减少跨网络请求：
- 嵌套文档读取 1 次网络 IO，JOIN 读取多次（`$lookup` 本质也是 JOIN）
- 但嵌套文档会导致数据重复，更新时要同步多处
- MongoDB 不是"不能 JOIN"，而是**默认不推荐**：性能损耗大，且跨集合 JOIN 无法利用索引

### Q3：MongoDB 的 ObjectId 是怎么保证全局唯一的？

ObjectId 是 12 字节，结构如下：

```
| 4 字节（时间戳） | 5 字节（随机数） | 3 字节（递增计数器） |
    秒级时间戳           机器 ID+PID         同一个进程内每秒可生成
    保证唯一             保证同机器内唯一      1677万+个 ObjectId
```

- **时间戳**：同一秒内不同机器可能冲突（但时间不同则不冲突）
- **随机数**：同机器不同进程冲突概率极低（5 字节 = 2^40）
- **计数器**：同进程内保证严格递增，绝对不冲突

**结论**：全局唯一由"时间戳 + 机器指纹 + 计数器"三层保证，冲突概率约 1/2^64。
