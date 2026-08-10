[🏠 首页](../../../README.md) · [📦 分布式系统](../README.md) · [💬 消息队列](../03-mq-problems/README.md)

---

# Kafka 消息可靠性：ACK 机制、ISR、幂等生产者、事务

## 面试官考察意图

考察候选人对 Kafka 消息可靠性的理解深度。初级能说出"acks=all 配置"，高级要能讲清楚 **ISR 机制、min.insync.replicas 的边界条件、幂等生产者防止重复的原理、Kafka 事务的 exactly-once 语义**，并能结合生产故障（如 ISR 收缩、脑裂）给出分析。

---

## 核心答案（30 秒版）

Kafka 可靠性从三个层面保障：

| 层面 | 核心配置 | 作用 |
|------|----------|------|
| **生产者** | `acks=all` + 幂等生产者 | 确保消息写入成功且不重复 |
| **Broker** | `min.insync.replicas=2` + 同步刷盘 | 确保消息在多个副本持久化 |
| **消费者** | 手动提交 offset + 业务幂等 | 确保处理成功后再提交 |

**exactly-once 语义**：Kafka 3.3+ 通过幂等生产者 + 事务 API 实现端到端 exactly-once，适用于金融级场景。

---

## 深度展开

### 1. 生产者 ACK 机制

```go
// Kafka Go 客户端配置
producer := &kafka.ProducerConfig{
    // acks 配置决定写入成功条件
    "acks": 1,        // Leader 写入即返回（快，可能丢）
    // "acks": -1,    // 等所有 ISR 确认（慢，不丢）
    // "acks": 0,     //  fire-and-forget（最快，必然丢）
}
```

**acks=-1（等价于 all）的写入流程**：

```
Producer 发送消息
    │
    ├─ Leader Broker 写入本地日志
    │
    ├─ Follower Broker 同步日志（HW 更新）
    │     等待条件：ISR 列表中所有副本都写入
    │
    └─ Producer 收到确认
```

**HW（High Watermark）机制**：

```
┌─────────────────────────────────────────────┐
│  LEO (Log End Offset) ──────────────────│  ← 最新消息
│                                             │
│  ...                                        │
│                                             │
│  HW ────────────────────────────────────│  ← 已同步水位，消费者可见上限
│                                             │
│  ...                                        │
│  0 ─────────────────────────────────────│  ← 旧消息，消费者已可见
└─────────────────────────────────────────────┘

HW = min(all ISR LEO)，确保消费者不读到未同步的脏数据
```

### 2. ISR 机制（In-Sync Replicas）

**ISR 列表**：与 Leader 保持同步的副本集合。

```
ISR 动态变化场景：

正常情况：
  Partition 0: Leader(P0) + Follower(P1) + Follower(P2)  →  ISR = [P0, P1, P2]

Follower P1 宕机（落后超过 replica.lag.time.max.ms）：
  ISR = [P0, P2]          →  容忍 1 节点故障

Leader P0 宕机：
  Controller 从 ISR 中选择新 Leader（P2）
  选举后的 ISR = [P2]（P2 成为新 Leader）
  
  如果 acks=all 且 min.insync.replicas=2：
  写入会被拒绝（因为 ISR < min.insync.replicas）
  → 写入失败，Producer 收到 NotEnoughReplicasException
```

**关键配置**：

```properties
# 副本落后多少ms认为失联
replica.lag.time.max.ms=30000

# 副本落后多少条消息认为失联（已废弃，优先用时间）
replica.lag.max.messages=10000

# ISR 最少副本数（核心！）
min.insync.replicas=2

# 不选脏数据为 Leader（防止数据丢失）
unclean.leader.election.enable=false
```

**生产故障案例：ISR 收缩导致写入失败**

```
故障场景：3 节点 Kafka，min.insync.replicas=2
  1. 网络抖动，P1 落后超过 30s
  2. Controller 将 P1 踢出 ISR
  3. ISR = [P0, P2]，此时如果 P0 宕机
  4. 新 Leader P2 成为唯一节点，ISR = [P2]
  5. ISR(1) < min.insync.replicas(2)，写入拒绝

影响：生产者重试，但超过重试次数后消息丢失

解决方案：
  - 监控 ISR 变化（Kafka JMX: kafka.server:type=ReplicaManager,name=IsrExpandsRate）
  - 设置 acks=all + retries=Integer.MAX_VALUE
  - 适当调大 replica.lag.time.max.ms
```

### 3. 幂等生产者（Idempotent Producer）

**解决的问题**：网络重试导致消息重复。

```
重试场景（无幂等）：
  1. Producer 发送消息给 Broker
  2. Broker 写入成功，但响应丢失
  3. Producer 超时重试
  4. Broker 收到重复消息 → 重复写入

幂等生产者原理：
  每个 Producer 有唯一的 ProducerId
  每个 Batch 有序列号（Sequence Number）
  Broker 只接受"连续下一个序列号"的消息
  → 重复消息被拒绝
```

