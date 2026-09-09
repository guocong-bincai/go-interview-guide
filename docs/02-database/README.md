# 02 · 数据库

> 考察频率：★★★★★  优先级：P0

## 文章清单

### 01-mysql · MySQL
- [x] [索引结构（B+ 树）、聚簇索引 vs 二级索引、索引选择策略](01-mysql/10-01-index.md)
- [x] [ACID、隔离级别、MVCC 实现原理](01-mysql/11-02-transaction.md)
- [✅] [行锁/表锁/间隙锁/临键锁、死锁检测与处理](01-mysql/06-03-lock.md)
- [x] [慢查询优化：EXPLAIN 解读、索引失效场景](01-mysql/12-04-slow-query.md)
- [x] [分库分表方案、ShardingSphere、数据迁移](01-mysql/13-05-sharding.md)
- [✅] [主从复制原理、binlog、半同步复制、延迟处理](01-mysql/07-06-replication.md)
- [✅] [MySQL 三大日志：binlog / redo log / undo log 与两阶段提交（crash-safe）](01-mysql/26-01-mysql-logs.md)
- [✅] [一条 SQL 语句的完整执行流程：连接器→分析器→优化器→执行器](01-mysql/27-02-sql-execution.md)
- [✅] [InnoDB Buffer Pool：三大链表、LRU 冷热分区、脏页刷盘与调优](01-mysql/28-03-buffer-pool.md)
- [✅] [MySQL 主键选择：自增 vs UUID vs 雪花 ID（聚簇索引视角）](01-mysql/29-04-primary-key.md)
- [ ] [AUTO_INCREMENT 高并发锁竞争与 innodb_autoinc_lock_mode 优化方案](01-mysql/32-05-auto-increment-lock.md)
- [ ] [布隆过滤器（Bloom Filter）防缓存穿透原理与 Go 实现](01-mysql/33-06-bloom-filter.md)
- [ ] [读写分离延迟补偿：Session Stickiness + 强制主库读取](01-mysql/34-07-read-write-separation.md)
- [ ] [乐观锁 CAS vs 悲观锁 FOR UPDATE：Go 实战选型决策树](01-mysql/35-08-optimistic-pessimistic-lock.md)

### 02-redis · Redis
- [✅] [5 种基本类型 + 3 种高级类型底层实现](02-redis/01-01-data-structures.md)
- [✅] [RDB vs AOF、混合持久化、数据恢复](02-redis/18-02-persistence.md)
- [✅] [缓存穿透/击穿/雪崩：原理、方案、代码实现](02-redis/02-03-cache-problems.md)
- [✅] [Sentinel vs Cluster、槽位分配、故障转移](02-redis/19-04-cluster.md)
- [✅] [Redlock 算法、单机锁、Lua 脚本原子性](02-redis/03-05-distributed-lock.md)
- [✅] [热 key 识别、大 key 处理、本地缓存方案](02-redis/20-06-hot-key.md)
- [✅] [主从复制原理：psync 全量同步/增量同步、repl_backlog、复制偏移量](02-redis/30-05-replication.md)
- [✅] [内存碎片管理 / MULTI 回滚 / PubSub vs Stream / Pipeline 性能优化](02-redis/31-06-memory-mgmt.md)
- [ ] [缓存与数据库双写一致性：Cache Aside / Delayed Dual Delete / Canal + Binlog](02-redis/32-07-cache-db-consistency.md)
- [ ] [Redis 数据结构选型指南：String vs Hash vs List vs Set vs ZSet](02-redis/33-08-data-structure-selection.md)

### 03-elasticsearch · Elasticsearch（可选）
- [✅] [倒排索引原理、分词、相关性评分](03-elasticsearch/22-01-inverted-index.md)
- [✅] [MySQL 查询优化：EXPLAIN、慢查询、索引失效场景](01-mysql/08-08-query-optimization.md)
