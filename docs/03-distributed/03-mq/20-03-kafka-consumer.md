[🏠 首页](../../../README.md) · [📦 分布式系统](../README.md) · [💬 消息队列](../03-mq-problems/README.md)

---

# Kafka 消费者组与 Rebalance：分区分配策略、触发条件、Session Timeout

## 面试官考察意图

考察候选人对 Kafka 消费者组机制的理解深度。初级能说出"消费者组内分区均分"，高级要能讲清楚 **分区分配算法（Range vs RoundRobin vs StickyAssignor）、Rebalance 触发条件和副作用、Session Timeout 与心跳的关系、Max Poll Interval 的影响**，并能在生产中处理 Rebalance 导致的服务抖动问题。

---

## 核心答案（30 秒版）

Kafka 消费者组机制：

| 概念 | 说明 |
|------|------|
| **Consumer Group** | 多个消费者组成一个组，共同消费一个 Topic |
| **分区分配** | 每个分区只被组内一个消费者消费（负载均衡）|
| **Rebalance** | 消费者增减时，重新分配分区的过程 |
| **副作用** | Rebalance 期间消费暂停，且可能重复消费 |

**核心原则**：一个 Partition 只能被一个 Consumer 消费；一个 Consumer 可以消费多个 Partition。

---

## 深度展开

### 1. 消费者组机制

```
Topic: order-events (6 个 Partition)

场景1：消费者组 group-A 有 3 个消费者
┌─────────────────────────────────────────────────────┐
│  Partition 0 ─── Consumer C1                        │
│  Partition 1 ─── Consumer C1                        │
│  Partition 2 ─── Consumer C2                        │
│  Partition 3 ─── Consumer C2                        │
│  Partition 4 ─── Consumer C3                        │
│  Partition 5 ─── Consumer C3                        │
└─────────────────────────────────────────────────────┘
  → 每个 Consumer 消费 2 个 Partition（均分）

场景2：消费者组 group-A 有 6 个消费者
┌─────────────────────────────────────────────────────┐
│  Partition 0 ─── Consumer C1                        │
│  Partition 1 ─── Consumer C2                        │
│  Partition 2 ─── Consumer C3                        │
│  ...                                                │
└─────────────────────────────────────────────────────┘
  → 每个 Consumer 消费 1 个 Partition

场景3：消费者组 group-A 有 8 个消费者
┌─────────────────────────────────────────────────────┐
│  Partition 0 ─── Consumer C1                        │
│  Partition 1 ─── Consumer C2                        │
│  ...                                                │
│  Partition 5 ─── Consumer C6                        │
│  Consumer C7 ─── (不消费任何 Partition)             │
│  Consumer C8 ─── (不消费任何 Partition)             │
└─────────────────────────────────────────────────────┘
  → 多余消费者空闲（Partition 数 = Consumer 数上限）
```

### 2. 分区分配策略（Partition Assignor）

Kafka 提供三种分配策略：

```go
// Go 客户端配置
consumer := &kafka.ConfigMap{
    "partition.assignment.strategy": "range",  // 默认
    // "range"       → RangeAssignor（按 Topic 逐个分配）
    // "roundrobin"  → RoundRobinAssignor（全局轮询）
    // "sticky"      → StickyAssignor（保持已有分配，最小化移动）
}
```

#### RangeAssignor（默认）

按 Topic 逐个分配，**可能导致分配不均**。

```
Topic A 有 10 个 Partition，消费者组有 3 个消费者 C1, C2, C3

分配计算：10 / 3 = 3（每人至少），余数 1

C1: 分区 0,1,2,3      （3 + 1，余数给前面的）
C2: 分区 4,5,6
C3: 分区 7,8,9

问题：如果有多个 Topic，多个 Topic 的余数都集中给 C1 → C1 负载过重
```

#### RoundRobinAssignor

全局轮询，分配更均匀。

```
Topic A (3 Partition) + Topic B (3 Partition)，消费者 C1, C2, C3

打平所有 Partition：P0A, P1A, P2A, P0B, P1B, P2B
轮询分配：
  C1: P0A, P1B
  C2: P1A, P2B
  C3: P2A, P0B
```

#### StickyAssignor（推荐生产使用）

保持已有分配不变，**Rebalance 时最小化 Partition 移动**。

```
Rebalance 前：C1 消费 P0,P1,P2，C2 消费 P3,P4,P5
新增 C3 后（Sticky）：

C1: P0,P1  （尽可能保留）
C2: P3,P4  （尽可能保留）
C3: P2,P5  （新分配的，且移动最少）

对比 RoundRobin：C1 可能变成 P0,P3 → 需要关闭消费位点再重启
```

