[🏠 首页](../../../README.md) · [📦 分布式系统](../README.md) · [🔄 分布式事务](../02-transactions/README.md)

---

# 消息最终一致性：本地消息表、事务消息（RocketMQ）、Saga 补偿

## 面试官考察意图

考察候选人对分布式事务解决方案的理解深度。初级能说出"2PC/TCC"，高级要能讲清楚 **本地消息表如何兜住最大努力通知、事务消息如何实现半消息原子性、Saga 模式何时优于 TCC**，并能针对具体业务场景选择合适的方案。

---

## 核心答案（30 秒版）

| 方案 | 原理 | 一致性 | 性能 | 适用场景 |
|------|------|--------|------|----------|
| **本地消息表** | 业务表 + 消息表同一事务，保证消息可查 | 最终一致 | 高 | 普通场景 |
| **事务消息**（RocketMQ）| Broker 半消息 + 回查，保证投递 | 最终一致 | 高 | 高可靠场景 |
| **Saga 补偿** | 正向执行 + 逆向补偿，支持长事务 | 最终一致 | 高 | 长流程、多服务 |
| **TCC** | Try/Confirm/Cancel 人工干预 | 强一致 | 低 | 金融转账 |

**核心理念**：不用 2PC 锁定资源，通过**异步 + 重试 + 幂等** 实现最终一致。

---

## 深度展开

### 1. 本地消息表（Best Effort）

**原理**：把消息写入业务数据库的同一个事务中，通过轮询消息表实现可靠投递。

```
核心流程：
┌─────────────────────────────────────────────────────────────┐
│  事务 T1                                                      │
│  ┌──────────────────┐  ┌──────────────────┐                   │
│  │ 业务表：扣余额    │  │ 消息表：插入消息 │  ← 同一事务      │
│  └──────────────────┘  └──────────────────┘                   │
│                                                             │
│  事务提交 → 消息必定已写入（DB 和 消息表原子）                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  消息投递 worker（轮询消息表）                                │
│  ┌──────────────────────────────────────────────────┐        │
│  │ 1. SELECT * FROM msg WHERE status='pending' LIMIT 10  │   │
│  │ 2. 投递消息到 MQ                                    │        │
│  │ 3. UPDATE msg SET status='success' WHERE id=?         │   │
│  │ 4. 失败 → 重试（status='failed' → 'pending'）     │        │
│  └──────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

**Go 实现**：

```go
// 本地消息表实现
type Order struct {
    ID        string
    UserID    string
    Amount    float64
    Status    string
    CreateAt  time.Time
}

type LocalMessage struct {
    ID          int64
    BusinessID  string    // 关联业务 ID（如 OrderID）
    Topic       string
    Payload     string    // JSON 消息体
    Status      string    // pending | success | failed
    RetryCount  int
    NextRetryAt time.Time
    CreateAt    time.Time
}

// 创建订单 + 消息（同一事务）
func createOrderWithMessage(db *sqlx.DB, msgQ *sqlx.Tx, order *Order, msg *LocalMessage) error {
    tx, err := db.BeginTxx(context.Background(), nil)
    if err != nil {
        return err
    }

    // 1. 插入订单
    _, err = tx.ExecContext(context.Background(),
        `INSERT INTO orders (id, user_id, amount, status) VALUES (?, ?, ?, ?)`,
        order.ID, order.UserID, amount, "pending")
    if err != nil {
        tx.Rollback()
        return err
    }

    // 2. 插入消息（同一事务）
    _, err = tx.ExecContext(context.Background(),
        `INSERT INTO local_messages (business_id, topic, payload, status, next_retry_at)
         VALUES (?, ?, ?, 'pending', NOW())`,
        msg.BusinessID, msg.Topic, msg.Payload)
    if err != nil {
        tx.Rollback()
        return err
    }

    return tx.Commit()
}

// 消息投递 Worker
func startMessageWorker(db *sqlx.DB, producer sarama.SyncProducer) {
    ticker := time.NewTicker(5 * time.Second)
    for range ticker.C {
        msgs, _ := db.QueryxContext(context.Background(), `
            SELECT * FROM local_messages
            WHERE status IN ('pending', 'failed')
              AND next_retry_at <= NOW()
              AND retry_count < 5
            ORDER BY create_at ASC
            LIMIT 10
        `)

        for msgs.Next() {
            var msg LocalMessage
            msgs.StructScan(&msg)

            err := sendMessage(producer, msg.Topic, msg.Payload)
            if err != nil {
                // 投递失败，更新重试次数
                db.ExecContext(context.Background(), `
                    UPDATE local_messages
                    SET status = 'failed',
                        retry_count = retry_count + 1,
                        next_retry_at = NOW() + INTERVAL '5 minutes' * (retry_count + 1)
                    WHERE id = ?
                `, msg.ID)
            } else {
                // 投递成功
                db.ExecContext(context.Background(),
                    `UPDATE local_messages SET status='success' WHERE id=?`, msg.ID)
            }
        }
    }
}
```

**本地消息表的局限性**：

```
局限性：
  1. 消息表和业务表耦合，数据库压力大
  2. 轮询有延迟（最快 5 秒一轮）
  3. 只支持单向流程（无法回滚上游）

