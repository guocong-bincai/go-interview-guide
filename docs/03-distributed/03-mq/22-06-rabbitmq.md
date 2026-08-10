# RabbitMQ 深度剖析：与 Kafka/RocketMQ 的核心差异

> 考察频率：★★★☆☆  难度：★★★☆☆
> 关键词：AMQP、Exchange、Queue、Binding、Publisher Confirm、Consumer ACK、镜像队列

## 核心定位

| 对比项 | RabbitMQ | Kafka | RocketMQ |
|--------|----------|-------|----------|
| 定位 | **消息代理**（智能路由） | **分布式日志系统**（高吞吐日志/流） | **金融级消息平台**（事务敏感） |
| 协议 | AMQP 0-9-1 / STOMP / MQTT / HTTP | 自定义二进制协议 | 自定义协议 |
| 模型 | Exchange → Binding → Queue | Partition + Offset | Topic/Queue 混合 |
| 顺序消息 | ✅ 单一 Queue 内有序 | ✅ 单 Partition 有序 | ✅ 队列有序 |
| 延迟消息 | ✅ 插件支持 | ❌ 需插件 | ✅ 原生延迟级别 |
| 事务消息 | ❌ 仅局部事务 | ✅ 事务（0.11+） | ✅ 原生两阶段提交 |
| 消息堆积 | 百万级（内存受限） | **亿级**（磁盘无限） | 亿级 |
| 单机吞吐 | ~5万/秒 | ~100万/秒 | ~10万/秒 |
| 延迟 | **毫秒级（最低）** | 微秒级 | 毫秒级 |
| 消息回溯 | ❌ 不支持 | ✅ 按 Offset 回溯 | ✅ 按时间回溯 |

**RabbitMQ 的核心优势**：低延迟、灵活的路由规则（Exchange+Banding）、支持消息优先级、死信队列、多协议接入，适合**异步任务调度、快速消息传递**等场景。

---

## 核心概念：Exchange 路由模型

RabbitMQ 与 Kafka/RocketMQ 最大的架构差异在于**路由层抽象**。

```
Kafka:   Producer ──▶ Topic/Partition ──▶ Consumer Group
RocketMQ: Producer ──▶ Topic ──▶ Queue ──▶ Consumer

RabbitMQ: Producer ──▶ Exchange ──▶ Binding ──▶ Queue ──▶ Consumer
                              (路由规则)    (绑定键)
```

### 四种 Exchange 类型

```go
// RabbitMQ 的 Exchange 类型决定了路由策略

// 1. Direct Exchange：精确匹配 Binding Key
//    路由键 = Binding Key 时，消息进入对应 Queue
exchange := "order.direct"
routingKey := "order.created"  // 必须完全匹配

// 2. Fanout Exchange：广播，所有绑定的 Queue 都收到
//    类似 RocketMQ 的广播模式
exchange := "notification.fanout"

// 3. Topic Exchange：通配符匹配
//    * 匹配一个单词，# 匹配零个或多个单词
//    routingKey: "stock.inventory.created" → binding: "stock.*.created"
routingKey := "user.order.paid"
bindingKey := "user.*.paid"

// 4. Headers Exchange：根据消息头部属性匹配（较少用）
```

### 完整路由流程

```
  Producer
     │
     │  publish(exchange="order", routing_key="order.created", body)
     ▼
┌─────────────┐
│   Exchange  │──Binding(routing_key="order.*")──▶ Queue(order.created)
│ order.direct│──Binding(routing_key="order.#")──▶ Queue(order.all)
└─────────────┘
     │
     │  找不到匹配 → 消息丢弃（除非配置 DLX）
     ▼
  (黑洞)
```

---

## RabbitMQ 核心机制

### 1. 消息确认机制（ACK）

RabbitMQ 有**三种 ACK 模式**，精细控制消息可靠性：

```go
// RabbitMQ ACK 模式
type AckMode int

const (
    // 1. AutoAck = true（默认）：消费者收到消息时自动确认
    //    问题：处理失败时消息丢失，无法重试
    channel.Consume(queue, "", false, false, false, false, nil)

    // 2. Manual ACK（推荐）：手动确认，处理成功后 ACK
    msgs, err := channel.Consume(
        queue,
        "",    // consumer tag
        false, // autoAck = false（手动模式）
        false, // exclusive
        false, // noLocal
        false, // noWait
        nil,   // args
    )

    go func() {
        for msg := range msgs {
            err := process(msg.Body)
            if err != nil {
                // NACK：拒绝消息，requeue=true 表示重新入队
                // 注意：可能导致无限循环
                channel.BasicNack(msg.DeliveryTag, false, true)
            } else {
                // ACK：确认消息已被正确处理
                channel.BasicAck(msg.DeliveryTag, false)
            }
        }
    }()

    // 3. AckMultiple：批量确认
    //    channel.BasicAck(tag, true) → 确认所有 ≤ tag 的消息
}()
```

