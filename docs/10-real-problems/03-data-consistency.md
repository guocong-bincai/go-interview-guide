# 数据一致性问题与解决方案

> 考察频率：★★★★★
> 面试官考察点：分布式系统下数据一致性是高级工程师必须掌握的核心能力。

---

## 问题导航

| # | 问题 | 核心关键词 |
|---|------|----------|
| 1 | [缓存与数据库不一致](#1-缓存与数据库不一致) | Cache-Aside、延迟双删、监听binlog |
| 2 | [分布式事务失败部分回滚](#2-分布式事务失败部分回滚) | TCC、Saga补偿、最终一致 |
| 3 | [MQ 消息丢失/重复消费](#3-mq-消息丢失重复消费) | ACK机制、幂等消费、死信队列 |
| 4 | [主从延迟导致读到旧数据](#4-主从延迟导致读到旧数据) | 强制读主、等待延迟、semi-sync |
| 5 | [并发写导致数据覆盖](#5-并发写导致数据覆盖) | 乐观锁、CAS、版本号 |

---

## 1. 缓存与数据库不一致

### 面试官考察意图
缓存一致性几乎每次高级岗都会考，要能讲清楚不同策略的适用场景和权衡。

### 为什么会不一致

```
场景：先更新 DB，再更新/删除缓存
  请求A: 更新 DB 成功 → 删除缓存成功
  → 问题：如果两步之间有请求B读到了旧缓存？（短暂不一致，一般可接受）

更严重的场景（并发写+读）：
  时间线：请求A写DB → 请求B读缓存未命中，读DB旧值 → 请求A删缓存 → 请求B写入旧值到缓存
  结果：缓存里是旧数据，永久错误！
```

### 推荐策略：Cache-Aside（旁路缓存）

```go
// 读：先查缓存，未命中查 DB 再写缓存
func getUser(rdb *redis.Client, db *sql.DB, userID int) (*User, error) {
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", userID)

    // 1. 查缓存
    data, err := rdb.Get(ctx, key).Bytes()
    if err == nil {
        var user User
        json.Unmarshal(data, &user)
        return &user, nil
    }

    // 2. 缓存未命中，查 DB
    user, err := queryDB(db, userID)
    if err != nil {
        return nil, err
    }

    // 3. 写入缓存（设 TTL 兜底）
    data, _ = json.Marshal(user)
    rdb.Set(ctx, key, data, 10*time.Minute)
    return user, nil
}

// 写：先更新 DB，再删除缓存（不是更新缓存）
func updateUser(rdb *redis.Client, db *sql.DB, user *User) error {
    // 1. 更新数据库
    if err := updateDB(db, user); err != nil {
        return err
    }

    // 2. 删除缓存（而不是更新，避免并发写时互相覆盖）
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", user.ID)
    rdb.Del(ctx, key)
    return nil
}
```

### 延迟双删（处理并发读写问题）

```go
// 解决并发场景下"写DB→读DB旧值写缓存"的问题
func updateUserSafe(rdb *redis.Client, db *sql.DB, user *User) error {
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", user.ID)

    // 第一次删缓存（防止写DB期间有人读旧缓存再回写）
    rdb.Del(ctx, key)

    // 更新 DB
    if err := updateDB(db, user); err != nil {
        return err
    }

    // 延迟第二次删（等其他请求把旧数据写入缓存后再删）
    // 延迟时间 > 主从同步延迟 + 业务处理时间，通常 500ms~1s
    go func() {
        time.Sleep(500 * time.Millisecond)
        rdb.Del(ctx, key)
    }()

    return nil
}
```

### 最终一致方案：监听 binlog（最可靠）

```
MySQL binlog → Canal/Debezium → Kafka → 缓存更新消费者 → 更新/删 Redis

优点：业务代码无侵入，DB 是唯一数据源，缓存跟着 binlog 同步
缺点：有延迟（毫秒到秒级），需要额外中间件
适合：对一致性要求不是极高（允许短暂不一致）的场景
```

---

## 2. 分布式事务失败部分回滚

### 面试官考察意图
分布式事务是微服务架构必考题，考察候选人能不能根据业务场景选择合适方案。

### 选型决策树

```
你的场景需要强一致吗？
├── 是（金融、资金）→ TCC（预留资源，成功confirm，失败cancel）
└── 否（电商、物流）→ Saga + 最终一致
     ├── 链路短（3步以内）→ 编排型 Saga（直接写补偿代码）
     └── 链路长（5步以上）→ 消息最终一致（本地消息表 + 重试）
```

### TCC 实现要点

```go
// Try 阶段：资源预留（冻结，不真正扣减）
func (s *InventoryService) Try(ctx context.Context, orderID string, qty int) error {
    // 冻结库存（不是真正扣减）
    _, err := s.db.ExecContext(ctx,
        "UPDATE inventory SET frozen=frozen+? WHERE product_id=? AND available>=?",
        qty, s.productID, qty,
    )
    if err != nil {
        return errors.New("库存不足或冻结失败")
    }
    // 记录 TCC 操作日志（用于超时恢复）
    s.logTCCAction(ctx, orderID, "try", qty)
    return nil
}

// Confirm 阶段：真正扣减
func (s *InventoryService) Confirm(ctx context.Context, orderID string) error {
    // 幂等：已经 confirm 过的直接返回
    if s.isConfirmed(ctx, orderID) {
        return nil
    }
    _, err := s.db.ExecContext(ctx,
        "UPDATE inventory SET frozen=frozen-?, real_stock=real_stock-? WHERE order_id=?",
        s.qty, s.qty, orderID,
    )
    return err
}

// Cancel 阶段：释放冻结的资源
func (s *InventoryService) Cancel(ctx context.Context, orderID string) error {
    // 空回滚：Try 阶段没执行，Cancel 直接返回成功
    if !s.isTried(ctx, orderID) {
        return nil
    }
    // 幂等：已经 cancel 过的直接返回
    if s.isCancelled(ctx, orderID) {
        return nil
    }
    _, err := s.db.ExecContext(ctx,
        "UPDATE inventory SET frozen=frozen-? WHERE order_id=?",
        s.qty, orderID,
    )
    return err
}
```

### 消息最终一致（本地消息表）

```go
// 业务操作和写消息在同一个本地事务里，保证原子性
func createOrderWithMessage(db *sql.DB, order Order) error {
    tx, _ := db.Begin()
    defer tx.Rollback()

    // 1. 创建订单
    if _, err := tx.Exec("INSERT INTO orders ...", order); err != nil {
        return err
    }

    // 2. 写本地消息表（和订单在同一事务）
    if _, err := tx.Exec(
        "INSERT INTO outbox_messages (topic, payload, status) VALUES ('order.created', ?, 'PENDING')",
        orderToJSON(order),
    ); err != nil {
        return err
    }

    if err := tx.Commit(); err != nil {
        return err
    }

    return nil
}

// 后台任务：扫描 outbox_messages 发送到 MQ
func outboxWorker(db *sql.DB, producer kafka.Writer) {
    ticker := time.NewTicker(time.Second)
    for range ticker.C {
        rows, _ := db.Query(
            "SELECT id, topic, payload FROM outbox_messages WHERE status='PENDING' LIMIT 100",
        )
        for rows.Next() {
            var id int
            var topic, payload string
            rows.Scan(&id, &topic, &payload)

            // 发送到 Kafka
            if err := producer.WriteMessages(context.Background(),
                kafka.Message{Topic: topic, Value: []byte(payload)},
            ); err == nil {
                db.Exec("UPDATE outbox_messages SET status='SENT' WHERE id=?", id)
            }
        }
    }
}
```

---

## 3. MQ 消息丢失/重复消费

### 面试官考察意图
消息可靠性三大问题（丢失、重复、顺序）是 MQ 必考题。

### 消息丢失的三个环节

```
生产者 → Broker → 消费者
  ↓           ↓          ↓
ACK确认    持久化+多副本  手动ACK
```

```go
// 生产者：确认消息到达 Broker（同步确认）
func sendMessageSafe(producer sarama.SyncProducer, topic string, value []byte) error {
    msg := &sarama.ProducerMessage{
        Topic: topic,
        Value: sarama.ByteEncoder(value),
    }
    // SyncProducer 等待 Broker 确认，如果失败会重试
    _, _, err := producer.SendMessage(msg)
    return err
}

// 消费者：手动 ACK，处理成功后才提交 offset
func consumeMessages(consumer sarama.ConsumerGroup, handler func([]byte) error) {
    for msg := range consumer.Messages() {
        if err := handler(msg.Value); err != nil {
            // 处理失败，不 ACK，让 MQ 重新投递
            log.Errorf("处理失败: %v，等待重试", err)
            continue
        }
        // 处理成功，提交 offset
        consumer.MarkMessage(msg, "")
    }
}
```

### 消费幂等（防重复消费）

```go
// 每条消息用唯一 ID 做幂等键，处理前检查是否已处理
func idempotentConsume(db *sql.DB, rdb *redis.Client, msg Message) error {
    ctx := context.Background()
    idempotentKey := fmt.Sprintf("msg_processed:%s", msg.ID)

    // Redis SETNX：只有第一次能成功
    ok, err := rdb.SetNX(ctx, idempotentKey, 1, 24*time.Hour).Result()
    if err != nil {
        return err
    }
    if !ok {
        // 已经处理过，跳过（幂等）
        log.Infof("消息 %s 已处理，跳过", msg.ID)
        return nil
    }

    // 真正处理
    if err := processMessage(db, msg); err != nil {
        // 处理失败，删除幂等键，允许重试
        rdb.Del(ctx, idempotentKey)
        return err
    }
    return nil
}
```

---

## 4. 主从延迟导致读到旧数据

### 面试官考察意图
主从复制是 MySQL 高可用的基础，考察候选人对读写分离副作用的理解。

### 为什么有延迟

```
主库写入 → binlog → 从库 IO 线程拉取 → relay log → SQL 线程回放
                                                            ↑
                                                    这一步可能有延迟（单线程回放）
```

### 解决方案

```go
// 方案1：写操作后，后续读强制走主库
type DBRouter struct {
    master *sql.DB
    slave  *sql.DB
}

func (r *DBRouter) AfterWrite(ctx context.Context) context.Context {
    // 在 Context 里标记"需要读主库"
    return context.WithValue(ctx, "readMaster", true)
}

func (r *DBRouter) Query(ctx context.Context, query string, args ...interface{}) *sql.Rows {
    if ctx.Value("readMaster") == true {
        rows, _ := r.master.QueryContext(ctx, query, args...)
        return rows
    }
    rows, _ := r.slave.QueryContext(ctx, query, args...)
    return rows
}

// 使用示例
func handleUpdateAndRead(router *DBRouter) {
    ctx := context.Background()

    // 写操作
    router.master.ExecContext(ctx, "UPDATE users SET name=? WHERE id=?", "new_name", 1)

    // 标记：后续需要读主库
    ctx = router.AfterWrite(ctx)

    // 读操作（走主库，避免读到旧数据）
    rows := router.Query(ctx, "SELECT * FROM users WHERE id=?", 1)
    // ...
}

// 方案2：查询时带 min GTID，等从库追上再读（MySQL 5.7+）
// SELECT * FROM users WHERE id=1 WAIT_FOR_EXECUTED_GTID_SET('xxx:1-100', 1);
```

---

## 5. 并发写导致数据覆盖

### 面试官考察意图
并发场景下的数据竞争，考察候选人对乐观锁/CAS的理解与应用。

### 场景复现

```
用户A和用户B同时修改同一条记录
  A读取: score=100
  B读取: score=100
  A写入: score=110（+10）
  B写入: score=95（-5）  ← 覆盖了A的修改！实际应该是 105
```

### 乐观锁（版本号）

```go
// 更新时检查版本号，不匹配则重试
func updateScore(db *sql.DB, userID, delta, expectedVersion int) error {
    result, err := db.Exec(
        "UPDATE users SET score=score+?, version=version+1 WHERE id=? AND version=?",
        delta, userID, expectedVersion,
    )
    if err != nil {
        return err
    }
    rows, _ := result.RowsAffected()
    if rows == 0 {
        return errors.New("数据已被修改，请重试")
    }
    return nil
}

// 带重试的更新
func updateScoreWithRetry(db *sql.DB, userID, delta int) error {
    for i := 0; i < 3; i++ {
        var version int
        db.QueryRow("SELECT version FROM users WHERE id=?", userID).Scan(&version)

        err := updateScore(db, userID, delta, version)
        if err == nil {
            return nil
        }
        if i < 2 {
            time.Sleep(time.Duration(i+1) * 50 * time.Millisecond)
        }
    }
    return errors.New("更新失败，超过最大重试次数")
}
```

### Redis CAS（Compare-And-Swap）

```go
// 用 Lua 脚本实现 Redis 层面的 CAS
func redisCAS(rdb *redis.Client, key string, oldVal, newVal interface{}) (bool, error) {
    script := redis.NewScript(`
        if redis.call('GET', KEYS[1]) == ARGV[1] then
            redis.call('SET', KEYS[1], ARGV[2])
            return 1
        end
        return 0
    `)
    result, err := script.Run(context.Background(), rdb,
        []string{key}, oldVal, newVal,
    ).Int()
    return result == 1, err
}
```

---

## 高频追问汇总

| 问题 | 核心答案 |
|------|---------|
| 先删缓存还是先更新DB？ | 推荐先更新DB再删缓存（Cache-Aside）。先删缓存有窗口期会读到脏数据。 |
| 为什么不直接更新缓存而是删除？ | 并发写时多个线程更新缓存，最终顺序不可控，可能写入旧值；删缓存让下次读时重建，更安全。 |
| TCC 的空回滚和悬挂问题怎么解决？ | 空回滚：Cancel 执行时 Try 未执行，记录 cancel 状态返回成功。悬挂：Try 来的比 Cancel 晚，检查 cancel 记录，直接拒绝 Try。 |
| 消息最终一致有多"最终"？ | 通常秒级到分钟级。可以通过重试策略、告警监控来缩短和发现超期的不一致。 |
| 如何保证消息顺序？ | Kafka 同一分区内有序，同一业务的消息发到同一分区（按业务 ID hash）；消费者单线程消费保证顺序。 |
