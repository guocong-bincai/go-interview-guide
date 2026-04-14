# 消息最终一致性：本地消息表与事务消息

> 考察频率：★★★★☆  难度：★★★★☆

## 核心问题

分布式事务的 CAP 困境：强一致（2PC/3PC）性能差、高可用差；TCC/Saga 业务侵入强。**最终一致性**是大多数场景的最优解，通过消息系统异步协调，代价是短暂的不一致窗口。

```
T1: 本地事务 + 消息落库（原子）
T2: 消息投递（可靠）
T3: 消费者处理（幂等）
```

---

## 方案一：本地消息表

### 原理

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  业务表     │     │  消息表      │     │  消息队列   │
│  (本地事务) │ ──▶ │  (同库原子)  │ ──▶ │  (投递)     │
└─────────────┘     └──────────────┘     └──────┬──────┘
                                                 │
                                                 ▼
                                        ┌─────────────┐
                                        │  消费者     │
                                        │  (幂等处理) │
                                        └─────────────┘
```

核心思路：**把消息表的写入和业务表的写入放在同一个本地事务里**，天然保证原子性。

### 建表

```sql
CREATE TABLE local_message (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    msg_id VARCHAR(64) NOT NULL UNIQUE,  -- 消息唯一ID
    topic VARCHAR(128) NOT NULL,          -- 目标 Topic
    body TEXT NOT NULL,                   -- 消息体
    status TINYINT NOT NULL DEFAULT 0,    -- 0=待发送, 1=已发送, 2=发送失败
    retry_count INT DEFAULT 0,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status_time (status, update_time)
);
```

### Go 实现

```go
type LocalMessage struct {
    ID         int64  `json:"id"`
    MsgID      string `json:"msg_id"`
    Topic      string `json:"topic"`
    Body       string `json:"body"`
    Status     int    `json:"status"` // 0=待发送, 1=已发送, 2=失败
    RetryCount int    `json:"retry_count"`
}

const (
    MsgStatusPending   = 0
    MsgStatusSent      = 1
    MsgStatusFailed    = 2
)

// SendTransactional 业务操作和消息落库在同一事务
func SendTransactional(db *sql.DB, kafkaProducer *kafka.Writer, order *Order) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer func() {
        if err != nil {
            tx.Rollback()
        }
    }()

    // 1. 业务操作：创建订单
    _, err = tx.Exec(`
        INSERT INTO orders (order_id, user_id, amount, status)
        VALUES (?, ?, ?, 'pending')
    `, order.OrderID, order.UserID, order.Amount)
    if err != nil {
        return err
    }

    // 2. 消息落库：同一事务
    msgID := fmt.Sprintf("order-created-%s", order.OrderID)
    _, err = tx.Exec(`
        INSERT INTO local_message (msg_id, topic, body, status)
        VALUES (?, 'order-events', ?, 0)
    `, msgID, mustMarshal(order))
    if err != nil {
        return err
    }

    // 提交本地事务（业务 + 消息原子落库）
    if err = tx.Commit(); err != nil {
        return err
    }

    return nil
}

// 定时任务：扫描待发送消息，投递到 MQ
func ScanAndPublish(db *sql.DB, producer *kafka.Writer) {
    rows, err := db.Query(`
        SELECT id, msg_id, topic, body FROM local_message
        WHERE status = 0 AND retry_count < 3
        ORDER BY id LIMIT 100
    `)
    if err != nil {
        return
    }
    defer rows.Close()

    for rows.Next() {
        var m LocalMessage
        rows.Scan(&m.ID, &m.MsgID, &m.Topic, &m.Body)

        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        err := producer.WriteMessages(ctx, kafka.Message{
            Topic: m.Topic,
            Key:   []byte(m.MsgID),
            Value: []byte(m.Body),
        })
        cancel()

        if err != nil {
            db.Exec(`UPDATE local_message SET status=2, retry_count=retry_count+1 WHERE id=?`, m.ID)
        } else {
            db.Exec(`UPDATE local_message SET status=1 WHERE id=?`, m.ID)
        }
    }
}
```

### 优缺点

| 优点 | 缺点 |
|------|------|
| 不依赖外部事务 | 消息表占用数据库存储 |
| 实现简单，与业务解耦 | 需要定时轮询投递，延迟稍高 |
| 强业务侵入 | 事务commit后投递失败有延迟 |

---

## 方案二：RocketMQ 事务消息

RocketMQ 原生支持事务消息，半消息（Prepare）机制，无需定时轮询。

### 原理

```
Producer ──▶ 发送半消息（Prepare）───▶ 执行本地事务
                  │
                  ▼
           Broker 存储半消息（不可投递给消费者）
                  │
                  ▼
Producer ──▶ 提交/回滚事务确认 ──▶ Broker 决定投递或删除
                  │
                  ▼
           投递给消费者
```

### Go 实现

```go
import (
   ocketmq "github.com/apache/rocketmq-client-go/v2"
    producer "github.com/apache/rocketmq-client-go/v2/producer"
)