**与 Kafka 的对比**：
- Kafka：按 Offset 提交，偏移量即确认
- RabbitMQ：按 DeliveryTag 确认，消息可**被消费者选择性拒绝（NACK）**

### 2. Publisher Confirm（生产者确认）

RabbitMQ 的生产者确认类似 Kafka 的 `acks=all`：

```go
// 开启 Publisher Confirm
channel.Confirm(false) // false = noWait 模式

confirmChan := channel.NotifyPublish(make(chan amqp.Confirmation, 1))

// 发布消息
err = channel.PublishWithContext(
    ctx,
    "order.direct",    // exchange
    "order.created",   // routing key
    false,             // mandatory（找不到队列则返回）
    false,             // immediate（找不到消费者则返回）
    amqp.Publishing{
        DeliveryMode: amqp.Persistent, // 持久化
        ContentType:  "application/json",
        Body:         body,
        MessageId:    msgId,
        Timestamp:   time.Now(),
    },
)
if err != nil {
    log.Printf("Publish failed: %v", err)
}

// 等待 Broker 确认
confirmation := <-confirmChan
if !confirmation.Ack {
    // 消息未被确认，需要重试
    log.Printf("Message %d not acknowledged: reason=%v",
        confirmation.SequenceId, confirmation.Delivery)
    // 重新发布或记录到本地重试队列
}
```

### 3. 消费模式：推（Push）vs 拉（Pull）

```go
// 推模式（Push）：Broker 主动推送，比 Kafka 更实时
// channel.Consume = 持续推送，默认预取 1 条
msgs, err := channel.Consume(
    "order-queue",
    "",    // consumer tag
    false, // autoAck
    false, // exclusive
    false, // noLocal
    false, // noWait
    nil,
)

// 拉模式（Pull）：消费者主动拉取，类似 Kafka 的 poll
// channel.Get = 单次拉取
msg, ok, err := channel.Get("order-queue", false)
if ok {
    process(msg.Body)
    channel.BasicAck(msg.DeliveryTag, false)
}
```

### 4. 预取（Prefetch）控制并发

```go
// 设置 QoS：消费者最多同时"持有" N 条未确认的消息
// 作用：控制并发消费数量，防止内存溢出
err = channel.Qos(
    10,    // prefetch count = 10（最多 10 条消息不 ACK）
    0,     // prefetch size（字节，0=无限制）
    false, // global（false=只影响当前消费者）
)

// 对比 Kafka 的 max.poll.records
// Kafka: max.poll.records=500（一次拉取 N 条）
// RabbitMQ: prefetch=10（最多 N 条未处理）
```

### 5. 死信队列（DLX/DLQ）

```go
// 消息被拒绝、过期或队列满时，进入死信队列
args := amqp.Table{
    "x-dead-letter-exchange":    "dlx.exchange",
    "x-dead-letter-routing-key": "dlx.order",
}

// 定义死信交换机和队列
channel.ExchangeDeclare("dlx.exchange", "direct", true, false, false, false, nil)
channel.QueueDeclare("dlx.order.queue", true, false, false, false, nil)
channel.QueueBind("dlx.order.queue", "dlx.exchange", "dlx.order", false, nil)

// 生产者发布消息
channel.Publish("order.direct", "order.created", false, false, amqp.Publishing{
    Body:      body,
    Expiration: "60000", // TTL=60秒，过期后进入 DLQ
})
```

---

## 镜像队列（高可用）

RabbitMQ 集群有两种模式：**普通集群**和**镜像队列**。

```properties
# RabbitMQ 镜像策略配置
# /etc/rabbitmq/rabbitmq.conf

# 镜像到所有节点（最高可靠性，但性能损耗）
ha-mode: all

# 镜像到指定数量的节点
ha-mode: exactly
ha-params: 3

# 同步模式：手动（默认）/ 自动
ha-sync-mode: manual  # 手动同步，新镜像需要显式同步
ha-sync-mode: automatic # 自动同步
```

