# 限流算法 / 熔断器 / 重试策略

> 考察频率：★★★★☆  优先级：P1
> 关键词：令牌桶、漏桶、滑动窗口计数器、熔断器（Hystrix）、指数退避重试、服务治理、防雪崩

---

## 面试官考察意图

这道题考察候选人对**分布式系统弹性治理能力**的理解。
初级只能说出"用 Redis + Lua 做限流"，高级要能讲清楚**不同限流算法的适用场景、熔断器的状态机设计、指数退避防止雪崩的原因**，并能在高并发场景下给出完整的防护方案。这是生产稳定性的核心技能。

---

## 核心答案（30 秒版）

**限流** = 保护下游不被打垮；**熔断** = 故障时快速失败；**重试** = 处理瞬时错误。三者构成服务治理三道防线。

### 限流算法对比

| 算法 | 核心思想 | 优点 | 缺点 | 适用场景 |
|------|---------|------|------|---------|
| **固定窗口计数器** | 单位时间内累计请求数 | 实现简单 | 临界突发问题 | QPS 波动小的场景 |
| **滑动窗口计数器** | 将固定窗口拆分为小段，加权统计 | 消除临界突发 | 内存占用随精度增加 | 需要平滑统计的场景 |
| **令牌桶** | 以固定速率产生令牌，请求消耗令牌 | 允许一定突发 | 无法精确控制峰值速率 | 保护带宽/吞吐量限定的服务 |
| **漏桶** | 请求进入队列，以固定速率处理 | 强制匀速输出 | 无法利用空闲容量 | 流量整形、削峰填谷 |

### 熔断器状态机

```
┌──────────┐     错误率超过阈值      ┌──────────┐    检测间隔到期    ┌──────────┐
│          │ ──────────────────────▶ │          │ ─────────────────▶ │          │
│   Closed  │ ◀───────────────────── │ Open     │                  │ Half-Open │
│ (正常)    │       成功请求达标      │ (熔断)   │                  │ (试探)    │
│          │                         │          │                  │          │
└──────────┘                         └──────────┘                  └──────────┘
```

- **Closed → Open**：失败率达到阈值（如 50%），立即熔断
- **Open → Half-Open**：冷却时间到期（如 30s），放少量请求探测
- **Half-Open → Closed**：探测请求通过，恢复正常
- **Half-Open → Open**：探测请求失败，再次熔断

### 重试策略

| 策略 | 说明 | 注意事项 |
|------|------|---------|
| **立即重试** | 收到错误后立刻重试 | 仅适用于瞬态故障 |
| **指数退避** | 等待时间 = base × 2^attempt | 防止雪崩 |
| **带随机抖动** | 等待时间 += random(0, cap) | 防止重试风暴 |
| **预算控制** | 总重试耗时不超过原始超时 | 避免无限重试 |

---

## 深度展开

### 一、限流算法详解

#### 1. 固定窗口计数器

```go
// 最简单的限流实现
type CounterLimiter struct {
    limit     int           // 窗口大小
    window    time.Duration // 窗口时长
    count     int           // 当前窗口内计数
    lastReset time.Time     // 上次重置时间
}

func (l *CounterLimiter) Allow() bool {
    now := time.Now()
    if now.Sub(l.lastReset) > l.window {
        l.count = 0
        l.lastReset = now
    }
    if l.count >= l.limit {
        return false
    }
    l.count++
    return true
}
```

**临界问题：** 如果窗口大小为 1 秒、限流 100 QPS，在 1s 末瞬间进来 100 个请求，又在下一秒初瞬间进来 100 个请求，实际上在 1ms 内通过了 200 个请求，限流形同虚设。

#### 2. 滑动窗口计数器

将 1 秒划分为 N 个子窗口（如 10 个 100ms 窗口），每次按权重累加相邻子窗口的计数：

```go
type SlidingWindowLimiter struct {
    windows    [10]int      // 10 个子窗口
    limit      int          // 限流阈值
    windowSize time.Duration // 总窗口时长
}

func (l *SlidingWindowLimiter) Allow() bool {
    l.resetIfNeeded()
    total := 0
    for _, c := range l.windows {
        total += c
    }
    if total >= l.limit {
        return false
    }
    l.windows[currentIndex()]++
    return true
}
```

**优点：** 平滑过渡，不存在临界突变。**代价：** 子窗口越多越精确，但内存占用也越大。

#### 3. 令牌桶算法