适用场景：内部系统、对延迟不敏感的场景
不适用：跨公司、金融交易
```

### 2. 事务消息（RocketMQ / Kafka Transaction）

**RocketMQ 事务消息原理**：

```
两阶段提交 + 回查机制：

阶段1：发送半消息（Half Message）
  Producer 发送消息到 Broker
  Broker 将消息标记为 prepared（对消费者不可见）
  返回 SendResult.SEND_OK

阶段2：执行本地事务
  Producer 执行本地事务（DB 操作）
  根据结果 commit 或 rollback 半消息

阶段3：Broker 回查（如果阶段2超时）
  Broker 向 Producer 查询事务状态
  Producer 查询本地事务状态，返回 COMMITTED / ROLLED_BACK

阶段4：消息对消费者可见
  COMMIT 后，消息对消费者可见
```

```go
// RocketMQ 事务消息实现
func sendOrderWithTransaction(producer *rocketmq.TransactionProducer, order *Order) error {
    msg := &rocketmq.Message{
        Topic: "order-topic",
        Body:  []byte(fmt.Sprintf(`{"order_id":"%s","amount":%f}`, order.ID, order.Amount)),
    }

    // 发送半消息 + 本地事务
    _, err := producer.SendMessageInTransaction(msg,
        func(ctx context.Context, ms *rocketmq.Message) rocketmq.TransactionSendResult {
            // 本地事务：创建订单
            err := createOrder(order)
            if err != nil {
                return rocketmq.TransactionRollback
            }
            return rocketmq.TransactionCommit
        },
        // 回查函数：如果 Broker 长时间未收到 commit/rollback
        func(ctx context.Context, ms *rocketmq.Message) rocketmq.TransactionResolution {
            // 查询本地订单状态
            order, err := getOrder(ms.Keys)
            if err != nil {
                return rocketmq.TransactionRollback
            }
            if order.Status == "completed" {
                return rocketmq.TransactionCommit
            }
            return rocketmq.TransactionRollback
        })
    return err
}
```

**Kafka 事务消息实现**（Kafka 2.3+）：

```go
// Kafka 事务：写 Kafka Topic + 写数据库 原子化
producer := &kafka.ProducerConfig{
    "enable.idempotence": true,
    "transactional.id": "order-service-1",  // 每个服务实例唯一
}

producer.InitTransactions()
producer.BeginTransaction()

// 1. 发送 Kafka 消息
producer.Input() <- &kafka.Message{
    Topic: "order-events",
    Key:   []byte(orderID),
    Value: []byte(orderJSON),
}

// 2. 写数据库
err := db.Exec("INSERT INTO orders ...")
if err != nil {
    producer.AbortTransaction()  // 放弃事务
    return err
}

// 3. 提交事务（Kafka 消息和数据库操作同时提交）
producer.CommitTransaction()

// 注意：如果只有 Kafka 写入（没有 DB），直接用普通事务即可
// Kafka 事务主要用于：Kafka → Kafka（跨分区原子） 或 Kafka → DB
```

### 3. Saga 补偿模式

Saga 把长事务拆成多个**本地事务**，每个本地事务有对应的**补偿操作**。

```
订单创建 Saga 流程：
┌──────────────────────────────────────────────────────────┐
│  Step 1: CreateOrder      （补偿: CancelOrder）           │
│    ↓ 成功                                              ↓ 失败
│  Step 2: ReserveInventory  （补偿: ReleaseInventory）     │
│    ↓ 成功                                              ↓ 失败
│  Step 3: Payment           （补偿: RefundPayment）        │
│    ↓ 成功                                              ↓ 失败
│  Step 4: Shipping          （补偿: CancelShipping）        │
└──────────────────────────────────────────────────────────┘

补偿执行顺序：从失败点向前，依次执行补偿操作
```

**Saga vs TCC 对比**：

| 维度 | Saga | TCC |
|------|------|-----|
| 资源锁定 | 不锁定（各服务正常执行）| Try 阶段锁定资源 |
| 实现复杂度 | 低（只需补偿逻辑）| 高（需三阶段 + 幂等）|
| 性能 | 高 | 低（两阶段提交）|
| 数据一致性 | 最终一致 | 最终一致（可做到强一致）|
| 适用场景 | 长流程、多服务 | 金融转账 |

**Saga 实现：编排模式（Choreography）vs 协调模式（Orchestration）**

```go
// 编排模式：中心化协调者
type OrderSagaOrchestrator struct {
    orderSvc    OrderService
    paymentSvc  PaymentService
    inventorySvc InventoryService
}