```
普通集群：队列只在一个节点，消息分布在多个节点
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Node A  │  │ Node B  │  │ Node C  │
│ [Queue] │  │ (元数据) │  │ (元数据) │
│  actual │  │         │  │         │
└─────────┘  └─────────┘  └─────────┘

镜像队列：队列内容同步到多个节点
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Node A  │  │ Node B  │  │ Node C  │
│[Queue]  │  │[Mirror] │  │[Mirror] │
│ primary │  │ replica │  │ replica │
└─────────┘  └─────────┘  └─────────┘
```

---

## Go + RabbitMQ 完整示例

### 安装

```bash
go get github.com/rabbitmq/amqp091-go
```

### 完整生产消费示例

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    amqp "github.com/rabbitmq/amqp091-go"
)

const (
    URL      = "amqp://guest:guest@localhost:5672/"
    Exchange = "order.direct"
    Queue    = "order.created.queue"
    RoutingKey = "order.created"
)

// setup: 声明交换机、队列、绑定
func setup(channel *amqp.Channel) error {
    // 1. 声明 Direct Exchange
    err := channel.ExchangeDeclare(
        Exchange, // name
        "direct",  // type
        true,      // durable（持久化）
        false,     // auto-deleted
        false,     // internal
        false,     // no-wait
        nil,       // arguments
    )
    if err != nil {
        return fmt.Errorf("exchange declare: %w", err)
    }

    // 2. 声明 Queue
    _, err = channel.QueueDeclare(
        Queue, // name
        true,  // durable
        false, // delete when unused
        false, // exclusive（排他队列，只允许当前连接）
        false, // no-wait
        amqp.Table{
            "x-dead-letter-exchange":    "dlx.exchange",
            "x-dead-letter-routing-key": "dlx.order",
        },
    )
    if err != nil {
        return fmt.Errorf("queue declare: %w", err)
    }

    // 3. 绑定 Queue 到 Exchange
    err = channel.QueueBind(
        Queue,      // queue name
        RoutingKey, // routing key
        Exchange,   // exchange
        false,
        nil,
    )
    return err
}

func main() {
    // 连接
    conn, err := amqp.Dial(URL)
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    channel, err := conn.Channel()
    if err != nil {
        log.Fatal(err)
    }
    defer channel.Close()

    // 开启 Publisher Confirm
    channel.Confirm(false)
    confirmChan := channel.NotifyPublish(make(chan amqp.Confirmation, 1))
    channel.NotifyPublishConfirm(confirmChan)

    // Setup
    if err := setup(channel); err != nil {
        log.Fatal(err)
    }

    // 生产者
    go func() {
        for i := 0; i < 10; i++ {
            body := fmt.Sprintf(`{"order_id":"order-%d","amount":%d}`, i, i*100)
            ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
            defer cancel()

            err := channel.PublishWithContext(
                ctx,
                Exchange,    // exchange
                RoutingKey,  // routing key
                false,       // mandatory
                false,       // immediate
                amqp.Publishing{
                    DeliveryMode: amqp.Persistent,
                    ContentType:  "application/json",
                    Body:         []byte(body),
                    MessageId:    fmt.Sprintf("msg-%d", i),
                    Timestamp:    time.Now(),
                },
            )
            if err != nil {
                log.Printf("Publish failed: %v", err)
            }

            // 等待确认
            confirm := <-confirmChan
            if !confirm.Ack {
                log.Printf("Message %d not confirmed", i)
            }

            time.Sleep(100 * time.Millisecond)
        }
    }()

    // 消费者
    // 设置 QoS（预取 10 条）
    channel.Qos(10, 0, false)

    msgs, err := channel.Consume(
        Queue,  // queue
        "",     // consumer tag
        false,  // autoAck = false（手动确认）
        false,  // exclusive
        false,  // noLocal
        false,  // noWait
        nil,
    )
    if err != nil {
        log.Fatal(err)
    }

    go func() {
        for msg := range msgs {
            orderID := string(msg.Body)
            fmt.Printf("Processing: %s\n", orderID)

            // 模拟处理
            time.Sleep(200 * time.Millisecond)

            // 手动 ACK
            if err := msg.Ack(false); err != nil {
                log.Printf("ACK failed: %v", err)
            }
        }
    }()

    fmt.Println("RabbitMQ demo running, press Ctrl+C to exit")
    time.Sleep(30 * time.Second)
}
```

---

## 延迟消息实现

RabbitMQ 原生不支持延迟消息（类似 RocketMQ 的延迟级别），但可以通过**插件**或**TTL+DLX**模拟：

### 方案一：TTL + DLX（不精确）

```go
// 利用消息 TTL + 死信队列实现延迟
// 注意：这是队列级 TTL，所有消息相同延迟

