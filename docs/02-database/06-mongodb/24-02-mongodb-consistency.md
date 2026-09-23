# MongoDB 副本集选举、读写关注与一致性（生产必考）

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：副本集、OPLOG、Raft 变体、w:majority、readConcern、writeConcern、读己之写、Change Stream

## 面试官考察意图

MongoDB 在面试里往往与「分布式一致性」联动考察：

1. "MongoDB 怎么保证高可用？" → 副本集 + 自动选举
2. "写入了会不会丢？" → `writeConcern` 与 journal 的配合
3. "主从切换期间读到了旧数据怎么办？" → `readConcern` 与显式因果一致性

**区分度**：能不能把 `writeConcern` / `readConcern` / `readPreference` 三个概念讲清楚且不混淆。

---

## 核心答案（30 秒版）

| 问题 | 一句话回答 |
|------|-----------|
| 副本集怎么选主？ | 类 Raft：需要**多数派（N/2+1）**投票，心跳 10s 超时触发选举；只有能投票的成员（最多 7 个）和优先级影响结果 |
| 写丢了怎么办？ | 默认 `w:1`（ack 主节点）在切换瞬间可能回滚；要强保证用 `w:"majority"` + `j:true` |
| 为什么切换后读到旧数据？ | 新主的数据不一定追平旧主；用 `readConcern: "majority"` 读多数派已确认的数据，或开启 `readConcern: "snapshot"` + 因果会话 |
| 三个 concern 区别？ | `writeConcern`=写要几份确认；`readConcern`=读的数据要什么一致性级别；`readPreference`=**从哪读**（primary/secondary） |
| 怎么做数据同步？ | 从节点持续拉取主节点的 `oplog`（幂等重放），Change Stream 让应用订阅 oplog 变更 |

---

## 深度展开

### 1. 副本集架构与选举

```
        ┌──────────────┐
        │  Primary 主   │ ← 唯一接受写
        └──────┬───────┘
        oplog  │  (capped collection, 幂等重放)
    ┌──────────┴──────────┐
    ▼                     ▼
┌─────────┐          ┌─────────┐
│Secondary│          │Secondary│   ← 可读(readPreference)、可投票
└─────────┘          └─────────┘
        （次选：Arbiter 仲裁节点，只投票不存数据 —— 生产不推荐）

选举规则：
  - 需要 多数派 = floor(N/2) + 1 票（3 节点需 2 票）
  - 只有 votes>0、priority>0、且数据足够新的成员能当选
  - 节点数必须为奇数，避免 2:2 平票
  - priority 高的先当选；设置 priority:0 的节点永远不会成为主
```

**关键参数：**

```javascript
// 心跳与选举超时（默认 10s）
rs.conf()   // settings.heartbeatTimeoutSecs、electionTimeoutMillis

// 隐藏节点 / 延迟节点（用于备份、报告，不参与前台读）
{ _id: 2, host: "...", priority: 0, hidden: true, secondaryDelaySecs: 3600 }
```

### 2. writeConcern：写到几分才算成功

| 取值 | 含义 | 风险 |
|------|------|------|
| `{w:1}`（默认） | 主节点写入内存即返回 | 主节点宕机且未同步时数据**回滚丢失** |
| `{w:"majority"}` | 大多数节点已确认 | 切换不丢已确认写；延迟略高 |
| `{w:1, j:true}` | 主节点写入 journal 才返回 | 防进程崩溃，但防不了主机宕机 |
| `{w:"majority", j:true}` | 多数派落 journal | **强持久性**，金融类场景基线 |

```go
// Go driver：生产基线
coll := db.Collection("orders", options.Collection().
    SetWriteConcern(writeconcern.Majority()))

_, err := coll.InsertOne(ctx, doc,
    options.InsertOne().SetWriteConcern(writeconcern.New(
        writeconcern.WMajority(), writeconcern.J(true), writeconcern.WTimeout(5*time.Second))))
```

### 3. readConcern 与 readPreference

```
readConcern（数据一致性级别，决定"能读到什么"）：
  local         默认，读节点本地最新（可能回滚）
  available     不保证顺序（分片场景）
  majority      只读多数派已确认的数据 —— 不会读到随后回滚的数据
  linearizable  线性一致（只对单文档、primary 读有效）
  snapshot      事务/因果一致读的快照

readPreference（决定"去哪读"）：
  primary          只读主（默认，强一致）
  primaryPreferred 优先主，主不可用读从
  secondary        只读从（可能延迟/脏读）
  nearest          延迟最低者（可能读到从）
```

**读己之写（Read Your Own Writes）**：写入用 `w:"majority"`，随后读用 `readConcern:"majority"` 且配合**因果一致性会话**（`Session: causal_consistency=true`），即可保证「自己刚写的能读到」。

```go
// 因果一致性会话（Go driver 默认开启 causal consistency）
sess, _ := client.StartSession()
defer sess.EndSession(context.Background())
// 同一 session 内：写 majority + 读 majority ⇒ 保证因果有序
```

### 4. Change Stream：基于 oplog 的变更订阅

```go
// 应用无需轮询，直接订阅集合变更（底层读 oplog，从节点可用）
cs, err := coll.Watch(ctx, mongo.Pipeline{}, options.ChangeStream().
    SetFullDocument(options.UpdateLookup))
for cs.Next(ctx) {
    var ev struct {
        OperationType string             `bson:"operationType"`
        FullDocument  bson.M             `bson:"fullDocument"`
    }
    cs.Decode(&ev)
    // 触发下游：缓存失效 / 同步到 ES / 数仓
}
```

**注意**：为拿到更新后的完整文档，需要 `fullDocument: "updateLookup"`；Change Stream 需要副本集或分片集群（单机不支持）。

---

## 高频追问

### Q1：MongoDB 是 CP 还是 AP？

副本集默认走 CP 路线：主节点不可达时，**多数派存活才能选出新主**，否则整个集合**拒绝写入**（保证不脑裂）——这是典型的 CP 取舍。可通过 `readPreference: secondary` + `readConcern: local` 在**读侧**换取可用性，但读到的可能是旧数据。

### Q2：w:majority 一定不丢数据吗？

在**副本集**内，`w:majority` 保证「已确认的写不会因主节点故障而回滚」。但它不是跨数据中心/跨集群的事务保证；若以为「写了多数派就绝对安全」而忽略了**重启时 majority 也可能被重新选举回滚**（在极端分区恢复场景），就会踩坑。生产稳妥做法：`w:"majority"` + `j:true`。

### Q3：MongoDB 的多文档事务和 MySQL 有何不同？

MongoDB 4.0 起支持多文档事务（4.2 起支持分片事务），底层是**两阶段提交（2PC）变体**，跨分片时需要 mongos 协调，延迟与吞吐明显低于 MySQL 单节点事务。因此**能用单文档原子更新（`findOneAndUpdate` + `$inc`）解决的，不要用多文档事务**。

---

## 延伸阅读

- [MongoDB Docs - Replica set elections](https://www.mongodb.com/docs/manual/replication/#replica-set-elections)
- [MongoDB Docs - Write concern](https://www.mongodb.com/docs/manual/reference/write-concern/)
- [MongoDB Docs - Read concern](https://www.mongodb.com/docs/manual/reference/read-concern/)
