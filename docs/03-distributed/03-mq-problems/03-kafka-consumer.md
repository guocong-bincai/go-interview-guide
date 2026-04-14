# Kafka 消费者与 Rebalance

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：消费者组、Rebalance、顺序消费、分区分配策略

## 消费者组（Consumer Group）

### 核心概念

```
消费者组：多个 Consumer 实例组成，共同消费一个 Topic

Topic: order-events
  Partition 0 ──────────────────────────────────┐
  Partition 1 ───────────────────────┐          │
  Partition 2 ───────────┐          │          │
                        │          │          │
              Group A:  │          │          │
              Consumer 1 ←──────────┘          │
              Consumer 2 ←─────────────────────┘

原理：
- 一个 Partition 只能被同组的同一个 Consumer 消费
- 不同 Consumer Group 可以重复消费同一个 Partition
- Consumer 数量 > Partition 数量时，多余的 Consumer 空闲
```

### Go 消费者组示例

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/IBM/sarama"
)

type ConsumerGroupHandler struct {
	name string
}

func (h *ConsumerGroupHandler) Setup(sarama.ConsumerGroupSession) error {
	fmt.Printf("[%s] Session setup\n", h.name)
	return nil
}

func (h *ConsumerGroupHandler) Cleanup(sarama.ConsumerGroupSession) error {
	fmt.Printf("[%s] Session cleanup\n", h.name)
	return nil
}

func (h *ConsumerGroupHandler) ConsumeClaim(
	session sarama.ConsumerGroupSession,
	claim sarama.ConsumerGroupClaim,
) error {
	for {
		select {
		case msg, ok := <-claim.Messages():
			if !ok {
				return nil
			}
			
			fmt.Printf("[%s] Received: partition=%d offset=%d key=%s value=%s\n",
				h.name, msg.Partition, msg.Offset, msg.Key, msg.Value)
			
			// 业务处理
			processMessage(msg)
			
			// 标记已消费
			session.MarkMessage(msg, "")
			
		case <-session.Context().Done():
			return nil
		}
	}
}

func processMessage(msg *sarama.ConsumerMessage) error {
	// 模拟处理
	return nil
}

func main() {
	brokers := []string{"localhost:9092"}
	groupID := "order-processor"
	topic := "order-events"

	config := sarama.NewConfig()
	config.Consumer.Group.Rebalance.GroupStrategies = []sarama.BalanceStrategy{
		// 策略选择见下文
		sarama.NewBalanceStrategyRoundRobin(), // 轮询分配
	}
	config.Consumer.Offsets.Initial = sarama.OffsetNewest

	ctx, cancel := context.WithCancel(context.Background())
	client, err := sarama.NewConsumerGroup(brokers, groupID, config)
	if err != nil {
		log.Fatal(err)
	}

	handler := &ConsumerGroupHandler{name: "consumer-1"}

	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)

	go func() {
		for {
			if err := client.Consume(ctx, []string{topic}, handler); err != nil {
				log.Printf("Consume error: %v", err)
			}
			if ctx.Err() != nil {
				return
			}
		}
	}()

	<-sigChan
	cancel()
	client.Close()
}
```

---

## Rebalance（再均衡）

### 触发条件

| 触发条件 | 说明 |
|---------|------|
| Consumer 加入 | 新 Consumer 加入组，触发 Rebalance |
| Consumer 离开 | Consumer 主动关闭或崩溃 |
| Partition 数量变化 | Topic 分区数增加 |
| Broker 故障 | 分区 Leader 迁移 |

### Rebalance 过程

```
消费者加入/离开
      ↓
Coordinator（Broker）通知所有 Consumer 停止消费
      ↓
Consumer 停止消费，等待分区分配
      ↓
Coordinator 计算新的分区分配方案
      ↓
Coordinator 通知所有 Consumer 新的分配方案
      ↓
Consumer 开始消费
```

### 分区分配策略

```go
// 三种分配策略对比

// 1. Range（默认）
// 每个 Consumer 连续分若干个分区，可能不均匀
// Topic: 3 partitions → Consumer A: [0], Consumer B: [1,2]
// 缺点：Consumer 多的组可能导致某些 Consumer 分到更多分区

// 2. RoundRobin（推荐大多数场景）
// 将所有 Topic 的所有分区混合后轮询分配
// 更均匀

// 3. StickyAssignor
// 保持原有分配，只迁移必要的分区（减少 Rebalance 影响）
// Kafka 2.4+ 默认

// 配置示例
config.Consumer.Group.Rebalance.GroupStrategies = []sarama.BalanceStrategy{
	sarama.NewBalanceStrategySticky(), // 粘性分配
}
```

### Rebalance 问题与优化

**问题：Rebalance 期间服务不可用（Stop The World）**

```go
// 优化1：增加心跳间隔，减少意外 Rebalance
config.Consumer.Heartbeat.Interval = 10 * time.Second  // 原3s
config.Consumer.Session.Timeout = 30 * time.Second     // 原10s
config.Consumer.MaxWaitTime = 500 * time.Millisecond

