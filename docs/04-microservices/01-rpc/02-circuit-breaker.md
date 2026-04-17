# 熔断与限流

> 考察频率：★★★★★  难度：★★★★☆

## 熔断器（Circuit Breaker）

### 核心思想

```
正常 → 熔断打开 → 半开尝试 → 恢复或继续熔断
```

当服务持续失败/超时，**快速失败**而不是排队等待，避免雪崩。

### 三种状态

| 状态 | 行为 | 触发条件 |
|------|------|---------|
| **Closed** | 正常调用，失败计数 | 失败率超过阈值 |
| **Open** | 直接拒绝，触发熔断 | 失败次数达到阈值 |
| **HalfOpen** | 允许一个请求尝试 | 熔断超时后 |

### Go 实现（手写熔断器）

```go
type State int
const (
    StateClosed   State = iota
    StateOpen
    StateHalfOpen
)

type CircuitBreaker struct {
    name          string
    maxRequests   int32           // HalfOpen 时最大请求数
    failureRatio  float64         // 触发熔断的失败率
    state         State
    failureCount  int32           // 连续失败次数
    successCount  int32           // 连续成功次数
    lastFailure  time.Time       // 上次失败时间
    timeout       time.Duration   // Open 持续时间
    mu            sync.Mutex
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case StateOpen:
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = StateHalfOpen
            cb.successCount = 0
            cb.failureCount = 0
        } else {
            return ErrCircuitOpen
        }
    case StateHalfOpen:
        // 只允许有限请求通过
        if cb.successCount >= cb.maxRequests {
            cb.state = StateClosed
            return nil
        }
    }

    // 执行请求
    err := fn()
    if err != nil {
        cb.onFailure()
        return err
    }
    cb.onSuccess()
    return nil
}

func (cb *CircuitBreaker) onFailure() {
    cb.failureCount++
    cb.lastFailure = time.Now()

    if cb.state == StateHalfOpen {
        cb.state = StateOpen
    } else if cb.failureCount >= int32(cb.maxRequests) {
        cb.state = StateOpen
    }
}

func (cb *CircuitBreaker) onSuccess() {
    cb.successCount++
    if cb.state == StateHalfOpen && cb.successCount >= cb.maxRequests {
        cb.state = StateClosed
        cb.failureCount = 0
        cb.successCount = 0
    }
}
```

### 使用示例

```go
cb := &CircuitBreaker{
    name:         "user-service",
    maxRequests:  3,
    failureRatio: 0.5,
    timeout:      30 * time.Second,
}

for i := 0; i < 10; i++ {
    err := cb.Call(func() error {
        return callService()
    })
    if err != nil {
        log.Printf("请求失败: %v", err)
    }
}
```

### 常见实现

- **Hystrix**（Netflix）：Java，目前维护较少
- **Sentinel**（Alibaba）：Java，功能丰富
- **sony/gobreaker**（Go）：轻量级

---

## 限流算法

### 1. 令牌桶（Token Bucket）

原理：桶内有 token，每次请求消耗一个 token，token 以固定速率补充。

```go
type TokenBucket struct {
    rate       float64       // 每秒补充 token 数
    capacity   int64         // 桶容量
    tokens     float64       // 当前 token 数
    lastUpdate time.Time
    mu         sync.Mutex
}

func NewTokenBucket(rate float64, capacity int64) *TokenBucket {
    return &TokenBucket{
        rate:       rate,
        capacity:   capacity,
        tokens:     float64(capacity),
        lastUpdate: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    return tb.AllowN(1)
}

func (tb *TokenBucket) AllowN(n int64) bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    // 补充 token
    tb.tokens += tb.rate * now.Sub(tb.lastUpdate).Seconds()
    if tb.tokens > float64(tb.capacity) {
        tb.tokens = float64(tb.capacity)
    }
    tb.lastUpdate = now

    if tb.tokens >= float64(n) {
        tb.tokens -= float64(n)
        return true
    }
    return false
}
```

**特点**：
- 允许**突发流量**（桶满时）
- 长时间空闲后仍可一次发出多个请求