核心思路：系统以**固定速率**向桶中放入令牌，请求到达时必须**从桶中取走令牌**才能放行。桶满后丢弃多余令牌。

```go
type TokenBucket struct {
    tokens     float64
    maxTokens  float64
    refillRate float64 // 每秒补充的令牌数
    lastRefill time.Time
}

func (b *TokenBucket) Allow() bool {
    b.refill()
    if b.tokens >= 1 {
        b.tokens--
        return true
    }
    return false
}

func (b *TokenBucket) refill() {
    now := time.Now()
    elapsed := now.Sub(b.lastRefill).Seconds()
    b.tokens = math.Min(b.maxTokens, b.tokens+elapsed*b.refillRate)
    b.lastRefill = now
}
```

**核心特性：**
- 平均速率严格受控，但有**突发能力**（桶中有剩余令牌时可一次性消费多个）
- 比漏桶灵活：漏桶只保证出口速率恒定，不允许任何突发

#### 4. 漏桶算法

请求进入一个固定容量的桶，桶以**固定速率**流出处理请求。桶满时新请求被拒绝。

```
请求到达 → [桶] → 处理(固定速率)
              ↑
            桶满则丢弃
```

**与令牌桶的区别：**
| 特性 | 令牌桶 | 漏桶 |
|------|--------|------|
| 速率控制 | 输入速率可变（有突发） | 输出速率恒定 |
| 突发处理 | ✅ 支持 | ❌ 不支持 |
| 适用场景 | 保护带宽上限 | 流量整形/削峰 |

---

### 二、熔断器模式

熔断器借鉴了电路熔断器的思想：**当检测到持续故障时自动断开电路，防止故障扩散**。

#### 状态机详细设计

```go
const (
    StateClosed    = iota  // 闭合，正常运行
    StateOpen              // 断开，全部拒绝
    StateHalfOpen          // 半开，探测状态
)

type CircuitBreaker struct {
    state         int
    errorCount    int
    successCount  int
    errorThreshold  int     // 触发熔断的错误数阈值
    halfOpenMax   int     // 半开状态最多放行的请求数
    resetTimeout  time.Duration // 熔断持续时间，超时后进入半开
    lastTrippedAt time.Time
    mu            sync.Mutex
}
```

**各状态行为：**

| 状态 | 对请求的处理 | 状态转换条件 |
|------|------------|------------|
| Closed（闭合） | 正常放行 | 连续失败达到阈值 → Open |
| Open（断开） | 直接拒绝（快速失败） | 经过 resetTimeout → Half-Open |
| Half-Open（半开） | 放行少量探测请求 | 探测成功达到阈值 → Closed<br>探测失败一次 → Open |

**为什么需要 Half-Open？**
- 纯 Open → Closed 太激进，故障未恢复就立刻恢复流量，可能再次打垮服务
- Half-Open 用少量请求测试下游是否已恢复，是一种渐进式恢复策略

#### Go 生态实现

推荐直接使用成熟的库，不要自己造轮子：

```go
// 使用 golang.org/x/time/rate 做令牌桶限流
limiter := rate.NewLimiter(rate.Limit(100), 200) // 100r/s，burst=200
if !limiter.Allow() {
    http.Error(w, "rate limited", http.StatusTooManyRequests)
    return
}

// 使用 github.com/sony/gobreaker 做熔断器
var cb gobreaker.Settings
cb.Name = "orderService"
cb.ReadyToTrip = func(counts gobreaker.Counts) bool {
    failRatio := float64(counts.Failures) / float64(counts.Requests)
    return counts.Requests >= 10 && failRatio >= 0.5
}
cb.ResetTimeout = 30 * time.Second  // 熔断持续 30s
cb.HalfOpenStateReqNum = 3           // 半开期放 3 个探测请求

cb.Set("orderservice", cb)
resp, err := cb.Execute(func() (interface{}, error) {
    return callOrderService()
})
```

#### 熔断 vs 限流的区别

| 维度 | 熔断器 | 限流器 |
|------|--------|--------|
| 触发条件 | 依赖方的错误率 | 自身请求量过高 |
| 目的 | 防止故障扩散（级联失效） | 保护自身资源不被耗尽 |
| 效果 | 断开→快速失败→自动恢复 | 排队或直接拒绝 |
| 关注点 | 下游健康度 | 上游压力 |

两者通常**配合使用**：限流保护服务本身不超载，熔断保护不因下游故障而死扛。

---

### 三、重试策略

#### 何时该重试？何时不该？

