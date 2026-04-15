# 02 · 数据库

> 考察频率：★★★★★  优先级：P0

## 文章清单

### 01-mysql · MySQL
- [x] `01-index.md` — 索引结构（B+ 树）、聚簇索引 vs 二级索引、索引选择策略
- [x] `02-transaction.md` — ACID、隔离级别、MVCC 实现原理
- [✅] `03-lock.md` — 行锁/表锁/间隙锁/临键锁、死锁检测与处理
- [x] `04-slow-query.md` — 慢查询优化：EXPLAIN 解读、索引失效场景
- [x] `05-sharding.md` — 分库分表方案、ShardingSphere、数据迁移
- [✅] `06-replication.md` — 主从复制原理、binlog、半同步复制、延迟处理

### 02-redis · Redis
- [✅] `01-data-structures.md` — 5 种基本类型 + 3 种高级类型底层实现
- [✅] `02-persistence.md` — RDB vs AOF、混合持久化、数据恢复
- [✅] `03-cache-problems.md` — 缓存穿透/击穿/雪崩：原理、方案、代码实现
- [✅] `04-cluster.md` — Sentinel vs Cluster、槽位分配、故障转移
- [✅] `05-distributed-lock.md` — Redlock 算法、单机锁、Lua 脚本原子性
- [✅] `06-hot-key.md` — 热 key 识别、大 key 处理、本地缓存方案

### 03-elasticsearch · Elasticsearch（可选）
- [✅] `01-inverted-index.md` — 倒排索引原理、分词、相关性评分
- [🟡] `02-query-optimization.md` — MySQL 查询优化：EXPLAIN、慢查询、索引失效场景