### 2. 漏桶（Leaky Bucket）

原理：请求以任意速率进入桶，以固定速率流出。

```go
type LeakyBucket struct {
    capacity int64
    water    int64
    rate     int64         // 每秒漏出数量
    lastTime time.Time
    mu       sync.Mutex
}

func (lb *LeakyBucket) Allow() bool {
    lb.mu.Lock()
    defer lb.mu.Unlock()

    now := time.Now()
    // 漏水
    elapsed := now.Sub(lb.lastTime).Seconds()
    leak := int64(elapsed * float64(lb.rate))
    lb.water -= leak
    if lb.water < 0 {
        lb.water = 0
    }
    lb.lastTime = now

    if lb.water < lb.capacity {
        lb.water++
        return true
    }
    return false
}
```

**特点**：
- 流出速率固定，**平滑流量**
- 不允许突发

### 3. 滑动窗口（Sliding Window）

```go
type SlidingWindow struct {
    windowSize  int64         // 窗口大小（秒）
    maxRequests int64         // 窗口内最大请求数
    requests   []int64       // 请求时间戳
    mu         sync.Mutex
}

func (sw *SlidingWindow) Allow() bool {
    sw.mu.Lock()
    defer sw.mu.Unlock()

    now := time.Now().Unix()

    // 移除窗口外的请求
    cutoff := now - sw.windowSize
    idx := 0
    for i, t := range sw.requests {
        if t > cutoff {
            idx = i
            break
        }
    }
    sw.requests = sw.requests[idx:]

    if int64(len(sw.requests)) < sw.maxRequests {
        sw.requests = append(sw.requests, now)
        return true
    }
    return false
}
```

### 算法对比

| 算法 | 特点 | 适用场景 |
|------|------|---------|
| 令牌桶 | 允许突发，平滑限流 | API 限流、爬虫控制 |
| 漏桶 | 平滑输出，不允许突发 | 流量整形、计费 |
| 滑动窗口 | 精确，内存占用高 | 高精度限流 |
| 固定窗口 | 实现简单，有临界问题 | 简单场景 |

---

## 分布式限流

### Redis 实现令牌桶

```go
func rateLimitWithRedis(ctx context.Context, key string, rate, burst int) (bool, error) {
    now := time.Now().UnixNano()

    // Lua 脚本保证原子性
    script := `
        redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1])
        redis.call('ZCARD', KEYS[1])
        if redis.call('ZCARD', KEYS[1]) < tonumber(ARGV[2]) then
            redis.call('ZADD', KEYS[1], ARGV[3], ARGV[3])
            redis.call('PEXPIRE', KEYS[1], ARGV[4])
            return 1
        else
            return 0
        end
    `

    result, err := scripts.Run(
        rdb, []string{key}, script,
        now-int64(rate)*1e6, // 时间窗口
        burst,                 // 桶容量
        now,                   // 当前时间
        rate*1000,             // 过期时间 ms
    ).Int64()

    return result == 1, err
}
```

### 问：为什么需要 Lua 脚本？

> Redis 操作需要原子性，多个 Redis 命令之间可能被打断，导致竞态条件。Lua 脚本在 Redis 中是原子执行的。

---

## 总结

| 机制 | 目的 | 实现 |
|------|------|------|
| 熔断 | 防止雪崩，快速失败 | 状态机 |
| 令牌桶 | 允许突发，平滑限流 | 令牌补充 |
| 漏桶 | 平滑输出 | 固定流出速率 |
| 滑动窗口 | 精确统计 | 时间窗口 |

### 面试话术

**Q：熔断和限流的区别？**

> 熔断是**被动**的，当服务已经失败/超时才触发，目的是防止雪崩；限流是**主动**的，在服务还没崩溃前就控制流量。两者配合使用，限流在前，熔断在后。

**Q：令牌桶和漏桶的区别？**

> 令牌桶允许一定程度的突发（桶满时可一次取出多个），漏桶输出速率固定不允许突发。令牌桶适合 API 限流（允许合理突发），漏桶适合流量整形（严格平滑）。
