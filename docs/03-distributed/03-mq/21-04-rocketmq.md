# RocketMQ vs Kafka

> 考察频率：★★★☆☆  难度：★★★☆☆
> 关键词：事务消息、延迟消息、顺序消息、推/拉模式

## 核心区别

| 对比项 | RocketMQ | Kafka |
|--------|----------|-------|
| 适用场景 | 电商、金融（事务敏感） | 日志、大数据（吞吐优先） |
| 延迟消息 | ✅ 原生支持 | ❌ 需插件 |
| 事务消息 | ✅ 原生支持 | ⚠️ 需 0.11+ 事务 |
| 顺序消息 | ✅ 支持 | ⚠️ 单 Partition 有序 |
| 消费模式 | 推/拉都行 | 拉（长轮询模拟推） |
| 单机吞吐量 | ~10万/秒 | ~100万/秒 |
| 延迟 | 毫秒级 | 微秒级（更低） |
| 消息堆积 | 支持亿级 | 支持亿级 |
| 生态 | 阿里巴巴深度定制 | Apache 顶级项目 |

---

## RocketMQ 核心概念

```
RocketMQ 架构：
  NameServer Cluster（无状态，服务发现）
      ↓
  Broker Cluster（消息存储）
      ├── Master（处理读写）
      └── Slave（热备）
      ↓
  Producer（生产者，可多实例）
  Consumer（消费者，支持集群/广播）
```

### 消息类型

| 类型 | 说明 | 典型场景 |
|------|------|---------|
| 普通消息 | 无特殊语义的普通消息 | 通用异步处理 |
| 顺序消息 | 按分区/队列有序 | 订单流程 |
| 事务消息 | 两阶段提交 | 扣款 + 发货 |
| 延迟消息 | 指定时间后才可消费 | 订单超时取消 |

---

## Go + RocketMQ 示例

### 安装

```bash
go get github.com/apache/rocketmq-client-go/v2
```

### 普通消息

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/apache/rocketmq-client-go/v2"
	"github.com/apache/rocketmq-client-go/v2/consumer"
	"github.com/apache/rocketmq-client-go/v2/primitive"
	"github.com/apache/rocketmq-client-go/v2/producer"
)

func main() {
	// 需要先启动 NameServer: mqnamesrv
}

// 生产者
func producerExample() {
	p, err := rocketmq.NewProducer(producer.WithNsResolver(primitive.NewPassthroughResolver([]string{"127.0.0.1:9876"})))
	if err != nil {
		log.Fatal(err)
	}
	
	if err := p.Start(); err != nil {
		log.Fatal(err)
	}
	
	msg := &primitive.Message{
		Topic: "order-topic",
		Body:  []byte("order created: order-123"),
	}
	
	// 设置 keys 和 tags
	msg.WithKeys([]string{"order-123"})
	msg.WithTag("order")
	
	result, err := p.SendSync(context.Background(), msg)
	if err != nil {
		log.Printf("Send failed: %v", err)
	} else {
		fmt.Printf("Send OK: msgID=%s\n", result.MsgID)
	}
	
	p.Shutdown()
}

// 消费者
func consumerExample() {
	c, err := rocketmq.NewPushConsumer(
		consumer.WithNsResolver(primitive.NewPassthroughResolver([]string{"127.0.0.1:9876"})),
		consumer.WithGroupName("order-consumer-group"),
		consumer.WithSubscription(map[string]string{
			"order-topic": "*",  // 订阅所有消息
		}),
	)
	if err != nil {
		log.Fatal(err)
	}
	
	err = c.Subscribe("order-topic", consumer.MessageSelector{}, func(ctx context.Context, msgs ...*primitive.MessageExt) (consumer.ConsumeResult, error) {
		for _, msg := range msgs {
			fmt.Printf("Received: topic=%s body=%s\n", msg.Topic, string(msg.Body))
		}
		return consumer.ConsumeSuccess, nil
	})
	
	if err != nil {
		log.Fatal(err)
	}
	
	if err := c.Start(); err != nil {
		log.Fatal(err)
	}
	
	time.Sleep(time.Hour)
	c.Shutdown()
}
```

---

## 事务消息（核心差异）

RocketMQ 的**最大优势**，用于解决分布式事务问题。

### 业务场景

```
场景：下单 + 扣库存
- 传统方案：本地事务 + MQ（可能丢消息）
- RocketMQ 方案：事务消息（两阶段提交）

      Producer                RocketMQ                Consumer
         │                        │                       │
         │─── HALF_MSG ──────────▶│                       │
         │◀── ACK ────────────────│                       │
         │                        │                       │
    ┌────┴────┐                   │                       │
    │ 本地事务│                   │                       │
    │ 扣库存  │                   │                       │
    └─────────┘                   │                       │
         │                        │                       │
    成功 ──▶ COMMIT_MSG ─────────▶│─── 投递 ──────────────▶│
         │                        │                       │
    失败 ──▶ ROLLBACK_MSG ───────▶│─── 丢弃               │
         │                        │                       │
