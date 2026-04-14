# 秒杀系统设计

> 考察频率：★★★★★  难度：★★★★☆

## 核心问题

| 问题 | 描述 |
|------|------|
| 高并发 | 瞬时流量是平时的 100-1000 倍 |
| 超卖 | 库存有限，请求远超库存 |
| 性能 | 响应延迟直接影响用户体验 |
| 一致性 | 库存、订单、支付的一致性 |

## 系统架构

```
                        ┌─────────────────┐
                        │     CDN          │  静态资源
                        └────────┬────────┘
                                 │
                        ┌────────▼────────┐
                        │   LVS / Nginx   │  负载均衡
                        └────────┬────────┘
                                 │
                 ┌───────────────┼───────────────┐
                 │               │               │
          ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
          │  Web Server │ │  Web Server │ │  Web Server │
          └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
                 │               │               │
          ┌──────▼──────────────▼──────────────▼──────┐
          │              消息队列 (RabbitMQ/Kafka)     │
          │         削峰填谷，异步处理订单              │
          └────────────────────┬─────────────────────┘
                               │
          ┌────────────────────▼─────────────────────┐
          │              库存服务 (Redis)              │
          │         预减库存，Lua 脚本保证原子性        │
          └────────────────────┬─────────────────────┘
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
          ┌──────▼──────┐            ┌───────▼──────┐
          │  订单服务    │            │   支付服务    │
          └──────────────┘            └──────────────┘
```

---

## 核心设计

### 1. 页面静态化

- 秒杀商品页面静态 HTML，提前推到 CDN
- 秒杀按钮倒计时由前端控制
- 避免请求打到后端

### 2. 流量控制

```nginx
# Nginx 限流
limit_req_zone $binary_remote_addr zone=seckill:10m rate=100r/s;

location /seckill {
    limit_req zone=seckill burst=50 nodelay;
    proxy_pass http://backend;
}
```

### 3. 缓存 + 库存扣减

```go
// Redis 预减库存
func preDecrStock(productID string, quantity int64) (bool, error) {
    key := fmt.Sprintf("stock:%s", productID)

    // Lua 脚本保证原子性
    script := `
        local stock = redis.call('GET', KEYS[1])
        if stock and tonumber(stock) >= tonumber(ARGV[1]) then
            redis.call('DECRBY', KEYS[1], ARGV[1])
            return 1
        else
            return 0
        end
    `

    result, err := scripts.Run(rdb, []string{key}, script, quantity).Int64()
    return result == 1, err
}
```

### 4. 消息队列削峰

```go
// 生产者：下单请求
func seckillOrder(userID, productID string) error {
    order := &SeckillOrder{
        UserID:    userID,
        ProductID: productID,
        CreateTime: time.Now(),
    }

    // 发送到队列
    data, _ := json.Marshal(order)
    return mq.Publish("seckill_orders", data)
}

// 消费者：异步处理订单
func consumeOrders() {
    for msg := range mq.Consume("seckill_orders") {
        var order SeckillOrder
        json.Unmarshal(msg.Data, &order)

        // 创建订单
        createOrder(&order)

        // 发送秒杀成功通知
        notifyUser(order.UserID, "秒杀成功")
    }
}
```

---

## 防超卖

### 方案一：Redis 原子扣减

```go
// Lua 脚本：扣减库存并记录订单
local stock = redis.call('GET', KEYS[1])
if tonumber(stock) < tonumber(ARGV[1]) then
    return -1  -- 库存不足
end

redis.call('DECRBY', KEYS[1], ARGV[1])
redis.call('HSET', KEYS[2], ARGV[2], '1')  -- 记录用户已购买
return 1
```

### 方案二：数据库乐观锁

```go
func createOrder(userID, productID string) error {
    // 重试 3 次
    for i := 0; i < 3; i++ {
        // 1. 查询库存
        product, _ := getProduct(productID)
        if product.Stock <= 0 {
            return errors.New("库存不足")
        }

        // 2. 乐观锁更新
        affected, err := db.Exec(`
            UPDATE products
            SET stock = stock - 1, version = version + 1
            WHERE id = ? AND version = ? AND stock > 0
        `, productID, product.Version)

        if err != nil {
            continue
        }

        rows, _ := affected.RowsAffected()
        if rows > 0 {
            // 成功
            return saveOrder(userID, productID)
        }
    }
    return errors.New("下单失败")
}
```

---

## 分布式 session

```go
func login(c *gin.Context) {
    sessionID := uuid.New().String()
    session.Set(sessionID, userID)

    c.SetCookie("session_id", sessionID, 3600, "/", "", false, true)
}

// 后续请求验证
func authMiddleware(c *gin.Context) {
    sessionID, _ := c.Cookie("session_id")
    userID, _ := session.Get(sessionID)
    if userID == nil {
        c.JSON(401, gin.H{"error": "未登录"})
        c.Abort()
        return
    }
    c.Set("userID", userID)
    c.Next()
}
```

---

## 超时处理

```go
// 订单超时未支付，回滚库存
func handleTimeout(orderID string) {
    order := getOrder(orderID)
    if order.Status == "pending" {
        // 库存加回去
        redis.Incr("stock:" + order.ProductID)

        // 订单取消
        updateOrderStatus(orderID, "cancelled")
    }
}

// 启动定时任务，每分钟检查超时订单
func startTimeoutChecker() {
    ticker := time.NewTicker(time.Minute)
    for range ticker.C {
        orders := getPendingOrdersOlderThan(15 * time.Minute)
        for _, order := range orders {
            handleTimeout(order.ID)
        }
    }
}
```

---

## 总结

| 环节 | 方案 |
|------|------|
| 页面静态化 | CDN + 浏览器缓存 |
| 流量控制 | Nginx 限流 + 验证码 |
| 库存扣减 | Redis Lua 原子操作 |
| 订单异步化 | 消息队列削峰 |
| 防超卖 | Redis + 乐观锁 |
| 超时处理 | 定时任务回滚 |

### 关键指标

- **QPS**：预估峰值 × 10 作为容量
- **库存**：一人一件，超卖 = 0
- **响应时间**：< 200ms（P99）

### 面试话术

**Q：如何解决超卖问题？**

> 核心是保证库存扣减的原子性。我用 Redis + Lua 脚本实现，Lua 脚本在 Redis 中原子执行，先检查库存是否足够，再扣减。同时配合数据库乐观锁作为兜底，即使 Redis 出现问题也不会超卖。

**Q：如何扛住瞬时高并发？**

> 三层防护：CDN 扛静态流量，Nginx 限流扛住无效请求，Redis 预减库存 + 消息队列异步处理有效请求。消息队列是削峰的核心，把瞬时流量平滑到后续几秒处理。
