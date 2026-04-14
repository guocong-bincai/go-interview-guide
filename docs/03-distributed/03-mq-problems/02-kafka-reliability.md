# Kafka 消息可靠性

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：ACK、ISR、幂等生产者、事务、Exactly-Once

## 消息可靠性的三个层次

```
┌─────────────────────────────────────────────────┐
│                 消息可靠性金字塔                   │
├─────────────────────────────────────────────────┤
│  Exactly-Once（恰好一次）← 最强，但代价最高        │
│  At-Least-Once（至少一次）← 最常用，可能重复      │
│  At-Most-Once（至多一次）← 单调，但可能丢消息     │
└─────────────────────────────────────────────────┘
```

---

## 1. Producer 可靠性配置

### acks 参数详解

| acks 值 | 含义 | 可靠性 | 吞吐量 |
|---------|------|--------|--------|
| `0` | Producer 不等待响应 | ⚠️ 可能丢消息 | 最高 |
| `1` | Leader 写入后即返回 | ⚠️ Leader 挂丢消息 | 高 |
| `all`（-1）| ISR 全部写入后返回 | ✅ 最高可靠 | 最低 |

```go
// Go Producer 配置最佳实践
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/IBM/sarama"
)

func newReliableProducer(brokers []string) (sarama.SyncProducer, error) {
	config := sarama.NewConfig()
	
	// acks = all 保证消息写入所有 ISR 才返回
	config.Producer.RequiredAcks = sarama.WaitForAll
	
	// 等待 ISR 同步的超时时间
	config.Producer.Acks = 100 * time.Millisecond
	
	// 开启幂等生产者（Kafka 0.11+）
	config.Producer.Idempotent = true
	
	// 重试次数（网络抖动时自动重试）
	config.Producer.Retry.Max = 5
	
	// 开启异步确认（性能更好）
	config.Producer.Return.Successes = true
	config.Producer.Return.Errors = true
	
	// 批量发送：积攒 batch.size 或 linger.ms 后发送
	config.Producer.Batch.Size = 16 * 1024  // 16KB
	config.Producer.Linger.ms = 5            // 5ms 延迟批量
	
	// 压缩（减少网络流量）
	config.Producer.Compression = sarama.CompressionSnappy
	
	producer, err := sarama.NewSyncProducer(brokers, config)
	if err != nil {
		return nil, fmt.Errorf("create producer failed: %w", err)
	}
	
	return producer, nil
}

// 发送消息并处理回调
func sendWithCallback(producer sarama.SyncProducer, topic, key, value string) error {
	msg := &sarama.ProducerMessage{
		Topic: topic,
		Key:   sarama.StringEncoder(key),
		Value: sarama.StringEncoder(value),
		Headers: []sarama.RecordHeader{
			{Key: []byte("trace-id"), Value: []byte("req-123")},
		},
	}
	
	partition, offset, err := producer.SendMessage(msg)
	if err != nil {
		return fmt.Errorf("send message failed: %w", err)
	}
	
	fmt.Printf("Message sent to partition %d at offset %d\n", partition, offset)
	return nil
}
```

### 幂等生产者（Idempotent Producer）

**解决的问题**：Producer 重试时可能产生重复消息。

```go
// 幂等生产者原理
// 开启 idempotent = true 后，Producer 会被分配一个唯一 PID
// 每个消息带 (PID, Sequence Number)
// Broker 根据 (PID, SN) 去重，丢弃重复消息

config := sarama.NewConfig()
config.Producer.Idempotent = true  // 开启幂等
config.Producer.Return.Successes = true

// 注意：幂等生产者只能保证单个 Producer 实例内的消息不重复
// 跨 Producer 实例去重需要业务层做幂等
```

---

## 2. ISR 机制

### 什么是 ISR

```
ISR = In-Sync Replicas（同步副本集合）

Leader + 跟上 Leader 进度的 Follower 组成 ISR

判断"跟上进度"的标准：
  replica.lag.max.messages ≤ 配置阈值（默认 4000）
  replica.time.lag.ms ≤ 配置阈值（默认 10s）
```