```

### Go 事务消息实现

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/apache/rocketmq-client-go/v2"
	"github.com/apache/rocketmq-client-go/v2/primitive"
	"github.com/apache/rocketmq-client-go/v2/producer"
)

type OrderService struct {
	// 模拟数据库
	orders    map[string]bool
	stock     map[string]int
}

func (s *OrderService) createOrder(orderID, productID string, count int) error {
	// 1. 扣库存
	if s.stock[productID] < count {
		return fmt.Errorf("stock insufficient")
	}
	s.stock[productID] -= count
	// 2. 创建订单
	s.orders[orderID] = true
	return nil
}

// TransactionListener 事务监听器
type TransactionListener struct {
	orderService *OrderService
}

func (t *TransactionListener) ExecuteLocalTransaction(msg *primitive.Message) primitive.LocalTransactionState {
	// 1. 解析消息
	orderID := string(msg.Body)
	
	// 2. 执行本地事务（创建订单 + 扣库存）
	err := t.orderService.createOrder(orderID, "product-001", 1)
	if err != nil {
		log.Printf("Local transaction failed: %v", err)
		return primitive.RollbackMessageState
	}
	
	log.Printf("Local transaction success: %s", orderID)
	return primitive.CommitMessageState
}

func (t *TransactionListener) CheckLocalTransaction(ext *primitive.MessageExt) primitive.LocalTransactionState {
	// Broker 未收到提交/回滚，询问事务状态
	// RocketMQ 会自动回查此方法（最大15次）
	orderID := string(ext.Body)
	if t.orderService.orders[orderID] {
		return primitive.CommitMessageState
	}
	return primitive.RollbackMessageState
}

func transactionProducerExample() {
	orderSvc := &OrderService{
		orders: make(map[string]bool),
		stock:  map[string]int{"product-001": 100},
	}
	
	txProducer, err := rocketmq.NewTransactionProducer(
		&TransactionListener{orderService: orderSvc},
		producer.WithNsResolver(primitive.NewPassthroughResolver([]string{"127.0.0.1:9876"})),
	)
	if err != nil {
		log.Fatal(err)
	}
	
	txProducer.Start()
	
	// 发送事务消息
	msg := &primitive.Message{
		Topic: "order-topic",
		Body:  []byte("order-456"),
	}
	
	res, err := txProducer.SendMessageInTransaction(context.Background(), msg)
	if err != nil {
		log.Printf("Send transaction msg failed: %v", err)
	} else {
		fmt.Printf("Transaction msg sent: status=%s msgID=%s\n", res.String(), res.MsgID)
	}
	
	time.Sleep(30 * time.Second)
	txProducer.Shutdown()
}
```

---

## 延迟消息

RocketMQ **原生支持**延迟消息（Kafka 需要插件）。

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/apache/rocketmq-client-go/v2"
	"github.com/apache/rocketmq-client-go/v2/primitive"
	"github.com/apache/rocketmq-client-go/v2/producer"
)

// RocketMQ 延迟级别：1s, 5s, 10s, 30s, 1m, 2m, 3m, 4m, 5m, 6m, 7m, 8m, 9m, 10m, 20m, 30m, 1h, 2h
// 延迟消息会先存入 SCHEDULE_TOPIC_XXXX，等待到期后投递

