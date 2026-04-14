# 限流系统设计

> 考察频率：★★★★☆  难度：★★★☆☆

## 限流维度

| 维度 | 说明 |
|------|------|
| IP | 防止单 IP 刷接口 |
| User | 防止单用户过度使用 |
| 接口 | 不同接口不同限流 |
| 全局 | 系统总 QPS |

---

## 单机限流算法

### 1. 令牌桶

```go
type TokenBucket struct {
    rate     float64 // 每秒补充速率
    capacity int64   // 桶容量
    tokens   float64
    lastTime time.Time
    mu       sync.Mutex
}

func (tb *TokenBucket) Allow() bool {
    return tb.AllowN(1)
}

func (tb *TokenBucket) AllowN(n int64) bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    // 补充 tokens
    tb.tokens += tb.rate * now.Sub(tb.lastTime).Seconds()
    if tb.tokens > float64(tb.capacity) {
        tb.tokens = float64(tb.capacity)
    }
    tb.lastTime = now

    if tb.tokens >= float64(n) {
        tb.tokens -= float64(n)
        return true
    }
    return false
}
```

### 2. 滑动窗口日志

```go
type SlidingWindowLog struct {
    windowSize time.Duration
    timestamps []time.Time
    mu         sync.Mutex
}

func (sw *SlidingWindowLog) Allow() bool {
    sw.mu.Lock()
    defer sw.mu.Unlock()

    now := time.Now()
    cutoff := now.Add(-sw.windowSize)

    // 移除过期的
    idx := 0
    for i, t := range sw.timestamps {
        if t.After(cutoff) {
            idx = i
            break
        }
    }
    sw.timestamps = sw.timestamps[idx:]

    // 检查是否超限
    if len(sw.timestamps) < 100 { // 100 req / windowSize
        sw.timestamps = append(sw.timestamps, now)
        return true
    }
    return false
}
```

### 3. 滑动窗口计数

```go
type SlidingWindowCount struct {
    windowSize int64   // 窗口大小（秒）
    maxCount   int64   // 窗口内最大请求数
    counts     []int64 // 每秒请求数
    mu         sync.Mutex
}

func (sw *SlidingWindowCount) Allow() bool {
    sw.mu.Lock()
    defer sw.mu.Unlock()

    now := time.Now().Unix()

    // 移除过期窗口
    sw.counts = sw.counts[len(sw.counts):]

    // 计算当前窗口内的请求数
    var total int64
    for _, c := range sw.counts {
        total += c
    }

    if total < sw.maxCount {
        sw.counts[len(sw.counts)-1]++
        return true
    }
    return false
}
```

---

## 分布式限流

### Redis + Lua 令牌桶

```go
func rateLimit(ctx context.Context, key string, rate int, burst int) (bool, error) {
    now := time.Now().Unix()
    script := `
        local key_meta = KEYS[1] .. ':meta'
        local key_tokens = KEYS[1] .. ':tokens'

        -- 获取元数据
        local meta = redis.call('HMGET', key_meta, 'last_time', 'tokens')
        local last_time = tonumber(meta[1]) or ARGV[1]
        local tokens = tonumber(meta[2]) or tonumber(ARGV[2])

        local now = tonumber(ARGV[1])
        local rate = tonumber(ARGV[3])
        local requested = tonumber(ARGV[4])

        -- 补充 tokens
        local elapsed = now - last_time
        tokens = math.min(tonumber(ARGV[2]), tokens + elapsed * rate)

        if tokens >= requested then
            tokens = tokens - requested
            redis.call('HMSET', key_meta, 'last_time', now, 'tokens', tokens)
            redis.call('EXPIRE', key_meta, 60)
            return 1
        else
            return 0
        end
    `

    result, err := scripts.Run(rdb,
        []string{key}, script,
        now, burst, rate, 1,
    ).Int64()

    return result == 1, err
}
```

### 固定窗口 + Redis INCR

```go
func fixedWindowRateLimit(key string, limit int, windowSec int) (bool, error) {
    now := time.Now().Unix()
    windowKey := fmt.Sprintf("%s:%d", key, now/int64(windowSec))

    count, err := rdb.Incr(windowKey).Result()
    if err != nil {
        return false, err
    }

    if count == 1 {
        rdb.Expire(windowKey, time.Duration(windowSec)*time.Second)
    }

    return count <= int64(limit), nil
}
```

---

## HTTP 限流中间件

```go
func RateLimitMiddleware(rate int, burst int) gin.HandlerFunc {
    bucket := NewTokenBucket(float64(rate), int64(burst))

    return func(c *gin.Context) {
        if !bucket.Allow() {
            c.JSON(429, gin.H{
                "error": "too many requests",
            })
            c.Abort()
            return
        }
        c.Next()
    }
}

// 使用
router.GET("/api", RateLimitMiddleware(100, 200), handler)
```

---

## 限流策略

### 降级方案

| 限流触发 | 降级策略 |
|---------|---------|
| 10% 超限 | 延迟响应 |
| 30% 超限 | 返回友好错误 |
| 50% 超限 | 跳转到静态页面 |
| 100% 超限 | 直接拒绝 |

### 用户提示

```json
{
    "error": "rate_limit_exceeded",
    "message": "请求过于频繁，请稍后再试",
    "retry_after": 60
}
```

---

## 总结：限流选型

| 场景 | 推荐算法 |
|------|---------|
| API 限流 | 令牌桶（允许突发） |
| 登录限流 | 固定窗口 + 滑动窗口 |
| 支付限流 | 滑动窗口（精确） |
| 爬虫控制 | 用户维度令牌桶 |

### 关键点
1. **单机限流**：token bucket / sliding window
2. **分布式限流**：Redis + Lua 保证原子性
3. **限流维度**：IP / User / 接口 / 全局
4. **降级策略**：分层降级，用户体验优先
