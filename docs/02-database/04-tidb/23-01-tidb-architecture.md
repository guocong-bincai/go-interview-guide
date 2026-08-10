# TiDB 架构与适用场景

> 考察频率：★★★☆☆  优先级：P2
> 关键词：HTAP、Percolator、TiKV、分布式事务、OLTP/OLAP

## 面试官考察意图

这道题考察候选人对**分布式数据库**的系统性认知，尤其是 MySQL 分库分表之外的另一条路。高级工程师会关注：
- TiDB 不是简单地把 MySQL 水平扩展，它的分布式事务模型（Percolator）和 MySQL 完全不同
- HTAP 是什么，为什么 TiFlash 列存引擎能同时支撑 OLTP 和 OLAP
- 什么时候选 TiDB，什么时候继续用 MySQL 分库分表（选错代价很大）
- 全局自增 ID 的实现原理（分布式 ID 生成）

初级工程师往往只知道"TiDB 就是 MySQL 兼容的分布式数据库"。

---

## 核心答案（30 秒版）

**TiDB = TiDB Server（SQL 层）+ TiKV（行存引擎）+ TiFlash（列存引擎）+ PD（调度器）。**

一句话总结：
- **水平扩展**：计算层和存储层分离，加节点就能扩容
- **HTAP**：行存 TiKV 支撑 OLTP，列存 TiFlash 支撑 OLAP，一套数据库两个场景
- **分布式事务**：Percolator 协议（2PC 变体），乐观锁 + timestampOracle
- **MySQL 兼容**：协议层面兼容，大多数业务无需修改代码

---

## 深度展开

### 1. 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                          TiDB 集群                              │
│                                                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  TiDB       │  │  TiDB       │  │  TiDB       │  SQL 层     │
│  │  Server 1   │  │  Server 2   │  │  Server 3   │  (无状态)    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          ▼                                      │
│              ┌───────────────────────┐                         │
│              │   PD (Placement Driver)│                        │
│              │  · 分配时间戳 (TSO)    │                        │
│              │  · Region 调度         │                        │
│              │  · 元信息管理          │                        │
│              └───────────┬───────────┘                         │
│                          │                                      │
│         ┌────────────────┼────────────────┐                      │
│         ▼                ▼                ▼                      │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐              │
│  │  TiKV      │   │  TiKV      │   │  TiKV      │   行存       │
│  │  Node 1    │   │  Node 2    │   │  Node 3    │   (OLTP)     │
│  │  (Peer)    │   │  (Leader)  │   │  (Peer)    │              │
│  └────────────┘   └────────────┘   └────────────┘              │
│                          │                                      │
│         ┌────────────────┼────────────────┐                      │
│         ▼                ▼                ▼                      │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐              │
│  │  TiFlash   │   │  TiFlash   │   │  TiFlash   │   列存       │
│  │  Replica 1 │   │  Replica 2 │   │  Replica 3 │   (OLAP)     │
│  └────────────┘   └────────────┘   └────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

**各组件职责：**

| 组件 | 职责 | 类比 |
|------|------|------|
| **TiDB Server** | SQL 解析、优化、执行（无状态）| MySQL Server |
| **TiKV** | 行存分布式 KV，支持 OLTP | 分片后的 MySQL 存储引擎 |
| **TiFlash** | 列存副本，支撑 OLAP 分析 | ClickHouse / Greenplum |
| **PD** | Timestamp Oracle (TSO)、Region 调度 | 分布式协调者 |

---

### 2. TiKV 存储引擎

#### 数据组织方式：Region + RocksDB

```
TiKV 内部存储结构：

key 格式：table_prefix/{table_id}/record_prefix/{row_id}
         或 table_prefix/{table_id}/index_prefix/{index_id}/{index_value}

每个 Region ≈ 96MB 数据（可配置）
├── Region 1: [start_key, end_key) → 负责 0~100000 的 row_id
├── Region 2: [start_key, end_key) → 负责 100000~200000 的 row_id
└── ...

Region 在 PD 的调度下在各 TiKV 节点间迁移（负载均衡）
```

**RocksDB 做持久化：**
- 每个 TiKV 是一个 RocksDB 实例
- WAL → MemTable → L0 → L1 → L2 ... (LSM-tree)
- TiKV 在 RocksDB 之上封装了自己的 MVCC 层

#### MVCC 与 Timestamp Oracle (TSO)

```go
// TiDB 的 MVCC 通过 (start_ts, commit_ts) 实现
//
// 写入时：
//   start_ts = PD.AllocTimestamp()  // 当前事务的起始时间戳
//   commit_ts = PD.AllocTimestamp()  // 提交时的时间戳
//
// key 存储格式：
//   key + commit_ts → value  (已提交)
//   key + start_ts  → value  (未提交，会被 GC 清理)
//
// 读取时：
//   read_ts = current_read_timestamp
//   只读取 commit_ts <= read_ts 的版本
```

**TSO（Timestamp Oracle）：**
- PD 节点负责分配全局递增时间戳
- 解决了分布式环境下"谁的时间戳更大"的问题
- 单调递增 → 保证事务的全局顺序