### 3. Rebalance 触发条件

```
触发 Rebalance 的 5 种情况：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. 消费者组成员加入（新增 Consumer）
2. 消费者组成员离开（主动退出 / 宕机）
3. 消费者组成员心跳超时（Session Timeout 过期）
4. Topic 分区数变更（运维操作）
5. Topic 订阅表达式匹配的新 Topic（正则订阅）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**心跳机制**：

```go
consumer := &kafka.ConfigMap{
    "session.timeout.ms": 30000,    // Session 超时，30s 没心跳视为宕机
    "heartbeat.interval.ms": 3000,  // 心跳间隔（必须 < session.timeout.ms 的 1/3）
    "max.poll.interval.ms": 300000, // 最大 poll 间隔（业务处理超时上限）
}
```

```
时间线：
  t=0    → Consumer poll() 消息，处理耗时 5 分钟
  t=30s  → 心跳发送（正常）
  t=60s  → 心跳发送（正常）
  t=3min → poll() 返回，但处理还在进行
  t=5min → 消费者还未再次 poll，Broker 认为心跳超时
  t=5min → Broker 触发 Rebalance，该消费者被踢出组
  t=5min → 其他消费者开始 Rebalance，重新分配分区
  
  ⚠️ 如果 max.poll.interval.ms < 处理耗时，会触发 Rebalance
```

### 4. Rebalance 生命周期（详细流程）

```
Rebalance 流程（Consumer 侧视角）：

1. Consumer 加入组
   Consumer.send JoinGroupRequest
   │
   2. 等待 Leader Consumer 收集所有成员
      Group Leader = 第一个加入的 Consumer
      │
   3. Leader 制定分配方案
      Leader 收到所有成员的 subscription 信息
      根据 PartitionAssignor 计算分配
      │
   4. 同步分配方案
      Leader 将分配方案写入 __consumer_offsets Topic
      所有 Consumer 从该 Topic 读取自己的分配
      │
   5. 开始消费
      每个 Consumer 各自 poll() 自己分配的 Partition
```

**JoinGroup 请求详解**：

```go
// 消费者订阅
consumer.SubscribeTopics([]string{"order-events", "payment-events"}, nil)

// Consumer Group 状态机
// ┌──────────────┐  JoinGroup   ┌──────────────────┐
// │  Empty       │ ──────────→ │  PreparingRebalance│
// └──────────────┘              └──────────────────┘
//                                   │
//                         所有成员 JoinGroup 到来
//                                   │
// ┌──────────────┐  StartRebalance  ┌──────────────────┐
// │  AwaitSync   │ ←────────────── │  PreparingRebalance│
// └──────────────┘                 └──────────────────┘
//       │ SyncGroup                      │
//       │ (Leader 同步分配方案)              │
//       ▼                                ▼
// ┌──────────────┐              ┌──────────────────┐
// │  Stable      │ ←────────── │  AwaitSync        │
// │  (正常消费)  │              └──────────────────┘
// └──────────────┘
```

### 5. Rebalance 带来的问题与解决方案

#### 问题一：消费重复（最常见）

```
Rebalance 发生在 poll() 返回消息后、业务处理中：

时刻1：Consumer poll() 100 条消息，返回给业务处理
时刻2：业务处理 50 条后，Rebalance 触发
时刻3：Broker 认为 Consumer 心跳超时，踢出组
时刻4：Consumer 2 接管这些分区，重新消费
时刻5：Consumer 1 处理完之前 50 条，发现 Rebalance，重新 poll
结果：100 条消息中，50 条被 Consumer 1 处理，50 条被 Consumer 2 处理 → 重复
```

**解决方案**：

```go
// 1. 业务处理幂等（兜底方案）
// 2. 手动提交 offset（先处理，再提交）
consumer := &kafka.ConfigMap{
    "enable.auto.commit": false,  // 关闭自动提交
}

for msg := consumer.Poll(); msg != nil; msg = consumer.Poll() {
    err := process(msg.Value)
    if err != nil {
        logError(err)
        continue  // 不提交，继续重试
    }
    // 处理成功，手动提交
    consumer.CommitOffset(msg.Topic, msg.Partition, msg.Offset+1)
}

