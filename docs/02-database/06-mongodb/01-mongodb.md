# MongoDB 核心原理与 Go 实战

> 考察频率：★★★☆☆  优先级：P1

## TODO（待填写）

## 1. 文档模型 vs 关系型
- [ ] BSON 格式：二进制 JSON，支持额外类型（ObjectId/Date/Binary）
- [ ] 嵌套文档 vs 关联表：什么时候内嵌，什么时候引用（范式 vs 反范式）
- [ ] Collection 无 Schema 的优势和风险

## 2. 索引
- [ ] 单字段索引、复合索引（ESR 规则：Equality → Sort → Range）
- [ ] 多键索引（数组字段）、地理空间索引（2dsphere）、文本索引
- [ ] `explain()` 分析执行计划

## 3. 聚合管道（Aggregation Pipeline）
- [ ] `$match` → `$group` → `$sort` → `$project` → `$lookup`
- [ ] 与 MySQL GROUP BY + JOIN 的对比
- [ ] 完整 Go 代码示例：统计订单金额

## 4. WiredTiger 存储引擎
- [ ] MVCC 多版本并发控制（文档级锁，非表锁）
- [ ] CheckPoint 机制（60s 或 2GB Journal）
- [ ] 压缩算法（Snappy/zstd）

## 5. 副本集与分片集群
- [ ] 副本集：Primary + Secondary + Arbiter，选举机制（Raft 变体）
- [ ] 分片集群：mongos → Config Server → Shard
- [ ] 分片键选择原则（高基数、写均匀、查询覆盖）

## 6. 适用场景判断
- [ ] 选 MongoDB：文档结构灵活、嵌套数据多、快速迭代产品
- [ ] 不选 MongoDB：强事务（跨文档多表）、复杂 JOIN 查询
- [ ] MongoDB 4.x 多文档事务与 MySQL 事务对比

## 7. Go 实战（mongo-driver）
- [ ] 连接池配置（`ClientOptions`）
- [ ] CRUD 操作代码示例
- [ ] Context 超时控制

## 高频追问
- [ ] MongoDB 的事务和 MySQL 的事务有什么区别？
- [ ] 为什么 MongoDB 默认不支持 JOIN，而是用嵌套文档代替？
- [ ] MongoDB 的 ObjectId 是怎么保证全局唯一的？