func (s *OrderSagaOrchestrator) CreateOrderSaga(ctx context.Context, req *CreateOrderReq) error {
    // 正向执行
    err := s.orderSvc.Create(ctx, req.OrderID)
    if err != nil {
        return err  // 无需补偿，直接失败
    }

    err = s.inventorySvc.Reserve(ctx, req.OrderID, req.Items)
    if err != nil {
        s.orderSvc.Cancel(ctx, req.OrderID)  // 补偿：取消订单
        return err
    }

    err = s.paymentSvc.Charge(ctx, req.OrderID, req.Amount)
    if err != nil {
        s.inventorySvc.Release(ctx, req.OrderID)  // 补偿：释放库存
        s.orderSvc.Cancel(ctx, req.OrderID)        // 补偿：取消订单
        return err
    }

    err = s.shippingSvc.Create(ctx, req.OrderID)
    if err != nil {
        s.paymentSvc.Refund(ctx, req.OrderID)     // 补偿：退款
        s.inventorySvc.Release(ctx, req.OrderID)  // 补偿：释放库存
        s.orderSvc.Cancel(ctx, req.OrderID)        // 补偿：取消订单
        return err
    }

    return nil
}
```

### 4. 三种方案对比与选型

```
选型决策树：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Q1：是否需要强一致性（实时一致）？
  是 → 考虑 2PC / Seata AT 模式
  否 → 继续

Q2：是否锁定资源（对性能要求高）？
  是 → Saga
  否 → 继续

Q3：消息队列是否支持事务消息？
  是（RocketMQ）→ 事务消息（推荐）
  否（RabbitMQ）→ 本地消息表

Q4：是否有多个下游服务需要协调？
  是 → Saga 编排模式
  否 → 本地消息表 / 事务消息（单下游）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**生产经验**：

```go
// 幂等是最终一致性的基础（所有方案通用）
func ensureIdempotent(db *sqlx.DB, msgID string) bool {
    // 使用业务 ID 做幂等，而非消息 ID（消息 ID 可能重复）
    var count int
    err := db.Get(&count, `SELECT COUNT(*) FROM processed_events WHERE event_id=?`, msgID)
    if err != nil {
        return false
    }
    return count > 0
}

// 最大努力通知：失败后重试降级
// 适用于：日志同步、数据归档等非核心场景
func bestEffortNotify(url string, payload []byte) error {
    for i := 0; i < 3; i++ {
        err := httpPost(url, payload)
        if err == nil {
            return nil
        }
        time.Sleep(time.Duration(i+1) * time.Second)  // 指数退避
    }
    // 3 次失败后，记录日志，不阻塞主流程
    logError("Notify failed after 3 retries", url)
    return nil
}
```

---

## 高频追问

**Q：2PC 和 TCC 的区别是什么？**

> 2PC 是数据库层面的协调者，通过 DB 的 XA 协议锁定资源；TCC 是业务层面的三阶段设计，在 Try 阶段锁定资源（预留资源），Confirm 确认使用，Cancel 释放。2PC 锁定时间长且 Coordinator 宕机会导致参与者长时间阻塞；TCC 锁定时间短，但实现复杂且需要各服务实现 Try/Confirm/Cancel。

**Q：如何保证消息不重复消费？**

> 所有消费者都必须实现幂等。常见方式：1）数据库唯一键（订单 ID 作为唯一索引）；2）Redis SETNX（消息 ID 作为 Key）；3）版本号 + 状态机（只有状态机允许的转换才执行）。消息队列本身是 at-least-once 语义，不重复靠消费者幂等。

**Q：RocketMQ 事务消息的回查机制是什么？**

> 如果 Producer 在发送半消息并执行本地事务后，commit/rollback 指令丢失（网络问题），Broker 会在一段时间后主动查询 Producer 的事务状态。Producer 需要实现回查函数，查询本地数据库事务状态并返回 COMMITTED 或 ROLLED_BACK。这个回查是有次数限制的（默认 15 次），超过后消息被丢弃。

**Q：Saga 的补偿失败怎么办？**

> 补偿失败通常是下游服务不可用。解决方案：1）重试机制（指数退避）；2）告警 + 人工干预（对账系统）；3）死信队列（最终无法补偿的消息进入死信队列，定期处理）。Saga 本身不保证一定能补偿成功，最终靠人工兜底和对账系统保证最终一致性。

---

## 延伸阅读

- [Seata 官方文档](https://seata.io/zh-cn/)
- [RocketMQ 事务消息官方文档](https://rocketmq.apache.org/docs/transaction-example/)
- [AWS Saga Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-decentralized-saga/chapter-saga-pattern.html)