```go
// 开启幂等生产者
producer := &kafka.ProducerConfig{
    "enable.idempotence": true,   // 开启幂等（默认 true，Go 客户端需手动开启）
    "acks": "all",                // 幂等必须配合 acks=all
    "retries": 3,
    "max.in.flight.requests.per.connection": 5,
    // 注意：max.in.flight ≤ 5 才能保证幂等下的顺序
}

// 幂等生产者保证了"生产者级别的 exactly-once"
// 但不保证"消费端的 exactly-once"（事务解决）
```

**幂等限制**：

- 单 Producer 内有序（通过 Sequence Number 保证）
- 跨 Producer 无序（不同 ProducerId 不能去重）
- 最多保证 5 个 in-flight 请求（Go 客户端）

### 4. Kafka 事务（Exactly-Once Semantics）

**三种语义对比**：

| 语义 | 定义 | 实现难度 |
|------|------|---------|
| **at-most-once** | 消息最多投递一次（可能丢，不重复）| 简单 |
| **at-least-once** | 消息至少投递一次（不丢，可能重复）| 中等 |
| **exactly-once** | 消息恰好投递一次（不丢，不重复）| 复杂 |

**Kafka 事务实现**：跨分区原子写入 + 消费者精确提交。

```go
// Kafka 事务示例：写 Kafka + 写数据库 原子提交
func produceWithTransaction(kafkaProducer *kafka.Producer, db *sql.DB) error {
    tx, err := kafkaProducer.BeginTransaction()
    if err != nil {
        return err
    }

    // 开启数据库事务
    dbTx, err := db.Begin()
    if err != nil {
        tx.Abort()
        return err
    }

    // 1. 发送 Kafka 消息（PREPARE 阶段）
    err = tx.SendMessage(&kafka.Message{
        Topic: "orders",
        Key:   []byte("order-123"),
        Value: []byte(`{"order_id":"123","amount":100}`),
    })
    if err != nil {
        tx.Abort()
        dbTx.Rollback()
        return err
    }

    // 2. 写数据库
    _, err = dbTx.Exec(`INSERT INTO orders ...`)
    if err != nil {
        tx.Abort()
        dbTx.Rollack()
        return err
    }

    // 3. 提交数据库事务
    dbTx.Commit()

    // 4. 提交 Kafka 事务
    return tx.Commit()  // 原子保证：要么全成功，要么全回滚
}
```

**消费端事务（Read-Commit）**：

```go
// 消费 + 业务处理 + 提交 offset 原子化
consumer := kafka.NewConsumer([]string{"localhost:9092"}, "my-group")

// 配置事务隔离级别
consumer.SetOffsetIsolationLevel(kafka.IsolationLevelReadCommitted)

for {
    msg := consumer.Poll()
    if msg == nil {
        continue
    }

    // 消费事务消息（只有已提交的事务消息才会被投递）
    err := process(msg.Value)
    if err != nil {
        // 处理失败，不提交 offset，下次重新消费
        continue
    }

    // 业务处理成功后，手动提交 offset
    consumer.CommitOffsets(msg.Offset)
}
```

**exactly-once 适用场景**：

```
✅ 适合：跨系统原子写入（Kafka → MySQL）、金融交易
❌ 不适合：普通日志收集（at-least-once + 幂等足够）
```

---

## 高频追问

**Q：acks=all 性能很差，怎么优化？**

> acks=all 确实慢，因为要等所有 ISR 确认。优化方案：
> 1. 批量发送：`batch.size=65536` + `linger.ms=20`（合并小消息）
> 2. 压缩：`compression.type=snappy`（减少网络传输量）
> 3. 异步 + 回调：发送和确认解耦，不阻塞主流程
> 4. 选择合适的 acks：普通日志用 acks=1，核心数据用 acks=all

**Q：min.insync.replicas=1 会有什么问题？**

> 写入成功后 Leader 宕机，新 Leader 没有这条消息，导致数据丢失。这通常发生在网络分区场景：写入成功 → Leader 故障 → 旧 Leader 被踢出 ISR → 新 Leader 成为唯一节点（数据落后）。兜底方案：`unclean.leader.election.enable=false`。

**Q：如何监控 Kafka 可靠性指标？**

```bash
# 核心 JMX 指标
kafka.server:type=ReplicaManager,name=UnderMinIsrPartitionCount  # ISR 不足分区数
kafka.server:type=ReplicaManager,name=OfflinePartitionsCount      # 离线分区数
kafka.producer:type=producer-metrics,name=record-error-rate        # 生产错误率
kafka.consumer:type=consumer-metrics,name=records-lag-max           # 最大消费延迟
```

---

## 延伸阅读

- [Kafka 官方文档 - Replication](https://kafka.apache.org/documentation/#replication)
- [KIP-631: KRaft 模式替代 ZooKeeper](https://cwiki.apache.org/confluence/display/KAFKA/KIP-631%3A+The+Quorum-based+Controller)
- [Go Kafka 客户端 sarama](https://github.com/Shopify/sarama)