func delayMessageExample() {
	p, err := rocketmq.NewProducer(producer.WithNsResolver(
		primitive.NewPassthroughResolver([]string{"127.0.0.1:9876"}),
	))
	if err != nil {
		log.Fatal(err)
	}
	p.Start()
	
	msg := &primitive.Message{
		Topic: "order-timeout",
		Body:  []byte("order-123 timeout, please cancel"),
	}
	
	// 设置延迟级别（延迟 30 秒）
	// RocketMQ 支持的延迟级别：1s, 5s, 10s, 30s, 1m, 2m, 3m, 4m, 5m, 6m, 7m, 8m, 9m, 10m, 20m, 30m, 1h, 2h
	msg.WithDelayTimeLevel(4) // 30s
	
	result, err := p.SendSync(context.Background(), msg)
	if err != nil {
		log.Printf("Send failed: %v", err)
	} else {
		fmt.Printf("Delay message sent: msgID=%s, will be delivered after 30s\n", result.MsgID)
	}
	
	p.Shutdown()
}
```

---

## 顺序消息

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/apache/rocketmq-client-go/v2"
	"github.com/apache/rocketmq-client-go/v2/consumer"
	"github.com/apache/rocketmq-client-go/v2/primitive"
	"github.com/apache/rocketmq-client-go/v2/producer"
)

// RocketMQ 顺序消息：按 FIFO 顺序消费
// 通过 MessageQueueSelector 指定消息发到哪个队列
// 同一队列内的消息有序

type OrderStep struct {
	OrderID string
	Step    string
}

func main() {
	// 模拟订单流程
	steps := []OrderStep{
		{"order-001", "create"},
		{"order-001", "pay"},
		{"order-001", "ship"},
		{"order-001", "complete"},
	}
	
	p, _ := rocketmq.NewProducer(producer.WithNsResolver(
		primitive.NewPassthroughResolver([]string{"127.0.0.1:9876"}),
	))
	p.Start()
	
	for _, step := range steps {
		msg := &primitive.Message{
			Topic: "order-flow",
			Body:  []byte(fmt.Sprintf("%s: %s", step.OrderID, step.Step)),
		}
		msg.WithKeys([]string{step.OrderID}) // 相同 OrderID 路由到同一队列
		
		// 自定义选择队列：相同 key 选同一队列
		res, err := p.SendSync(context.Background(), msg)
		if err != nil {
			log.Printf("Send failed: %v", err)
		} else {
			fmt.Printf("Sent: %s to partition %d\n", step.Step, res.Queue.QueueId)
		}
	}
	
	p.Shutdown()
}

// 消费时也要按顺序
func orderConsumerExample() {
	c, _ := rocketmq.NewPushConsumer(
		consumer.WithNsResolver(primitive.NewPassthroughResolver([]string{"127.0.0.1:9876"})),
		consumer.WithGroupName("order-flow-consumer"),
		consumer.WithConsumeOrderly(true), // 开启顺序消费
	)
	
	c.Subscribe("order-flow", consumer.MessageSelector{}, func(ctx context.Context, msgs ...*primitive.MessageExt) (consumer.ConsumeResult, error) {
		for _, msg := range msgs {
			fmt.Printf("[ORDERLY] %s\n", string(msg.Body))
		}
		return consumer.ConsumeSuccess, nil
	})
	
	c.Start()
}
```

---

## 面试话术

**Q：RocketMQ 和 Kafka 怎么选？**

> 核心看业务场景。**金融支付、电商交易**选 RocketMQ，因为有原生事务消息和延迟消息；**日志采集、大数据流处理**选 Kafka，因为吞吐量更高、生态更成熟。阿里双十一用的就是 RocketMQ，扛得住高并发。如果只是做异步解耦，Kafka 足够；如果需要事务一致性（如下单后发消息），RocketMQ 更合适。

**Q：RocketMQ 事务消息的原理？**

> 两阶段提交 + 补偿机制。第一阶段：Producer 发 HALF_MSG（预投递），本地执行事务，提交或回滚；第二阶段：Broker 根据提交状态决定投递还是丢弃。如果 Broker 长时间没收到最终状态，会自动回查 Producer 的 `CheckLocalTransaction` 方法，Producer 在这里查数据库确认事务状态（最大回查15次，间隔递增）。这样就保证了"本地事务成功，消息一定投递；本地事务失败，消息一定不投递"。

**Q：RocketMQ 延迟消息的原理？**

> 延迟消息不会立即投递给消费者，而是先存到 `SCHEDULE_TOPIC_XXXX` 系统 Topic，等延迟时间到了再投递给目标消费者。实现原理：每个延迟级别对应一个队列，Broker 内部有一个定时器定期检查消息是否到期，到期后从延迟队列移动到真实目标队列。这种方式比 Kafka 插件（时间轮）更简单，但延迟级别是固定的18个等级，不支持任意精确延迟。