### acks=all 时的可靠性保证

```
Producer 发送消息 → Leader 写入 → 等待所有 ISR 确认 → 返回 ACK

场景分析：
  假设 replication.factor = 3, min.insync.replicas = 2

  ✅ 正常情况：Leader + 1个 Follower 确认，消息可靠
  ⚠️ 极端情况：只有 Leader 在 ISR，只剩1个节点，仍可写入
  ❌ 崩溃情况：ISR < min.insync.replicas → 拒绝写入，返回异常
```

### min.insync.replicas 配置建议

```properties
# 平衡可靠性与可用性
# 推荐：replication.factor=3, min.insync.replicas=2
# 这样：容忍1个副本故障，仍可写入

min.insync.replicas = 2   # 写入时最少需要2个同步副本
```

---

## 3. 消费者可靠性

### 核心问题：消费到提交

```
Consumer 的可靠性关键：先消费，后提交 offset

❌ 错误顺序（At-Most-Once）：
  提交 offset → 处理消息
  → 如果处理失败，消息丢失

✅ 正确顺序（At-Least-Once）：
  处理消息 → 提交 offset
  → 如果处理失败，重试消费，消息不丢
```

### Go 消费者最佳实践

```go
package consumer

import (
	"context"
	"fmt"
	"log"
	"sync"

	"github.com/IBM/sarama"
)

// ReliableConsumer 可靠消费者
type ReliableConsumer struct {
	client        sarama.ConsumerGroup
	topics        []string
	handler       *messageHandler
	ready         chan bool
	ctx           context.Context
	cancel        context.CancelFunc
}

type messageHandler struct {
	processingFn func(msg *sarama.ConsumerMessage) error
}

func (h *messageHandler) Setup(sarama.ConsumerGroupSession) error {
	fmt.Println("Consumer group session started")
	return nil
}

func (h *messageHandler) Cleanup(sarama.ConsumerGroupSession) error {
	fmt.Println("Consumer group session ended")
	return nil
}

// ConsumeClaim 处理消息的核心方法
func (h *messageHandler) ConsumeClaim(
	session sarama.ConsumerGroupSession,
	claim sarama.ConsumerGroupClaim,
) error {
	for {
		select {
		case msg, ok := <-claim.Messages():
			if !ok {
				return nil
			}
			
			// ========== 关键：先处理，再标记 ==========
			
			// 1. 业务处理（幂等操作）
			err := h.processMessage(msg)
			if err != nil {
				// 处理失败：记录日志，不要提交 offset（会重试）
				log.Printf("Process failed, will retry: err=%v offset=%d", 
					err, msg.Offset)
				continue // 不调用 session.MarkMessage()，下次还会收到
			}
			
			// 2. 处理成功，标记已消费
			// MarkMessage 不是立即提交，只是一个 offset 标记
			session.MarkMessage(msg, "")
			
			// 3. 可选：手动提交（Kafka 默认自动提交，但自动提交有延迟）
			session.Commit()
			
			log.Printf("Message processed: topic=%s partition=%d offset=%d", 
				msg.Topic, msg.Partition, msg.Offset)
				
		case <-session.Context().Done():
			return nil
		}
	}
}

func (h *messageHandler) processMessage(msg *sarama.ConsumerMessage) error {
	// 模拟业务处理
	// 实际项目中这里做幂等检查（查 DB / Redis）
	return nil
}

// 启动消费者
func StartConsumer(brokers []string, groupID string) error {
	config := sarama.NewConfig()
	
	// 关闭自动提交，改为手动提交
	config.Consumer.Offsets.AutoCommit.Enable = false
	config.Consumer.Offsets.Initial = sarama.OffsetNewest
	
	// 设置 Rebalance 策略
	config.Consumer.Group.Rebalance.GroupStrategies = []sarama.BalanceStrategy{
		sarama.NewBalanceStrategyRoundRobin(),
	}
	
	// 心跳超时（超过这个时间没心跳，Broker 认为 Consumer 挂了）
	config.Consumer.Heartbeat.Interval = 3 * time.Second
	config.Consumer.Session.Timeout = 10 * time.Second
	
	ctx, cancel := context.WithCancel(context.Background())
	
	client, err := sarama.NewConsumerGroup(brokers, groupID, config)
	if err != nil {
		cancel()
		return fmt.Errorf("create consumer group failed: %w", err)
	}
	
	handler := &messageHandler{
		processingFn: func(msg *sarama.ConsumerMessage) error {
			return nil // 业务逻辑
		},
	}
	
	consumer := &ReliableConsumer{
		client:  client,
		topics:  []string{"my-topic"},
		handler: handler,
		ctx:     ctx,
		cancel:  cancel,
	}
	
	go func() {
		for {
			// ConsumerGroup 会自动处理 rebalance
			if err := client.Consume(ctx, []string{"my-topic"}, handler); err != nil {
				log.Printf("Consume error: %v", err)
			}
		}
	}()
	
	return nil
}
```