// 1. 创建延迟队列（TTL=10秒）
delayQueue, err := channel.QueueDeclare(
    "order.delay.queue", // name
    true, false, false, false,
    amqp.Table{
        "x-message-ttl":             int64(10000), // 10秒 TTL
        "x-dead-letter-exchange":    "",
        "x-dead-letter-routing-key": "order.created.queue", // 到期后回到原队列
    },
)

// 2. 原队列绑定到延迟队列
// 消息先发到 delayQueue → 10秒后自动路由到 order.created.queue
```

### 方案二：rabbitmq-delayed-message-exchange 插件（精确）

```bash
# 安装延迟消息插件
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# 生产者
channel.ExchangeDeclare("order.delayed", "x-delayed-message", true, false, false, false,
    amqp.Table{
        "x-delayed-type": "direct",
    },
)

// 发布消息（headers 指定延迟时间）
channel.Publish("order.delayed", "order.created", false, false, amqp.Publishing{
    Headers: amqp.Table{
        "x-delay": int32(5000), // 延迟 5 秒（毫秒级精度）
    },
    Body: []byte("order to process in 5 seconds"),
})
```

---

## 常见问题与解决方案

### 消息堆积

**原因**：消费者速度 < 生产速度，或消费者宕机

**排查**：
```bash
# 查看队列状态
rabbitmqctl list_queues name messages messages_ready messages_unacknowledged
```

**解决**：
1. 增加消费者实例（扩展并发）
2. 消费者开启 Prefetch 并行处理
3. 配置惰性队列（Lazy Queue）将消息存入磁盘

```go
// 惰性队列：消息直接落盘，减少内存压力
channel.QueueDeclare("lazy.queue", true, false, false, false,
    amqp.Table{"x-queue-mode": "lazy"},
)
```

### 消息丢失

| 环节 | 原因 | 解决方案 |
|------|------|---------|
| 生产者 | 网络抖动 | Publisher Confirm + 重试 |
| Broker | 单机故障 | 镜像队列（HA） |
| 消费者 | AutoAck 后处理失败 | 改为 Manual ACK |

```go
// 确保可靠性：持久化 + Confirm + Manual ACK
channel.Confirm(false)
channel.Publish("", "order.queue", false, false, amqp.Publishing{
    DeliveryMode: amqp.Persistent, // 消息持久化
})
// 消费者手动确认
msg.Ack(false)
```

### 重复消费

与 Kafka 类似，RabbitMQ **不保证 exactly-once**，需要消费者实现幂等：

```go
// Redis 幂等表
func processWithIdempotency(msg amqp.Delivery) error {
    key := "processed:" + msg.MessageId
    ok, _ := redis.SetNX(key, "1", 24*time.Hour).Result()
    if !ok {
        return nil // 已处理过
    }
    return doProcess(msg.Body)
}
```

---

## 面试话术

**Q：RabbitMQ 与 Kafka 的核心区别是什么？**

> 本质区别在于架构定位。Kafka 是分布式日志系统，以 Partition 为核心，消息有序、持久化到磁盘、堆积能力强，适合大数据流处理。RabbitMQ 是智能消息代理，以 Exchange 路由为核心，支持多种路由规则（Direct/Fanout/Topic），**延迟更低**，但堆积能力受内存限制。
>
> 选型原则：日志采集、实时流处理选 Kafka；**低延迟任务调度、快速消息传递**选 RabbitMQ；金融级分布式事务选 RocketMQ。

**Q：RabbitMQ 如何保证消息不丢失？**

> 三层保证：① 生产者开启 Publisher Confirm，确保消息被 Broker 持久化存储；② Broker 配置镜像队列，数据在多个节点冗余；③ 消费者关闭 AutoAck，改为手动 ACK，处理成功后才确认。这样即使最坏情况（消费者宕机），消息也不会丢，因为未被 ACK 的消息会重新投递给其他消费者。

[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [💎 消息队列](../README.md)