// TransactionListener 实现事务监听器
type OrderTransactionListener struct {
    db *sql.DB
}

func (t *OrderTransactionListener) ExecuteLocalTransaction(msg *primitive.Message) primitive.LocalTransactionState {
    // 解析消息体
    var order Order
    json.Unmarshal(msg.Body, &order)

    // 执行本地事务
    tx, err := t.db.Begin()
    if err != nil {
        return primitive.RollbackMessageState
    }
    _, err = tx.Exec(`INSERT INTO orders (order_id, user_id, amount) VALUES (?, ?, ?)`,
        order.OrderID, order.UserID, order.Amount)
    if err != nil {
        tx.Rollback()
        return primitive.RollbackMessageState
    }
    tx.Commit()
    return primitive.CommitMessageState
}

func (t *OrderTransactionListener) CheckLocalTransaction(msg *primitive.MessageExt) primitive.LocalTransactionState {
    // Broker 回查：查询本地事务是否成功
    var count int
    t.db.QueryRow(`SELECT COUNT(*) FROM orders WHERE order_id = ?`, msg.Keys).Scan(&count)
    if count > 0 {
        return primitive.CommitMessageState // 订单存在，提交
    }
    return primitive.RollbackMessageState // 订单不存在，回滚
}

// 发送事务消息
func CreateOrderWithTransaction(producer *producer.TransactionProducer, order *Order) error {
    msg := primitive.NewMessage("order-events", mustMarshal(order))
    msg.SetKeys(order.OrderID)

    _, err := producer.SendMessageInTransaction(context.Background(), msg, nil)
    return err
}
```

### RocketMQ vs 本地消息表

| 对比 | 本地消息表 | RocketMQ 事务消息 |
|------|-----------|-----------------|
| 实现复杂度 | 中（自研轮询投递） | 低（原生支持） |
| 依赖 | MySQL | RocketMQ |
| 延迟 | 轮询间隔（秒级） | 即时（回查确认后即投递） |
| 可靠性 | 依赖 DB + MQ 双写 | RocketMQ 半消息保证 |
| 适用场景 | 无特定 MQ 系统 | 已使用 RocketMQ |

---

## 消息消费者：幂等处理

无论哪种方案，消费者都必须**幂等**，因为消息可能重复投递。

### 方案一：唯一键去重

```go
func consumeOrderCreated(msg *kafka.Message) error {
    orderID := extractOrderID(msg.Value)

    // 数据库唯一键幂等
    _, err := db.Exec(`
        INSERT IGNORE INTO order_processed (order_id, status, create_time)
        VALUES (?, 'done', NOW())
    `, orderID)
    if err != nil {
        return nil // Duplicate key，幂等丢弃
    }

    return processOrder(orderID)
}
```

### 方案二：Redis 幂等键

```go
func consumeOrderCreatedRedis(msg *kafka.Message, rdb *redis.Client) error {
    orderID := extractOrderID(msg.Value)
    key := "order:processed:" + orderID

    // SETNX 原子操作
    set, err := rdb.SetNX(context.Background(), key, "1", 24*time.Hour).Result()
    if err != nil {
        return err
    }
    if !set {
        return nil // 已处理过，幂等丢弃
    }

    return processOrder(orderID)
}
```

---

## 三种分布式事务方案对比

| 方案 | 一致性 | 性能 | 业务侵入 | 复杂度 | 适用场景 |
|------|-------|------|---------|--------|---------|
| 2PC/3PC | 强一致 | 差 | 无 | 高 | 数据强一致：金融、库存 |
| TCC | 最终一致 | 好 | 高（改业务逻辑） | 高 | 跨服务资源锁定 |
| Saga | 最终一致 | 好 | 中 | 中 | 长流程、多服务编排 |
| 本地消息表 | 最终一致 | 中 | 低 | 中 | 无特定 MQ |
| RocketMQ 事务 | 最终一致 | 好 | 低 | 低 | 已用 RocketMQ |

---

## 面试话术

**Q：本地消息表和 RocketMQ 事务消息哪个更好？**

> 取决于你的技术栈。如果已经用了 RocketMQ，用它的原生事务消息最简单，不需要额外的消息表和轮询逻辑。本地消息表的优势是不依赖特定 MQ，但需要自研消息投递补偿逻辑，延迟稍高（轮询间隔）。两者本质上都是"事务 + 消息"的原子化，只是存储和投递机制不同。

**Q：如何保证消息一定被消费？**

> 三层保证：1）生产者端：RocketMQ 事务消息的半消息机制，确保本地事务提交后才投递；2）Broker 端：消息持久化 + 消费者 ack 确认才删除；3）消费端：幂等去重 + 失败重试。这样最坏情况（消费者彻底崩溃）可能丢消息，需要结合告警和人工补偿。

**Q：消息延迟送达怎么办？**

> 如果消费者对时间敏感（比如超时取消订单），需要在消费端做"最大处理时间"检查：消息里的创建时间和当前时间差超过阈值就丢弃，而不是处理。另外可以用 RocketMQ 的延迟消息来处理超时逻辑，把整个流程异步化。