---

## 4. Exactly-Once 语义

### Kafka 的 Exactly-Once 实现

Kafka 0.11+ 支持**事务**（Transaction），实现 Exactly-Once：

```go
// Kafka 事务：Producer 原子性发送 + Consumer 原子性提交 offset
package main

import (
	"github.com/IBM/sarama"
)

func main() {
	config := sarama.NewConfig()
	
	// 开启事务
	config.Producer.TransactionalID = "my-transactional-id"  // 需要唯一 ID
	config.Producer.Idempotent = true
	config.Producer.Return.Successes = true
	
	producer, _ := sarama.NewProducer(config)
	
	// 开启事务
	err := producer.BeginTransaction()
	if err != nil {
		panic(err)
	}
	
	// 发送业务消息
	producer.Input() <- &sarama.ProducerMessage{
		Topic: "payment-events",
		Value: sarama.StringEncoder("payment-success"),
	}
	
	// 发送 offset 提交消息（特殊 Topic）
	// 在一个事务里同时包含业务消息和 offset，原子提交
	producer.Input() <- &sarama.ProducerMessage{
		Topic: "__consumer_offsets",
		Value: sarama.StringEncoder(offsetCommit),
	}
	
	// 提交事务
	if err := producer.CommitTransaction(); err != nil {
		producer.AbortTransaction()
	}
}
```

### 三种 Exactly-Once 场景

| 场景 | Kafka 支持 | 说明 |
|------|-----------|------|
| Producer → Kafka | ✅ 幂等生产者 / 事务 | 发送端不重复 |
| Kafka → External System | ⚠️ 需要外部幂等 | 需要业务端配合 |
| Kafka → Kafka | ✅ 事务 | 跨 Topic 原子写入 |

---

## 面试话术

**Q：如何保证 Kafka 消息不丢？**

> 三端都要做：1）**Producer 端**：acks=all + 幂等生产者 + 重试机制；2）**Broker 端**：replication.factor=3 + min.insync.replicas=2 + unclean.leader.election=false；3）**Consumer 端**：关闭自动提交，手动在业务处理完成后提交 offset。三端缺一不可。

**Q：Kafka 的高可靠和低延迟如何权衡？**

> acks=all 可靠性最高但延迟大，因为要等所有 ISR 同步。可以：1）增加 batch.size 和 linger.ms，批量发送减少 RTT；2）用异步发送 + 回调；3）选择同机房部署减少网络延迟；4）适当降低 replication.factor 到 2（代价是容忍度降低）。

**Q：消费者重复消费怎么解决？**

> 业务层面做幂等：1）数据库唯一索引（user_id + 业务主键）；2）Redis 去重（SETNX）；3）版本号乐观锁。关键是想清楚"处理失败怎么办"——At-Least-Once 允许重复，但不允许丢失。