| 场景 | 建议 | 原因 |
|------|------|------|
| 网络抖动/超时 | ✅ 重试 | 通常是瞬态故障 |
| HTTP 502/503/504 | ✅ 重试 | 对方可能正在恢复 |
| HTTP 5xx | ⚠️ 谨慎重试 | 确认不是服务端 Bug |
| HTTP 4xx（除 429） | ❌ 不重试 | 客户端错误，重试无意义 |
| 死信队列/业务逻辑错误 | ❌ 不重试 | 需要人工介入 |
| 强一致写入 | ❌ 不重试 | 可能导致重复提交（需幂等保障）|

#### 指数退避（Exponential Backoff）

```
attempt 0 → wait 0s   → 立即重试
attempt 1 → wait 1s   → base × 2¹
attempt 2 → wait 2s   → base × 2²
attempt 3 → wait 4s   → base × 2³
attempt 4 → wait 8s   → base × 2⁴
... capped at maxWait (e.g., 60s)
```

**公式：** `waitTime = min(baseDelay × 2^attempt + jitter, maxDelay)`

#### 为什么加随机抖动（Jitter）？

不加抖动时，大量服务同时宕机恢复，所有客户端在同一时刻发起重试，造成**重试风暴（Thundering Herd）**：

```
没有 Jitter 的情况:
T0: Service A ↓ (宕机)
T1: 所有客户端同时重试 T0
T2: Service A 被重试流量压垮，恢复延迟更高
T3: 更多客户端重试... 恶性循环 ✗

有 Jitter 的情况:
T0: Service A ↓ (宕机)
T1: 客户端 1 重试 @ T1.0, 客户端 2 重试 @ T1.3, 客户端 3 重试 @ T1.8...
T2: Service A 平稳恢复 ✓
```

```go
func WithRetry(ctx context.Context, maxRetries int, baseDelay time.Duration, fn func() error) error {
    var err error
    delay := baseDelay
    for i := 0; i <= maxRetries; i++ {
        err = fn()
        if err == nil {
            return nil
        }
        if i == maxRetries {
            break
        }
        // 指数退避 + 随机抖动
        sleepMs := delay.Milliseconds()
        jitter := time.Duration(rand.Int63n(sleepMs))
        select {
        case <-time.After(delay + jitter):
            delay *= 2
            if delay > 30*time.Second {
                delay = 30 * time.Second
            }
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    return fmt.Errorf("retry exhausted after %d attempts: %w", maxRetries, err)
}
```

---

## 生产实战经验

### 经验 1：Go-Zero 内置了全套治理能力

如果你用 Go-Zero 或 Kratos 框架，限流和熔断都是开箱即用的，不必从零实现：

```go
// Go-Zero: 限流 + 熔断一起配置
r := rest.MustNewServer(rest.RestConfig{
    Host:  "0.0.0.0",
    Port:  8888,
    Chain: chain.Chain{
        NewMw(),             // 自定义中间件
    },
}, ...)
r.Use(
    middleware.RateLimitMiddleware(rate.NewLimiter(100, 200)),
    middleware.CircuitBreakerMiddleware(gobreaker.DefaultSettings),
)
defer r.Stop()
fmt.Printf("Server is running at %s\n", r.Address())
```

### 经验 2：全局限流需要分布式协同

单机限流容易遗漏——API 网关有多副本时，需要**全局限流**。常见方案：
- **Redis Lua 脚本**：用 incr + expire 实现共享计数器
- **Sentinel**：阿里云开源的流量防卫兵，支持集群模式
- **网关层统一限流**：Nginx/Envoy 在入口拦截，下游不再重复计算

### 经验 3：熔断器不是银弹

熔断器判断错误率时需要注意采样基数。**错误率 50% 在只有 2 个请求时毫无意义**。正确做法是：
- 设置最小采样数（如 10 个请求以上才判断）
- 结合绝对错误数（如连续 5 个失败也熔断）
- 给不同依赖配不同阈值的熔断器

---

## 面试话术模板

> "我们的防护体系分三层：**网关限流**（全局兜底）→ **服务熔断**（保护下游）→ **重试补偿**（处理瞬态故障）。限流用令牌桶允许适量突发，熔断器用三态机渐进恢复，重试用指数退避+随机抖动防止雪崩。"

---

📌 **扩展阅读：**
- Go-Zero 限流源码解读
- Netflix Hystrix 断路器实现
- Sentinel 流控规则详解