// 3. 增大 max.poll.interval.ms（给业务更多处理时间）
"max.poll.interval.ms": 600000  // 10 分钟
```

#### 问题二：消费抖动（业务中断）

```
Rebalance 期间：
  - 所有 Consumer 停止消费（STW 效应）
  - Group Leader 重新计算分配
  - 消费中断时间 = JoinGroup + Sync 时间
  
  通常 100ms~2s，但高负载下可能 > 10s
```

**优化方案**：

```go
// 1. 使用 StickyAssignor（减少 Partition 移动）
"partition.assignment.strategy": "org.apache.kafka.clients.consumer.StickyAssignor"

// 2. 合理设置 Session Timeout
"session.timeout.ms": 10000      // 太短容易误判，太长发现慢消费者慢
"heartbeat.interval.ms": 3000    // 必须是 session.timeout.ms 的 1/3

// 3. 监控 Rebalance 频率
# JMX 监控
kafka.consumer:type=consumer-coordinator-metrics,name=rebalance-latency-avg

// 如果 rebalance-latency 持续 > 5s，说明有消费者处理太慢
```

#### 问题三：消费者 "僵尸"（Zombie Consumer）

```
场景：Consumer 处理非常慢，Session Timeout 后被踢出
      但处理仍在进行，完成后提交 offset
      此时分区已分配给其他 Consumer → offset 错乱

解决方案：引入 GenerationId（代数）

每次 Rebalance 后，Broker 分配新的 generation_id
Consumer 提交 offset 时必须携带当前的 generation_id
如果 generation_id 不匹配，说明 Consumer 是僵尸，拒绝提交
```

```go
// Kafka 新版本（2.4+）默认保护
// 旧版 Consumer 需要升级
```

### 6. 消费者数量与分区数的关系

```
黄金法则：消费者数量 ≤ Topic 分区数

分区数 = N
  - 消费者 = N：每个消费者消费 1 个分区（最大并行度）
  - 消费者 < N：部分分区空闲（浪费）
  - 消费者 > N：多余消费者空闲（无意义）

动态调整消费者：
  扩容消费者 → Rebalance → 分区重新分配
  缩容消费者 → Rebalance → 分区重新分配
  
  建议：使用 Kubernetes HPA 配合 Kafka 消费者，根据 lag 自动扩缩
```

```go
// 监控消费 lag，根据 lag 决策扩缩容
func monitorLag(consumer *kafka.Consumer) {
    for {
        lags, _ := consumer.Lag()
        for _, lag := range lags {
            if lag > 10000 {
                // 触发扩容：增加消费者实例
                scaleUp()
            }
            if lag < 1000 {
                // 触发缩容：减少消费者实例
                scaleDown()
            }
        }
        time.Sleep(10 * time.Second)
    }
}
```

---

## 高频追问

**Q：Session Timeout 和 Heartbeat Interval 怎么配合？**

> Heartbeat Interval 必须小于 Session Timeout 的 1/3。比如 Session Timeout=30s，Heartbeat Interval 就设 10s（3次/30s），留足够时间让心跳触发。如果业务处理时间长，优先调大 `max.poll.interval.ms`（控制业务处理超时），而不是调大 Session Timeout。

**Q：消费者实例数比分区数多怎么办？**

> 多余的消费者实例会空闲，不消费任何分区。这通常发生在临时扩容后忘记缩容，或者分区数规划不合理时。解决方案：监控 `consumer.assigned.partitions.count`，如果实例数 > 分区数，告警。

**Q：Rebalance 时如何保证消息不丢失？**

> Kafka 本身不丢消息（消息在 Broker 磁盘），但 Rebalance 期间消费中断可能导致处理中断。关键是：1）手动提交 offset，在处理成功后提交；2）业务处理幂等，应对重复消费；3）Rebalance 时记录当前处理进度，支持断点恢复。

**Q：如何减少 Rebalance 对业务的影响？**

> 1. 业务处理异步化：poll() 后立即提交到 channel，异步 worker 处理，主线程继续 poll()。2. 使用 StickyAssignor 减少分区移动。3. 业务处理超时后不退出，只记录异常并继续。4. 监控 rebalance 频率，出现频繁 Rebalance 时排查根因（通常是消费者心跳超时）。

---

## 延伸阅读

- [Kafka KIP-548: Consumer Group Membership](https://cwiki.apache.org/confluence/display/KAFKA/KIP-548%3A+Add+Static+Membership+to+the+Consumer+Group)
- [Kafka Java Consumer 设计解析](https://mp.weixin.qq.com/s/XhHUA0mIU16iNrsBSX3Wew)
- [sarama consumer example](https://github.com/Shopify/sarama/blob/main/examples/consumer_group/consumer_group.go)