// 优化2：设置合理的 Rebalance 超时
config.Consumer.Group.Rebalance.Timeout = 60 * time.Second

// 优化3：使用 StickyAssignor 减少分区迁移
```

---

## 顺序消费

### Kafka 顺序消费的约束

```
顺序保证只在单个 Partition 内有序
跨 Partition 无序

要保证全局顺序：
  方案1：Topic 只设置 1 个 Partition（但失去并行能力）
  方案2：按消息 Key 哈希到固定 Partition（相同 Key 有序）
```

### Go 保证顺序消费示例

```go
package main

import (
	"fmt"

	"github.com/IBM/sarama"
)

// 按用户 ID 保证顺序：相同 userID 的消息一定路由到同一分区
type UserOrderProducer struct {
	producer sarama.SyncProducer
}

func (p *UserOrderProducer) SendOrder(userID string, orderID string) error {
	msg := &sarama.ProducerMessage{
		Topic: "user-orders",
		Key:   sarama.StringEncoder(userID), // Key 决定分区
		Value: sarama.StringEncoder(orderID),
	}
	
	partition, _, err := p.producer.SendMessage(msg)
	if err != nil {
		return err
	}
	
	fmt.Printf("Order %s for user %s sent to partition %d\n", 
		orderID, userID, partition)
	return nil
}

// 消费者：按分区消费，保证同一用户的订单有序
func consumeUserOrders(client sarama.ConsumerGroup, topic string) {
	// 每个分区一个 Consumer，保证有序
	client.Consume()
}
```

---

## 消费者 offset 管理

### offset 存储位置

```
旧版（__consumer_offsets Topic）：
  ZooKeeper 管理 offset（已废弃）

新版（Kafka 内部 Topic）：
  __consumer_offsets 自动管理
  默认 50 个分区，ReplicationFactor = 3
```

### offset 提交策略

```go
// 自动提交（默认，但可能有重复消费）
config.Consumer.Offsets.AutoCommit.Enable = true
config.Consumer.Offsets.AutoCommit.Interval = 5 * time.Second  // 5s 提交一次

// 手动提交（精确控制，推荐）
config.Consumer.Offsets.AutoCommit.Enable = false

// 手动同步提交
handler := &ConsumerGroupHandler{}
client.Consume(ctx, topics, handler)
client.CommitOffsets()

// 手动异步提交（高吞吐）
go func() {
	for {
		select {
		case <-ticker.C:
			if err := client.CommitOffsets(); err != nil {
				log.Printf("Commit failed: %v", err)
			}
		}
	}
}()
```

---

## 常见问题与排查

### 消费者 lag 过大

```bash
# 查看消费滞后
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --describe

# 输出示例：
# GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# my-group        order-events    0          5000            10000           5000  ← lag 过大

# 原因排查：
# 1. Consumer 消费速度慢 → Profiler 看 CPU / 检查处理逻辑
# 2. Consumer 实例太少 → 增加 Consumer 数量（上限是 Partition 数）
# 3. 分区分配不均 → 检查分区分配策略
```

### 重复消费的解决方案

```go
// 业务层幂等
type OrderProcessor struct {
	db DB
}

func (p *OrderProcessor) Process(msg *sarama.ConsumerMessage) error {
	// 1. 提取业务 ID
	orderID := extractOrderID(msg.Value)
	
	// 2. 幂等检查：Redis SETNX
	ok, err := p.db.Setnx(ctx, "processed:"+orderID, 1, 24*time.Hour)
	if err != nil {
		return err
	}
	if !ok {
		log.Printf("Already processed: %s", orderID)
		return nil // 已处理过，跳过
	}
	
	// 3. 业务处理
	return p.db.CreateOrder(orderID)
}
```

---

## 面试话术

**Q：消费者组内 Consumer 数量和 Partition 数量应该是什么关系？**

> Consumer 数量**建议等于或小于 Partition 数量**。Consumer 多于 Partition 会导致多余的 Consumer 空闲。最佳配置：Partition 数 = N × Consumer 数（每个 Consumer 处理 N 个分区）。如果消费慢，可以增加分区和 Consumer（但 Partition 数定了就不能减少，只能增加）。

**Q：Rebalance 时消息会丢失吗？**

> 不会丢失，但会有短暂不可用。Rebalance 期间 Consumer 停止拉取消息，Rebalance 完成后从新的 offset 继续消费。如果 Consumer 在处理消息时崩溃，消息会重新投给其他 Consumer（因为 offset 还没提交）。所以要确保**先处理消息，再提交 offset**。

**Q：如何保证消息不被重复消费？**

> 三个层次：1）Producer 端开启幂等生产者（Kafka 0.11+）；2）Consumer 端先处理消息，再提交 offset；3）业务层做幂等（数据库唯一索引 / Redis 去重 key）。Kafka 天然支持 At-Least-Once 语义，允许少量重复但不允许丢失。
