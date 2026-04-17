# 事件驱动架构与 Outbox Pattern

> 考察频率：★★★☆☆  难度：★★★★☆
> 重点：理解事件驱动优势、Outbox Pattern 解决消息可靠性

## 事件驱动架构

### 核心思想

系统间通过**事件（Event）**异步通信，而非同步调用。

```
同步调用：                        事件驱动：
A → B → C（串行等待）           A 发布事件
                               B、C 异步消费
```

### 事件驱动 vs 传统调用

| 维度 | 同步调用 | 事件驱动 |
|------|---------|---------|
| **耦合** | 高（直接依赖） | 低（通过 Broker 解耦） |
| **性能** | 串行，慢 | 并行，快 |
| **可靠性** | 调用失败即失败 | 事件持久化，可重试 |
| **一致性** | 强一致 | 最终一致 |
| **调试** | 简单（同步链路清晰） | 复杂（需链路追踪） |

---

## Go 实现：事件发布与消费

### 简单事件总线

```go
type Event interface {
    EventType() string
    Timestamp() time.Time
}

type EventBus struct {
    handlers map[string][]EventHandler
    mu       sync.RWMutex
}

type EventHandler func(ctx context.Context, event Event) error

func (eb *EventBus) Subscribe(eventType string, handler EventHandler) {
    eb.mu.Lock()
    defer eb.mu.Unlock()
    eb.handlers[eventType] = append(eb.handlers[eventType], handler)
}

func (eb *EventBus) Publish(ctx context.Context, event Event) error {
    eb.mu.RLock()
    handlers := eb.handlers[event.EventType()]
    eb.mu.RUnlock()

    var errs []error
    for _, h := range handlers {
        if err := h(ctx, event); err != nil {
            errs = append(errs, err)
        }
    }
    return errors.Join(errs...)
}
```

### Kafka 事件发布

```go
type KafkaEventPublisher struct {
    producer *kafka.Writer
}

func (p *KafkaEventPublisher) Publish(ctx context.Context, event Event) error {
    data, _ := json.Marshal(event)
    return p.producer.WriteMessages(ctx, kafka.Message{
        Topic: event.EventType(),
        Key:   []byte(event.AggregateID()),
        Value: data,
        Headers: []kafka.Header{
            {Key: "version", Value: []byte("1.0")},
            {Key: "trace_id", Value: []byte(ctx.Value("trace_id").(string))},
        },
    })
}
```

---

## Outbox Pattern（事务消息表）

### 问题：如何保证业务和消息同时成功/失败？

**普通方案的问题：**
```go
// ❌ 先提交业务，再发消息
db.Commit()           // 业务成功
kafka.Send(msg)        // Kafka 失败，消息丢失！
```

**另一个问题：**
```go
// ❌ 先发消息，再提交业务
kafka.Send(msg)        // 消息发出
db.Commit()            // 业务失败，消息已发出，不可撤回
```

### Outbox Pattern 解决方案

**核心思想：** 将消息写入业务表（或专门的 outbox 表），在同一事务中保证原子性。然后由 Relay 进程异步读取 outbox 表发送消息。

```
同一事务：
1. 业务表写入（orders）
2. outbox 表写入（event）
     ↓
事务提交（原子成功/失败）
     ↓
Relay 进程读取 outbox → 发到 Kafka → 删除/标记
```

### Go 实现

