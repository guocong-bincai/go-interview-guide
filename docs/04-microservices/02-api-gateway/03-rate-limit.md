# 网关层限流实现：分布式限流与 Redis + Lua

> 面试频率：★★★★☆  考察角度：令牌桶 vs 滑动窗口 vs 漏桶、Redis Lua 原子操作、Go 实现

---

## 1. 限流算法三剑客

### 1.1 令牌桶（Token Bucket）

**原理**：桶内有固定容量 N，每秒放入 R 个令牌，取出令牌才放行。

```
容量 = 10，令法速 = 5/s
瞬间涌入 10 个请求 → 全部放行（桶内有 10 个令牌）
下一个请求 → 等 1/5 秒后有令牌才放行
```

**优点**：允许一定程度的突发流量。
**适用场景**：API 限流、用户级别限流。

```go
type TokenBucket struct {
    mu       sync.Mutex
    capacity int64      // 桶容量
    rate     float64   // 每秒放入的令牌数
    tokens   float64   // 当前令牌数
    lastTime time.Time // 上次更新时间
}

func NewTokenBucket(capacity int64, rate float64) *TokenBucket {
    return &TokenBucket{
        capacity: capacity,
        rate:     rate,
        tokens:   float64(capacity), // 启动时满桶
        lastTime: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    return tb.AllowN(1)
}

func (tb *TokenBucket) AllowN(n int64) bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(tb.lastTime).Seconds()
    tb.lastTime = now

    // 补充令牌
    tb.tokens += elapsed * tb.rate
    if tb.tokens > float64(tb.capacity) {
        tb.tokens = float64(tb.capacity)
    }

    if tb.tokens >= float64(n) {
        tb.tokens -= float64(n)
        return true
    }
    return false
}
```

### 1.2 漏桶（Leaky Bucket）

**原理**：请求以任意速率进入桶，以固定速率漏出。

```
输入：突发 100 个请求
漏出：每秒处理 10 个
→ 后面 90 个请求在桶内排队，溢出则拒绝
```

**优点**：流量完全平整。
**缺点**：无法处理突发。
**适用场景**：限速出口流量（如 API 调用第三方服务）。

### 1.3 滑动窗口（Sliding Window）

**原理**：将时间窗口切分为多个小段，统计当前时间点前 N 秒的请求数。

```go
type SlidingWindow struct {
    mu       sync.Mutex
    windowSize time.Duration  // 总窗口大小
    bucketSize time.Duration  // 单个桶大小
    buckets    []int64        // 每个桶的计数
    currentIdx int
    lastUpdate time.Time
}

func NewSlidingWindow(windowSize, bucketSize time.Duration, limit int64) *SlidingWindow {
    numBuckets := int(windowSize / bucketSize)
    return &SlidingWindow{
        windowSize: windowSize,
        bucketSize: bucketSize,
        buckets:    make([]int64, numBuckets),
    }
}

func (sw *SlidingWindow) Allow() bool {
    sw.mu.Lock()
    defer sw.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(sw.lastUpdate)
    sw.lastUpdate = now

    // 滑动窗口：清除过期桶
    if elapsed >= sw.bucketSize {
        for i := range sw.buckets {
            sw.buckets[i] = 0
        }
        sw.currentIdx = 0
        sw.buckets[0] = 1
        return true
    }

    // 计算当前窗口内的总请求数
    var total int64
    for _, count := range sw.buckets {
        total += count
    }

    if total >= 100 { // limit
        return false
    }

    sw.buckets[sw.currentIdx]++
    return true
}
```

### 1.4 算法对比

| 算法 | 突发能力 | 平滑度 | 实现难度 | 典型场景 |
|------|---------|--------|---------|----------|
| 令牌桶 | ✅ 允许 | 中 | 低 | API 用户级限流 |
| 漏桶 | ❌ 不允许 | 高 | 中 | 出口限速 |
| 滑动窗口 | ✅ 允许 | 高 | 中 | 精确限流 |

---

## 2. 分布式限流：Redis + Lua

单机限流无法应对多实例部署，需要**分布式限流**。

### 2.1 Redis + Lua 滑动窗口限流

```go
// 滑动窗口限流 Lua 脚本
const slidingWindowLua = `
local key = KEYS[1]          -- 限流 key，如 "rate_limit:user:123"
local window = tonumber(ARGV[1])   -- 窗口大小 ms
local limit = tonumber(ARGV[2])    -- 窗口内最大请求数
local now = tonumber(ARGV[3])      -- 当前时间戳 ms
local expire = tonumber(ARGV[4])   -- key 过期时间

-- 删除窗口外的旧数据
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- 当前窗口请求数
local count = redis.call('ZCARD', key)

if count < limit then
    redis.call('ZADD', key, now, now .. '-' .. math.random())
    redis.call('PEXPIRE', key, expire)
    return 1  -- 允许
else
    return 0  -- 拒绝
end
`

func SlidingWindowLimit(ctx context.Context, rdb *redis.Client, key string, windowMs, limit int64) (bool, error) {
    now := time.Now().UnixMilli()

    result, err := rdb.Eval(ctx, slidingWindowLua,
        []string{key}, windowMs, limit, now, windowMs*2).Int()
    if err != nil {
        return false, err
    }
    return result == 1, nil
}
```

