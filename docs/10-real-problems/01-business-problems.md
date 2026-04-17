# 业务层高频问题与解决方案

> 考察频率：★★★★★
> 面试官考察点：你有没有真正在生产上遇到过问题、解决过问题，答案要有细节、有数据、有反思。

---

## 问题导航

| # | 问题 | 核心关键词 |
|---|------|----------|
| 1 | [超卖问题](#1-超卖问题) | 库存并发、Redis原子、数据库行锁 |
| 2 | [重复下单/重复支付](#2-重复下单重复支付) | 幂等、唯一键、token机制 |
| 3 | [订单状态不一致](#3-订单状态不一致) | 分布式事务、MQ补偿、对账 |
| 4 | [大促活动流量突增](#4-大促活动流量突增) | 限流、降级、预热、异步化 |
| 5 | [数据库慢查询拖垮服务](#5-数据库慢查询拖垮服务) | 慢查询、索引、连接池、读写分离 |
| 6 | [第三方接口超时](#6-第三方接口超时) | 超时熔断、降级、异步回调 |

---

## 1. 超卖问题

### 面试官考察意图
考察候选人对并发场景下数据一致性的理解，以及有没有在生产上处理过高并发抢购/秒杀场景。

### 什么是超卖

```
库存只有 1 件，10 个请求同时读到库存=1，全部判断"有货"，结果扣减了 10 次。
```

### 解决方案全景

```
方案一：数据库行锁（悲观锁）       ← 低并发、强一致场景
方案二：数据库乐观锁（版本号）      ← 中等并发、读多写少
方案三：Redis 原子操作             ← 高并发秒杀推荐
方案四：Redis + 异步队列           ← 超高并发（万级QPS）
```

### 方案一：数据库行锁

```go
// SELECT ... FOR UPDATE 锁住这一行，同一时间只有一个事务能读+写
func deductStock(db *sql.DB, productID int, quantity int) error {
    tx, _ := db.Begin()
    defer tx.Rollback()

    var stock int
    // FOR UPDATE：加行锁，其他事务阻塞等待
    err := tx.QueryRow(
        "SELECT stock FROM products WHERE id=? FOR UPDATE", productID,
    ).Scan(&stock)
    if err != nil {
        return err
    }

    if stock < quantity {
        return errors.New("库存不足")
    }

    _, err = tx.Exec(
        "UPDATE products SET stock=stock-? WHERE id=?", quantity, productID,
    )
    if err != nil {
        return err
    }
    return tx.Commit()
}
```

**缺点：** 行锁会阻塞其他事务，并发高时大量请求排队，吞吐量低。

### 方案二：数据库乐观锁

```go
// 更新时带上 version，如果 version 不匹配说明被别人改过，重试
func deductStockOptimistic(db *sql.DB, productID, quantity int) error {
    for retry := 0; retry < 3; retry++ {
        var stock, version int
        err := db.QueryRow(
            "SELECT stock, version FROM products WHERE id=?", productID,
        ).Scan(&stock, &version)
        if err != nil {
            return err
        }
        if stock < quantity {
            return errors.New("库存不足")
        }

        result, err := db.Exec(
            // WHERE 带上 version，并发更新时只有一个能成功
            "UPDATE products SET stock=stock-?, version=version+1 WHERE id=? AND version=?",
            quantity, productID, version,
        )
        if err != nil {
            return err
        }
        rows, _ := result.RowsAffected()
        if rows == 1 {
            return nil // 成功
        }
        // rows == 0 说明被其他请求抢先更新，重试
        time.Sleep(time.Duration(retry+1) * 10 * time.Millisecond)
    }
    return errors.New("更新失败，请重试")
}
```

**缺点：** 高并发时大量请求重试，ABA 问题需要用版本号而不是比较值。

### 方案三：Redis DECRBY 原子操作（推荐）

```go
// Redis DECRBY 是原子操作，天然防并发
func deductStockRedis(rdb *redis.Client, productID, quantity int) error {
    ctx := context.Background()
    key := fmt.Sprintf("stock:%d", productID)

    // Lua 脚本保证"查库存 + 扣减"原子性
    script := redis.NewScript(`
        local stock = tonumber(redis.call('GET', KEYS[1]))
        if stock == nil or stock < tonumber(ARGV[1]) then
            return -1  -- 库存不足
        end
        return redis.call('DECRBY', KEYS[1], ARGV[1])
    `)

    result, err := script.Run(ctx, rdb, []string{key}, quantity).Int()
    if err != nil {
        return err
    }
    if result == -1 {
        return errors.New("库存不足")
    }
    return nil
}
```

**关键点：**
- 用 Lua 脚本保证"读 + 判断 + 写"的原子性，不能拆成两次命令
- Redis 库存是预热进去的，数据库是最终落库

### 方案四：Redis + 异步队列（超高并发）

```
请求进来 → Redis Lua 扣减库存（纯内存，极快）
                    ↓
              写入 MQ 队列（Kafka/RocketMQ）
                    ↓
          消费者异步处理：落库、创建订单、通知支付
```

```go
func handleSeckill(rdb *redis.Client, mq *kafka.Writer, userID, productID int) error {
    // 1. Redis 原子扣减
    if err := deductStockRedis(rdb, productID, 1); err != nil {
        return err // 库存不足直接返回
    }

    // 2. 写入队列，异步创建订单
    msg := OrderMessage{UserID: userID, ProductID: productID, Timestamp: time.Now()}
    body, _ := json.Marshal(msg)
    return mq.WriteMessages(context.Background(), kafka.Message{Value: body})
}
```

### 面试话术

> "我们做活动秒杀时遇到过超卖。最初用的数据库行锁，但秒杀一开始 DB 连接池就打满了，P99 直接飙到 3 秒。后来改成 Redis Lua 脚本扣减 + Kafka 异步落库，库存扣减走纯内存，吞吐量从 800 QPS 提升到 12000 QPS，超卖问题彻底解决。"

---

## 2. 重复下单/重复支付

### 面试官考察意图
幂等性设计是后端必考题，考察候选人能不能从前端到后端全链路设计幂等。

### 为什么会发生

```
1. 网络超时：请求到达服务端处理成功，但响应丢失，客户端重试
2. 用户操作：用户快速点击多次"提交"按钮
3. MQ 重投：消费者处理失败，MQ 重新投递
4. 定时任务：重跑任务重复执行
```

### 全链路幂等设计

```
前端：提交前禁用按钮，生成唯一 idempotency_key（UUID）
  ↓
API 网关：记录 idempotency_key，相同 key 返回缓存结果
  ↓
业务服务：唯一索引 / 先查后插 / Token 机制
  ↓
数据库：唯一约束兜底
```

### 方案一：唯一索引（最简单最可靠）

```sql
-- 订单表加唯一索引
ALTER TABLE orders ADD UNIQUE KEY uk_user_order (user_id, idempotency_key);
```

```go
func createOrder(db *sql.DB, userID int, key string, order Order) error {
    _, err := db.Exec(
        "INSERT INTO orders (user_id, idempotency_key, ...) VALUES (?, ?, ...)",
        userID, key, ...
    )
    if isDuplicateKeyError(err) {
        // 重复请求，直接返回已有订单（不是错误）
        return nil
    }
    return err
}

func isDuplicateKeyError(err error) bool {
    if err == nil {
        return false
    }
    var mysqlErr *mysql.MySQLError
    return errors.As(err, &mysqlErr) && mysqlErr.Number == 1062
}
```

### 方案二：Token 机制（防重复提交）

```go
// 步骤1：进入下单页面时，生成 token 存入 Redis
func getOrderToken(rdb *redis.Client, userID int) (string, error) {
    token := uuid.New().String()
    key := fmt.Sprintf("order_token:%d", userID)
    // 设置 5 分钟过期
    err := rdb.Set(context.Background(), key, token, 5*time.Minute).Err()
    return token, err
}

// 步骤2：提交时校验并删除 token（删除成功才能继续）
func validateAndConsumeToken(rdb *redis.Client, userID int, token string) (bool, error) {
    key := fmt.Sprintf("order_token:%d", userID)
    // Lua 脚本：校验 + 删除 原子操作
    script := redis.NewScript(`
        if redis.call('GET', KEYS[1]) == ARGV[1] then
            redis.call('DEL', KEYS[1])
            return 1
        end
        return 0
    `)
    result, err := script.Run(context.Background(), rdb, []string{key}, token).Int()
    return result == 1, err
}
```

### 方案三：MQ 消费幂等

```go
// 消费者处理消息前先检查是否已处理过
func consumeOrder(db *sql.DB, msg OrderMessage) error {
    // 用消息的 offset 或业务 ID 做幂等键
    var exists int
    db.QueryRow(
        "SELECT COUNT(1) FROM processed_messages WHERE message_id=?", msg.ID,
    ).Scan(&exists)

    if exists > 0 {
        return nil // 已处理，跳过
    }

    // 业务处理 + 记录已处理（同一个事务）
    tx, _ := db.Begin()
    // ... 业务逻辑 ...
    tx.Exec("INSERT INTO processed_messages (message_id) VALUES (?)", msg.ID)
    return tx.Commit()
}
```

---

## 3. 订单状态不一致

### 面试官考察意图
分布式系统下数据一致性是高频考点，考察候选人对"最终一致性"方案的掌握程度。

### 常见场景

```
场景1：扣库存成功 → 创建订单失败 → 库存不回滚
场景2：支付成功 → MQ 消息丢失 → 订单状态未更新
场景3：退款成功 → 库存未回补
```

### 解决方案：事务消息 + 补偿对账

```go
// 使用 RocketMQ 事务消息（保证业务操作和消息发送的原子性）
func createOrderWithTxMsg(producer rocketmq.TransactionProducer, order Order) error {
    msg := &primitive.Message{
        Topic: "order-created",
        Body:  orderToBytes(order),
    }

    // 发送半消息（此时消费者看不到）
    _, err := producer.SendMessageInTransaction(
        context.Background(), msg,
        func(ctx context.Context, msg *primitive.Message, state primitive.LocalTransactionState) primitive.LocalTransactionState {
            // 执行本地事务：创建订单
            if err := doCreateOrder(order); err != nil {
                return primitive.RollbackMessageState // 回滚半消息
            }
            return primitive.CommitMessageState // 提交，消费者可见
        },
        nil,
    )
    return err
}

// 对账任务：定时扫描状态异常的订单
func reconcileOrders(db *sql.DB) {
    // 找出超过 10 分钟还在"待支付"状态的订单
    rows, _ := db.Query(`
        SELECT id, status, created_at
        FROM orders
        WHERE status='PENDING' AND created_at < NOW() - INTERVAL 10 MINUTE
    `)
    for rows.Next() {
        var order Order
        rows.Scan(&order.ID, &order.Status, &order.CreatedAt)
        // 调用支付系统查询实际支付状态，做补偿
        checkAndFixOrder(order)
    }
}
```

---

## 4. 大促活动流量突增

### 面试官考察意图
高并发保障方案，考察候选人有没有参与过大流量活动的保障工作。

### 流量突增的应对层次

```
第一层：接入层限流（Nginx/网关）
  → 超出 QPS 阈值直接返回 429

第二层：服务层限流（令牌桶/信号量）
  → 每个接口独立限流，保护下游

第三层：降级（返回缓存/静态页/兜底数据）
  → 非核心功能降级，保核心链路

第四层：异步化（写请求异步，快速返回）
  → 下单写队列，异步处理

第五层：资源预热（提前把热数据加载到缓存）
  → 活动开始前 10 分钟预热
```

```go
// 令牌桶限流（golang.org/x/time/rate）
import "golang.org/x/time/rate"

var limiter = rate.NewLimiter(rate.Limit(10000), 20000) // 每秒1万，桶容量2万

func handler(w http.ResponseWriter, r *http.Request) {
    if !limiter.Allow() {
        http.Error(w, "服务繁忙，请稍后重试", http.StatusTooManyRequests)
        return
    }
    // 正常处理...
}

// 活动预热：提前把商品信息、库存写入 Redis
func warmUp(rdb *redis.Client, db *sql.DB, activityID int) error {
    products, _ := fetchActivityProducts(db, activityID)
    pipe := rdb.Pipeline()
    for _, p := range products {
        pipe.Set(context.Background(), fmt.Sprintf("stock:%d", p.ID), p.Stock, time.Hour)
        pipe.Set(context.Background(), fmt.Sprintf("product:%d", p.ID), productToJSON(p), time.Hour)
    }
    _, err := pipe.Exec(context.Background())
    return err
}
```

### 面试话术

> "我们双十一前做了几件事：一是提前 30 分钟把活动商品库存预热到 Redis；二是在网关层加了令牌桶限流，把单接口 QPS 压在 8000 以内；三是非核心功能（评价、推荐）设了降级开关，流量大时返回空数据；四是下单改成异步，先写 Kafka，消费者异步落库，前端轮询订单状态。活动期间峰值 1.2 万 QPS，服务稳定，没有出现雪崩。"

---

## 5. 数据库慢查询拖垮服务

### 面试官考察意图
数据库优化是必考项，要求能讲清楚发现 → 分析 → 解决的完整流程。

### 排查流程

```
1. 发现：接口 P99 突然升高，告警触发
2. 定位：pprof 显示大量时间在 db.QueryRow，链路追踪看 DB span 耗时
3. 分析：慢查询日志找到具体 SQL，EXPLAIN 分析执行计划
4. 解决：加索引/改 SQL/读写分离/缓存
```

```go
// 开启慢查询日志（MySQL配置）
// slow_query_log = ON
// long_query_time = 1  (超过1秒记录)
// log_queries_not_using_indexes = ON

// Go 代码层面：打印慢 SQL
type slowQueryLogger struct {
    db        *sql.DB
    threshold time.Duration
}

func (l *slowQueryLogger) Query(query string, args ...interface{}) (*sql.Rows, error) {
    start := time.Now()
    rows, err := l.db.Query(query, args...)
    elapsed := time.Since(start)
    if elapsed > l.threshold {
        log.Warnf("慢查询 [%.2fms]: %s, args: %v", float64(elapsed.Milliseconds()), query, args)
    }
    return rows, err
}
```

### 典型慢查询原因与修复

```sql
-- 问题1：深分页（LIMIT 100000, 10 会扫描10万行）
-- 慢：
SELECT * FROM orders ORDER BY id LIMIT 100000, 10;

-- 快：游标分页（记录上次最大ID）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 10;

-- 问题2：函数导致索引失效
-- 慢：对索引列用了函数
SELECT * FROM users WHERE DATE(created_at) = '2024-01-01';

-- 快：改成范围查询
SELECT * FROM users WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02';

-- 问题3：隐式类型转换
-- user_id 是 VARCHAR，传入 int 导致全表扫描
SELECT * FROM users WHERE user_id = 12345;   -- 慢
SELECT * FROM users WHERE user_id = '12345'; -- 快
```

---

## 6. 第三方接口超时

### 面试官考察意图
考察候选人对外部依赖的治理能力，超时、熔断、降级是标准答案。

### 问题场景

```
调用短信服务、支付宝/微信支付、物流查询等第三方 API 时：
- 对方接口偶发超时（网络抖动）
- 对方接口持续故障（服务宕机）
```

### 解决方案

```go
// 1. 设置合理超时（必须有！）
client := &http.Client{
    Timeout: 3 * time.Second, // 绝对不能用默认无限超时
    Transport: &http.Transport{
        DialContext: (&net.Dialer{
            Timeout: 1 * time.Second, // 连接超时
        }).DialContext,
        ResponseHeaderTimeout: 2 * time.Second,
    },
}

// 2. 熔断器（使用 gobreaker）
import "github.com/sony/gobreaker"

var cb = gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "sms-service",
    MaxRequests: 5,                // 半开状态最多允许5个请求
    Interval:    60 * time.Second, // 统计窗口
    Timeout:     30 * time.Second, // 熔断后等待时间
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        // 失败率超过 60% 触发熔断
        return counts.ConsecutiveFailures > 5 ||
               float64(counts.TotalFailures)/float64(counts.Requests) > 0.6
    },
})

func sendSMS(phone, content string) error {
    _, err := cb.Execute(func() (interface{}, error) {
        return nil, callSMSAPI(phone, content)
    })
    if err == gobreaker.ErrOpenState {
        // 熔断中，降级处理（比如存库稍后重试）
        return saveSMSForRetry(phone, content)
    }
    return err
}

// 3. 重试（指数退避，只对幂等操作重试）
func callWithRetry(fn func() error, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        if !isRetryable(err) { // 4xx 错误不重试
            return err
        }
        // 指数退避：100ms, 200ms, 400ms...
        time.Sleep(time.Duration(100*(1<<i)) * time.Millisecond)
    }
    return errors.New("超过最大重试次数")
}
```

### 面试话术

> "我们对接微信支付时，遇到过对方接口偶发 5 秒超时，导致我们的下单接口也跟着超时，连接池被占满，最后整个服务被拖垮。后来做了三件事：一是把 HTTP 客户端超时从默认无限改成 3 秒；二是加了熔断器，失败率超 60% 时熔断 30 秒，熔断期间走降级（存库后定时重试）；三是支付结果改成异步回调而不是同步等待。改完之后，即使微信偶发超时，我们服务完全不受影响。"

---

## 高频追问汇总

| 问题 | 核心答案 |
|------|---------|
| 幂等和事务的区别？ | 事务保证操作的原子性；幂等保证同一操作多次执行结果相同。两者解决不同问题，通常配合使用。 |
| 乐观锁和悲观锁如何选择？ | 读多写少、冲突概率低 → 乐观锁；写多、冲突概率高 → 悲观锁；高并发秒杀 → Redis 原子操作。 |
| 如何防止缓存和数据库不一致？ | 先更新数据库再删缓存（Cache-Aside）；引入延迟双删；用消息队列异步更新。 |
| 熔断和限流的区别？ | 限流是主动保护（控制流量上限）；熔断是被动保护（发现下游故障后停止请求）。 |
| 超卖了怎么办（事后补救）？ | 建立对账任务，定时校验 Redis 库存和 DB 库存；超卖的订单联系用户补货或退款；复盘代码缺陷。 |