---

### 3. 分布式事务：Percolator 协议

这是 TiDB 面试的核心深度点。

#### Percolator 原理（简化的 2PC）

```
事务流程（乐观事务）：

1. Client 从 PD 获取 start_ts
2. 读取数据（检查是否有锁）
3. Client 在本地写入 buffer
4. Client 获取 commit_ts
5. 两阶段提交：
   a. Prewrite：所有 key 写入 + lock（带 start_ts）
   b. Commit：从 PD 获取 commit_ts，删除 lock，写入 data + commit_ts
6. 事务提交成功
```

```go
// Percolator 两阶段提交的简化代码逻辑

// Prewrite 阶段
func ( txn *Txn) Prewrite() error {
    for _, key := range txn.writes {
        // 检查是否有其他事务的锁
        lock, err := txn.store.getLock(key)
        if lock != nil {
            return ErrWriteConflict  // 乐观锁，冲突就回滚
        }
        // 写入 prewrite lock
        txn.store.put(key, &Lock{
            PrimaryKey: txn.primaryKey,
            StartTS:    txn.startTS,
            TTL:        0,  // 乐观事务无 TTL
        })
    }
    return nil
}

// Commit 阶段
func (txn *Txn) Commit() error {
    commitTS := pd.AllocTimestamp()

    // 1. 写入主 key 的 commit 信息
    txn.store.put(txn.primaryKey, &Write{
        StartTS:  txn.startTS,
        CommitTS: commitTS,
        Value:    ...,
    })
    txn.store.delete(txn.primaryKey, LockType)  // 删除锁

    // 2. 写入其他 key（异步）
    for _, key := range txn.writes[1:] {
        txn.store.put(key, &Write{...})
        txn.store.delete(key, LockType)
    }
    return nil
}
```

**乐观锁 vs 悲观锁：**

| 维度 | 乐观事务（默认）| 悲观事务 |
|------|--------------|---------|
| 冲突检测时机 | 提交时（两阶段检查）| 写入时（先加锁）|
| 适用场景 | 低冲突（多数写入为主）| 高冲突（多数更新为主）|
| 性能 | 冲突多时浪费工作 | 冲突多时效率更高 |
| 配置 | `set @@tidb_txn_mode='optimistic'` | `set @@tidb_txn_mode='pessimistic'` |

---

### 4. HTAP：行存 + 列存同时服务

#### 为什么需要 TiFlash？

```
传统架构（分库分表）：
  OLTP 系统 ──▶ MySQL ──▶ 定期 ETL ──▶ Hive/Spark（离线分析）

问题：T+1 延迟，数据量大时 ETL 很慢

TiDB HTAP：
  TiKV（行存）───同步复制───▶ TiFlash（列存）
       │                              │
       ▼                              ▼
   实时写入                        实时分析
   (OLTP)                         (OLAP)
```

**TiFlash 工作原理：**
- TiFlash 是 TiKV 的**只读副本**，通过 Raft Learner 协议同步数据
- 列式存储，兼容 ClickHouse 引擎（向量化执行）
- 分析查询（OLAP）自动路由到 TiFlash，不影响 OLTP 写入

```go
-- TiDB 自动判断走哪个引擎
-- OLTP 查询（点查）→ TiKV
EXPLAIN SELECT * FROM orders WHERE order_id = 123;
-- 输出：TableFullScan → TiKV

-- OLAP 查询（分析）→ TiFlash
EXPLAIN SELECT region, SUM(amount) FROM orders GROUP BY region;
-- 输出：TableFullScan → TiFlash
```

---

### 5. 与 MySQL 分库分表对比

| 维度 | TiDB | MySQL 分库分表（ShardingSphere）|
|------|------|--------------------------------|
| **一致性模型** | 强一致（Percolator）| 弱一致（跨分片无分布式事务）|
| **扩容方式** | 在线加节点，PD 自动调度 | 重新分片（ShardingSphere Proxy）|
| **SQL 支持度** | 99%+（跨节点 JOIN 支持）| 有限（跨分片 JOIN 困难）|
| **运维复杂度** | 中（TiDB/TiKV/PD）| 高（ShardingSphere + MySQL 集群）|
| **事务边界** | 全局分布式事务 | 单分片事务，跨分片需额外处理 |
| **MySQL 兼容性** | 高 | 高（语法兼容）|
| **单表数据上限** | 理论上无限制 | 受分片数限制 |
| **成本** | 高（3 节点起步，SSD 要求高）| 低（可用普通 MySQL）|
| **推荐场景** | OLTP + OLAP 混合、弹性扩展需求 | 纯 OLTP，数据量可预估 |

---

### 6. 全局自增 ID 实现

#### TiDB 的 auto_random