```go
type Order struct {
    ID        string
    UserID    string
    Status    string
    Amount    float64
}

// Outbox 表
type Outbox struct {
    ID        int64
    Aggregate string  // "order"
    EventType string  // "order.created"
    Payload   string  // JSON
    CreatedAt time.Time
    Processed bool
}

// 在事务中同时写入业务和 outbox
func CreateOrderTx(db *sql.Tx, order *Order, outboxRepo *OutboxRepo) error {
    // 1. 写入业务表
    _, err := db.Exec(`
        INSERT INTO orders (id, user_id, status, amount) VALUES (?, ?, ?, ?)`,
        order.ID, order.UserID, order.Status, order.Amount)
    if err != nil {
        return err
    }

    // 2. 写入 outbox 表（同一事务）
    event := &OrderCreated{OrderID: order.ID, Amount: order.Amount}
    payload, _ := json.Marshal(event)
    err = outboxRepo.Insert(db, &Outbox{
        Aggregate: "order",
        EventType: "order.created",
        Payload:   string(payload),
    })
    return err
    // 事务提交，业务和 outbox 同时成功/失败
}

// Relay 进程：读取 outbox → 发 Kafka → 标记已处理
func (r *OutboxRelay) Run(ctx context.Context) {
    ticker := time.NewTicker(100 * time.Millisecond)
    for {
        select {
        case <-ticker.C:
            r.processPending(ctx)
        case <-ctx.Done():
            return
        }
    }
}

func (r *OutboxRelay) processPending(ctx context.Context) {
    outboxes := r.repo.GetPending(ctx, limit=100)
    for _, o := range outboxes {
        // 发 Kafka
        err := r.kafka.Send(ctx, o.EventType, o.Payload)
        if err != nil {
            log.Printf("failed to send: %v", err)
            continue
        }
        // 标记已处理
        r.repo.MarkProcessed(ctx, o.ID)
    }
}
```

---

## 事务消息（Kafka Transaction）

### Kafka 事务 API

```go
func CreateOrderWithKafkaTx(order *Order) error {
    producer, _ := NewKafkaTransactionalProducer()
    
    err := producer.Begin()  // 开启事务
    // ...
    producer.Send("orders", order)    // 发到 Kafka（待提交）
    db.Insert(order)                  // 写数据库
    // ...
    return producer.Commit()  // 原子提交：Kafka + DB 同时成功/失败
}
```

### RocketMQ 事务消息

```go
// 1. 发送半消息（Prepared）
_, err = rocketmq.SendHalfMsg(ctx, &Message{
    Topic: "order",
    Body:  orderData,
})
if err != nil {
    return err
}

// 2. 执行本地事务
err = db.Insert(order)

// 3. 提交/回滚
if err == nil {
    rocketMQ.Commit(ctx, msgID)  // 提交
} else {
    rocketMQ.Rollback(ctx, msgID)  // 回滚
}

// 4. MQ 回调检查本地事务状态
type TransactionListener struct{}
func (t *TransactionListener) CheckLocalTransaction(ctx context.Context, msg *MessageExt) LocalTransactionState {
    // 查询本地事务是否成功
    if db.Exists(orderID) {
        return CommitMessageState
    }
    return RollbackMessageState
}
```

---

## Saga + 事件驱动

### 订单处理事件流

```
order.created          → 扣库存 → inventory.reserved
    ↓                        ↓
支付开始 → payment.started   ↓
    ↓                        ↓
支付成功 → payment.completed ↓
    ↓                        ↓
发货开始 → shipping.started
```

```go
// 事件处理器链
type EventProcessor struct {
    orderSvc     *OrderService
    inventorySvc *InventoryService
    paymentSvc   *PaymentService
}

func (ep *EventProcessor) HandleEvent(ctx context.Context, event Event) error {
    switch e := event.(type) {
    case *OrderCreated:
        return ep.inventorySvc.Reserve(ctx, e.OrderID, e.Items)
    case *PaymentCompleted:
        return ep.shippingSvc.Ship(ctx, e.OrderID)
    }
    return nil
}
```

---

## 常见问题

### Q：Outbox 和 RocketMQ 事务消息的区别？

> **Outbox**：自己管理消息表 + Relay 进程，代码可控，适合所有 MQ；**事务消息**：MQ 原生支持，两阶段提交，对 MQ 有依赖（Kafka Transaction 或 RocketMQ）。Outbox 更通用，事务消息性能更好。

### Q：事件驱动架构的缺点？

> 1. **调试困难**：异步链路长，本地调试不方便。2. **一致性**：只能保证最终一致，强一致场景不适用。3. **事件版本管理**：Schema 演进需要考虑兼容性。4. **幂等处理**：消费者必须幂等，防止重复消费。

### Q：如何保证事件顺序消费？

> Kafka Topic + Partition（按 ID 哈希到同一 Partition 保证顺序）；Consumer 端单线程消费；或者使用有序 MQ（如 RocketMQ 顺序消息）。

### Q：Outbox Relay 的可靠性？

> Relay 进程需要 HA（多实例部署，Zookeeper 抢锁）；处理失败时重试；定期补偿未处理的消息；监控 outbox 表堆积量。
