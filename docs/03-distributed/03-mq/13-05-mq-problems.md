# 消息队列问题排查

> 考察频率：★★★★☆  难度：★★★☆☆

## 三大经典问题

| 问题 | 后果 | 根因 |
|------|------|------|
| 消息积压 | 延迟、OOM | 生产 > 消费 |
| 重复消费 | 数据重复 | 消费幂等未保证 |
| 消息丢失 | 数据丢失 | 发送/存储/消费失败 |

---

## 消息积压

### 表现

- 消费延迟持续增长
- 队列水位线持续上升
- 消费者 CPU 100%，但队列不下降

### 排查步骤

```bash
# 查看 Kafka 消费 lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --describe

# 输出示例
TOPIC     PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
my-topic  0          5000             10000           5000
```

### 原因分析

| 原因 | 表现 | 解决方案 |
|------|------|---------|
| 消费者慢 | 消费时间长 | 增加消费者、优化逻辑 |
| 消费者挂 | lag 持续增长 | 检查消费者进程 |
| 消费者少 | 资源空闲 | 增加消费者实例 |
| 生产太快 | 超出处理能力 | 限流、批量消费 |

### 应急处理

```go
// 1. 紧急扩容消费者
// 2. 消费者开启批量消费
func batchConsume(msgs []string) {
    for _, msg := range msgs {
        process(msg)
    }
}

// 3. 消息快进（丢弃旧消息，仅处理最新）
// 适用于只关心最新状态的场景
```

### 预防措施

```go
// 配置合理的消费者数量
// 消费者数量 = Topic 分区数（最多）
// 或 消费者数量 = Topic 分区数 × 机器数 / 单机消费能力

// 监控告警
if lag > threshold {
    alert("消费积压！lag={}", lag)
}
```

---

## 重复消费

### 原因

1. **消费者消费后，处理成功但 ack 失败**
2. **Rebalance 期间重复投递给其他消费者**
3. **消息队列本身 at-least-once 语义**

### 幂等设计

必须保证**消费幂等**，即使同一条消息被消费多次，结果一致。

#### 方案一：数据库唯一键

```go
func consumeWithUniqueKey(msg *Message) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }

    // INSERT ... ON DUPLICATE KEY UPDATE
    // 或利用唯一索引
    _, err = tx.Exec(`
        INSERT INTO processed_messages (msg_id, status, create_time)
        VALUES (?, 'processed', NOW())
        ON DUPLICATE KEY UPDATE status='processed'
    `, msg.ID)

    if err != nil {
        tx.Rollback()
        return err // 重复插入会报错，但幂等
    }

    // 业务处理
    err = process(msg)
    if err != nil {
        tx.Rollback()
        return err
    }

    return tx.Commit()
}
```

#### 方案二：Redis 幂等表

```go
func consumeWithRedis(msg *Message) error {
    key := "msg:processed:" + msg.ID

    // SETNX 保证原子性
    exists, err := redis.SetNX(key, "1", 24*time.Hour).Result()
    if err != nil {
        return err
    }
    if !exists {
        return nil // 已处理过
    }

    return process(msg)
}
```

#### 方案三：版本号/状态机

```go
// 订单状态流转：pending → paid → completed
func processOrder(orderID string, msg *Message) error {
    order, err := getOrder(orderID)
    if err != nil {
        return err
    }

    // 状态机判断：只有 pending 才能变成 paid
    if order.Status != "pending" {
        return nil // 幂等丢弃
    }

    return updateOrderStatus(orderID, "paid")
}
```

---

## 消息丢失

### 三个环节

```
生产者 → Broker → 消费者
  ①       ②        ③
```

### ① 生产者丢失

**原因**：发送后未确认、网络抖动

**解决方案**：

```go
// Kafka 异步发送 + 回调确认
producer := &kafka.Producer{
    Acks: kafka.RequireAll,  // 等待所有 ISR 确认
}

producer.Input() <- &kafka.Message{
    Topic: "my-topic",
    Value: "message",
}

producer.Successes() // 阻塞等待确认

// RocketMQ 事务消息
producer.SendMessageInTransaction(msg, func(halfMsg *Message) {
    // 本地事务
    db.Save(data)
}, func(halfMsg *Message) {
    // 回查
})
```

### ② Broker 丢失

**原因**：刷盘策略、ISR 配置

**解决方案**：

```properties
# Kafka 配置
min.insync.replicas=2          # 至少 2 个副本确认
unclean.leader.election.enable=false  # 不选脏 Leader
replication.factor=3           # 3 副本
```

```go
// RocketMQ 同步刷盘
producer.SetAsyncTimeout(10 * time.Second)
```

### ③ 消费者丢失

**原因**：自动提交偏移量后处理失败

**解决方案**：

```go
// 手动提交偏移量，处理完再提交
consumer := kafka.NewConsumer([]string{"localhost:9092"}, "my-group")

for msg := range consumer.Messages() {
    err := process(msg.Value)
    if err != nil {
        // 记录失败，不提交
        logError(err)
        continue
    }
    // 处理成功，手动提交
    consumer.CommitOffsets(msg.Offset)
}
```

### 消息可靠性对比

| 队列 | 可靠性配置 | 性能影响 |
|------|----------|---------|
| Kafka | acks=all, minISR=2 | 高 |
| RocketMQ | 同步刷盘, 事务消息 | 中 |
| RabbitMQ | 镜像队列, publisher confirm | 中 |

---

## 顺序消息

### 生产顺序

```go
// Kafka：按 Partition Key 发送，保证同一 Key 的消息有序
producer.Input() <- &kafka.Message{
    Topic: "order-topic",
    Key:   []byte(orderID), // 同一订单的消息到同一 Partition
    Value: msg,
}
```

### 消费顺序

```go
// 单分区单消费者，或消费者内串行处理
consumer := kafka.NewConsumer(...)
for msg := range consumer.Messages() {
    process(msg) // 串行，不能并行
}
```

---

## 总结

| 问题 | 核心解决方案 |
|------|------------|
| 消息积压 | 扩容消费者、批量消费、监控告警 |
| 重复消费 | 幂等设计（唯一键/Redis/状态机） |
| 消息丢失 | 生产者确认、Broker 多副本、手动提交 |
| 顺序消息 | Partition Key、单消费者串行 |

### 面试话术

**Q：如何保证消息不丢失？**

> 消息可靠性需要从三个环节保证：生产端用同步发送+回调确认，Broker 配置多副本+同步刷盘，消费端手动提交偏移量。这样即使最坏情况（Broker 宕机）也不会丢数据。

**Q：如何处理消息积压？**

> 首先看是消费者慢还是消费者少。如果消费逻辑复杂，可以先扩容消费者并行处理；如果消费本身慢，可以优化消费逻辑或增加批量处理。同时要设置监控告警，提前发现积压趋势。