```sql
-- 创建表时使用 auto_random（推荐）
CREATE TABLE orders (
    id BIGINT NOT NULL PRIMARY KEY AUTO_RANDOM,  -- 分布式自增
    ...
) AUTO_ID_CACHE = 100;  -- 预分配 100 个 ID 批量使用

-- 插入时
INSERT INTO orders (id, ...) VALUES (DEFAULT, ...);
-- TiDB 自动分配全局唯一 ID

-- 获取 last_insert_id
SELECT LAST_INSERT_ID();  -- 返回本次插入的 ID
```

**auto_random 的原理：**
```
TiDB 的 auto_random = (shard_id << 58) | (auto_increment_id % cache_size)

shard_id = 事务所在 TiDB 实例的物理切分标识
auto_increment_id = PD 分配的批量 ID（每次 cache_size 个）

优势：
- 不同 TiDB 实例的 shard_id 不同，天然避免冲突
- 批量分配减少 PD 的压力
- 隐式 ID 设计，避免主键暴露
```

#### 为什么不用 auto_increment？

```sql
-- ❌ 不推荐：auto_increment 在 TiDB 中是表级锁，高并发有瓶颈
CREATE TABLE orders (
    id BIGINT NOT NULL PRIMARY KEY AUTO_INCREMENT,
    ...
);

-- ✅ 推荐：auto_random 无锁，高并发写入性能更好
CREATE TABLE orders (
    id BIGINT NOT NULL PRIMARY KEY AUTO_RANDOM,
    ...
);
```

---

### 7. 从 MySQL 迁移到 TiDB

#### 迁移步骤

```
1. 评估阶段
   ├── 使用 pt-table-checksum 校验数据一致性
   ├── 评估 SQL 兼容性（TiDB 已兼容 80%+ MySQL 语法）
   └── 识别不支持的 SQL（如外键约束、存储过程）

2. 数据迁移
   ├── 全量迁移：TiDB DM (Data Migration) 工具
   │   └── DM 支持全量 + 增量同步（binlog 监听）
   └── 增量同步：开启 MySQL binlog → DM 消费 → TiDB 写入

3. 切流量
   ├── 先灰度 1% 读流量到 TiDB
   ├── 验证查询延迟、错误率
   └── 逐步将写流量切到 TiDB

4. 回滚方案
   └── 保留 MySQL 只读副本，TiDB 出问题可回切
```

#### TiDB DM 工具使用

```bash
# dm-master 配置
server:
  endpoints:
    - 192.168.1.100:8261

# 创建数据源
tiup dmctl operate-source create ./source.yaml
# source.yaml:
# from:
#   host: 192.168.1.10
#   port: 3306
#   user: repl_user
#   password: "xxx"

# 创建迁移任务
tiup dmctl --master-addr 192.168.1.100:8261 \
    create-task full.yml --name mymigration

# full.yml:
# name: mymigration
# task-mode: full
# targets:
#   - mysql-replication-01
# migration:
#   filter:
#     schema-pattern: "orders_*"
#     table-pattern: "*"
```

---

## 高频追问

### Q1: TiDB 的全局自增 ID 是怎么实现的？

**答：** TiDB 用 `auto_random` + PD 批量分配实现全局唯一 ID：
1. PD 维护一个全局 allocator，每次给 TiDB Server 分配一个范围（如 1~100）
2. TiDB Server 在这个范围内自行递增，不依赖 PD
3. 分配完后再向 PD 申请下一个范围
4. 实际存储格式包含 shard_id（TiDB 实例标识），保证不同实例的 ID 不冲突

### Q2: TiDB 适合什么场景，不适合什么场景？

**答：**
**适合：**
- 数据量超过单 MySQL 实例上限（>1TB）
- 需要强一致性的 OLTP（银行、交易系统）
- 需要同时跑 OLTP + OLAP（HTAP）
- 需要水平扩展又不想分库分表

**不适合：**
- 小数据量（<100GB），用 MySQL 单机就够了
- 纯 OLAP 场景（用 ClickHouse / Snowflake 更便宜）
- 对成本敏感，TiDB 硬件要求高（NVMe SSD 起步）
- 超高并发写入（>10万 QPS），Kafka + ClickHouse 更合适

### Q3: TiDB 和 CockroachDB 对比？

**答：**
| 维度 | TiDB | CockroachDB |
|------|------|-------------|
| 共识协议 | Raft | Raft |
| SQL 兼容 | MySQL | PostgreSQL |
| HTAP | ✅ TiFlash | ❌ 纯 OLTP |
| 生态成熟度 | 高（国内企业大量使用）| 中（海外为主）|
| 成本 | 中 | 高 |
| 推荐场景 | 国内业务，MySQL 团队 | 海外业务，PostgreSQL 团队 |

---

## 延伸阅读

| 资料 | 链接 |
|------|------|
| TiDB 官方文档 | https://docs.pingcap.com/tidb/stable |
| Percolator 论文 | https://research.google.com/pubs/papers/percolator.pdf |
| TiFlash 原理 | https://docs.pingcap.com/tidb/stable/tiflash-overview |
| DM 数据迁移工具 | https://docs.pingcap.com/tidb/stable/dm-overview |
| TiDB vs MySQL 分库分表 | https://pingcap.com/blog/tidb-vs-mysql-sharding |