### 2.2 令牌桶分布式实现

```go
const tokenBucketLua = `
local key = KEYS[1]
local capacity = tonumber(ARGV[1])   -- 桶容量
local rate = tonumber(ARGV[2])       -- 每 ms 补充令牌数
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call('HMGET', key, 'tokens', 'lastTime')
local tokens = tonumber(data[1])
local lastTime = tonumber(data[2])

if tokens == nil then
    tokens = capacity
    lastTime = now
end

-- 补充令牌
local elapsed = now - lastTime
tokens = math.min(capacity, tokens + elapsed * rate)
lastTime = now

if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'lastTime', lastTime)
    redis.call('PEXPIRE', key, 60000)
    return 1
else
    return 0
end
`

func TokenBucketLimit(ctx context.Context, rdb *redis.Client, key string, capacity int64, rate float64, requested int64) (bool, error) {
    now := time.Now().UnixMilli()
    result, err := rdb.Eval(ctx, tokenBucketLua,
        []string{key}, capacity, rate, now, requested).Int()
    if err != nil {
        return false, err
    }
    return result == 1, nil
}
```

### 2.3 在 HTTP 中间件中使用

```go
func RateLimitMiddleware(redisAddr string, limit int64, window time.Duration) fiber.Handler {
    rdb := redis.NewClient(&redis.Options{Addr: redisAddr})

    return func(c *fiber.Ctx) error {
        // 用户维度限流
        userID := c.Locals("user_id").(string)
        key := fmt.Sprintf("rate_limit:user:%s", userID)

        allowed, err := SlidingWindowLimit(
            c.Context(),
            rdb,
            key,
            window.Milliseconds(),
            limit,
        )
        if err != nil {
            // Redis 挂了，放行（避免影响可用性）
            return c.Next()
        }

        if !allowed {
            c.Set("X-RateLimit-Limit", fmt.Sprintf("%d", limit))
            c.Set("X-RateLimit-Remaining", "0")
            return c.Status(429).JSON(fiber.Map{
                "code":    429,
                "message": "请求过于频繁，请稍后重试",
            })
        }

        return c.Next()
    }
}
```

---

## 3. 限流维度设计

生产环境限流通常需要多维度组合：

| 维度 | Key 格式 | 说明 |
|------|---------|------|
| 用户 ID | `rate:uid:{user_id}` | 防止单个用户刷接口 |
| IP | `rate:ip:{ip}` | 防止 IP 刷接口 |
| API | `rate:api:{api_path}` | 保护后端服务 |
| 全局 | `rate:global` | 保护整个系统 |
| 服务实例 | `rate:instance:{instance_id}` | 单机维度的兜底 |

**优先级**：用户维度和 IP 维度同时触发更严格的那个。

---

## 4. 限流策略：超出限制怎么办？

### 4.1 友好降级

```go
// 返回 429 之外，可以结合以下策略
type RateLimitResponse struct {
    code    int    // 429
    message string
    retryAfter int64  // 多少 ms 后可以重试
}

// 在 Header 中返回
c.Set("Retry-After", fmt.Sprintf("%d", retryAfter))
c.Set("X-RateLimit-Limit", fmt.Sprintf("%d", limit))
c.Set("X-RateLimit-Remaining", "0")
```

### 4.2 排队机制（适合短时突发）

```go
// 用 Channel 实现简单的请求排队
type QueueLimiter struct {
    queue   chan struct{}
    workers int
}

func NewQueueLimiter(limit int, workers int) *QueueLimiter {
    q := &QueueLimiter{
        queue:   make(chan struct{}, limit),
        workers: workers,
    }
    for i := 0; i < workers; i++ {
        go q.worker()
    }
    return q
}

func (q *QueueLimiter) Allow(ctx context.Context) error {
    select {
    case q.queue <- struct{}{}:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(5 * time.Second):
        return errors.New("timeout waiting for rate limit queue")
    }
}

func (q *QueueLimiter) worker() {
    for range q.queue {
        // 处理请求
        time.Sleep(100 * time.Millisecond)
    }
}
```

---

## 5. 常见问题

**Q：限流 Key 选 Redis 什么数据结构？**
> 滑动窗口用 **ZSET**（sorted set），时间戳做 score，自然过期清理；令牌桶用 **Hash**。

**Q：Redis 限流和本地限流如何选型？**
> 单实例 + 本地限流（单进程多协程），多实例 + Redis 分布式限流。也可以**本地+Redis混合**：本地先拦截大部分请求，Redis 做校准。

**Q：限流后如何不影响可用性？**
> 限流策略要有**分级**：提醒 → 限流 → 拒绝。不要一刀切把服务搞挂。

**Q：如何实现精确的滑动窗口限流？**
> 用 Redis ZSET 存储每次请求的时间戳，用 ZREMRANGEBYSCORE 删除窗口外数据，ZCARD 计数。Lua 脚本保证原子性。
