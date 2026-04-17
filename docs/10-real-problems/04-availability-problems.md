# 可用性与稳定性问题

> 考察频率：★★★★★
> 面试官考察点：线上故障处理、高可用架构设计，区分 P5 和高级工程师的关键模块。

---

## 问题导航

| # | 问题 | 核心关键词 |
|---|------|----------|
| 1 | [缓存雪崩](#1-缓存雪崩) | 大面积失效、随机TTL、多级缓存 |
| 2 | [缓存击穿](#2-缓存击穿) | 热key过期、互斥锁、逻辑过期 |
| 3 | [依赖服务宕机](#3-依赖服务宕机) | 熔断降级、服务隔离、快速失败 |
| 4 | [服务雪崩](#4-服务雪崩) | 级联故障、超时、线程池隔离 |
| 5 | [流量不均衡/热点问题](#5-流量不均衡热点问题) | 负载均衡策略、一致性哈希、热点路由 |

---

## 1. 缓存雪崩

### 面试官考察意图
缓存三大问题（雪崩/击穿/穿透）必考，要能讲清楚每个的成因和对应方案。

### 什么是缓存雪崩

```
大量缓存在同一时间过期（比如业务初始化时批量写入，TTL 相同）
→ 瞬间大量请求穿透缓存打到数据库
→ 数据库瞬间过载，接口超时，服务崩溃
```

### 解决方案

```go
// 方案1：TTL 加随机抖动，避免同时过期
func setCacheWithJitter(rdb *redis.Client, key string, value interface{}, baseTTL time.Duration) error {
    // 在基础 TTL 上加 ±20% 的随机抖动
    jitter := time.Duration(rand.Int63n(int64(baseTTL) / 5))
    if rand.Intn(2) == 0 {
        jitter = -jitter
    }
    ttl := baseTTL + jitter

    data, _ := json.Marshal(value)
    return rdb.Set(context.Background(), key, data, ttl).Err()
}

// 方案2：多级缓存（本地缓存 + Redis + DB）
// 即使 Redis 全部失效，本地缓存还能扛一部分流量
var localCache = &sync.Map{}

func getWithMultiLevel(rdb *redis.Client, db *sql.DB, key string) (string, error) {
    // L1：本地缓存（最快，无网络）
    if v, ok := localCache.Load(key); ok {
        return v.(string), nil
    }

    // L2：Redis
    val, err := rdb.Get(context.Background(), key).Result()
    if err == nil {
        localCache.Store(key, val) // 回填本地缓存
        return val, nil
    }

    // L3：数据库（兜底）
    val, err = queryFromDB(db, key)
    if err != nil {
        return "", err
    }
    // 写入两层缓存
    rdb.Set(context.Background(), key, val, 10*time.Minute)
    localCache.Store(key, val)
    return val, nil
}

// 方案3：缓存预热（重启/活动前提前加载）
func preloadCache(rdb *redis.Client, db *sql.DB) error {
    // 把高频访问的数据提前写入 Redis
    hotItems, _ := db.Query("SELECT * FROM products WHERE hot_rank < 100")
    pipe := rdb.Pipeline()
    for hotItems.Next() {
        var item Product
        hotItems.Scan(&item.ID, &item.Name, &item.Price)
        data, _ := json.Marshal(item)
        key := fmt.Sprintf("product:%d", item.ID)
        // 不同 key 给不同 TTL，避免同时过期
        ttl := time.Duration(30+rand.Intn(30)) * time.Minute
        pipe.Set(context.Background(), key, data, ttl)
    }
    _, err := pipe.Exec(context.Background())
    return err
}
```

### 面试话术

> "我们有一次大促活动，活动结束后把商品缓存批量设了统一的 1 小时 TTL，结果 1 小时后缓存集体过期，数据库 QPS 从 2000 飙到 18000，MySQL 直接崩了。后来做了两个改进：一是所有缓存 TTL 加随机抖动；二是加了本地缓存作为第一层，即使 Redis 故障也能扛住 70% 的请求。"

---

## 2. 缓存击穿

### 什么是缓存击穿

```
热点 key（如明星微博、爆款商品）突然过期
→ 大量并发请求同时穿透缓存，全部打到数据库重建缓存
→ 数据库瞬间过载
```

### 方案一：互斥锁（同时只允许一个请求重建缓存）

```go
var mu sync.Map // 按 key 维度的互斥锁

func getWithMutex(rdb *redis.Client, db *sql.DB, key string) (string, error) {
    ctx := context.Background()

    // 先查缓存
    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        return val, nil
    }

    // 缓存未命中，用分布式锁确保只有一个请求重建
    lockKey := "lock:" + key
    locked, _ := rdb.SetNX(ctx, lockKey, 1, 5*time.Second).Result()

    if locked {
        // 获取到锁，查 DB 重建缓存
        defer rdb.Del(ctx, lockKey)

        val, err = queryFromDB(db, key)
        if err != nil {
            return "", err
        }
        rdb.Set(ctx, key, val, 10*time.Minute)
        return val, nil
    }

    // 没获取到锁，等待短暂时间后重试（其他请求正在重建）
    time.Sleep(100 * time.Millisecond)
    return rdb.Get(ctx, key).Result()
}
```

### 方案二：逻辑过期（不设 TTL，在 value 里存过期时间）

```go
type CacheValue struct {
    Data      string    `json:"data"`
    ExpireAt  time.Time `json:"expire_at"` // 逻辑过期时间
}

func getWithLogicalExpiry(rdb *redis.Client, db *sql.DB, key string) (string, error) {
    ctx := context.Background()

    data, err := rdb.Get(ctx, key).Bytes()
    if err != nil {
        // 缓存完全不存在（冷启动）
        return rebuildCache(rdb, db, key)
    }

    var cv CacheValue
    json.Unmarshal(data, &cv)

    if time.Now().Before(cv.ExpireAt) {
        return cv.Data, nil // 未过期，直接返回
    }

    // 已过期，异步重建，当前请求返回旧数据（不阻塞）
    lockKey := "lock:" + key
    locked, _ := rdb.SetNX(ctx, lockKey, 1, 5*time.Second).Result()
    if locked {
        go func() {
            defer rdb.Del(ctx, lockKey)
            rebuildCache(rdb, db, key)
        }()
    }

    return cv.Data, nil // 返回旧数据，用户不感知
}
```

---

## 3. 依赖服务宕机

### 面试官考察意图
微服务架构下服务间依赖是常态，考察候选人对熔断、降级的实战理解。

### 熔断状态机

```
  [关闭] ←————失败率低————← [半开]
    |                         ↑
    | 失败率超阈值            等待超时后尝试
    ↓                         |
  [打开] ——————————————→ [半开]
    |
    | 处于打开状态时直接返回降级数据
    ↓
  快速失败（不等待）
```

```go
import "github.com/sony/gobreaker"

// 为每个下游服务配置独立的熔断器
var orderServiceCB = gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "order-service",
    MaxRequests: 3,               // 半开状态允许3个试探请求
    Interval:    10 * time.Second,
    Timeout:     30 * time.Second, // 熔断后30秒进入半开
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 10 && failureRatio >= 0.5 // 10次请求中失败超50%
    },
    OnStateChange: func(name string, from, to gobreaker.State) {
        log.Warnf("熔断器 %s 状态变化: %v → %v", name, from, to)
        // 发告警
    },
})

func callOrderService(orderID string) (*Order, error) {
    result, err := orderServiceCB.Execute(func() (interface{}, error) {
        return httpGetOrder(orderID) // 实际的 HTTP 请求
    })

    if err == gobreaker.ErrOpenState {
        // 熔断中：返回降级数据（缓存、默认值、空）
        return getOrderFromCache(orderID), nil
    }
    if err != nil {
        return nil, err
    }
    return result.(*Order), nil
}
```

### 降级策略

```go
// 降级级别：从优到差
// 1. 返回缓存数据（用户无感知）
// 2. 返回降级兜底数据（返回默认值）
// 3. 返回错误提示（服务繁忙，请稍后）

type Fallback struct {
    cacheClient *redis.Client
}

func (f *Fallback) GetUserInfo(userID int) (*UserInfo, error) {
    // 主路径：调用用户服务
    user, err := callUserService(userID)
    if err == nil {
        // 成功，更新缓存
        cacheUser(f.cacheClient, userID, user)
        return user, nil
    }

    // 降级1：查本地缓存
    if cached := f.getFromCache(userID); cached != nil {
        log.Warn("用户服务异常，返回缓存数据")
        return cached, nil
    }

    // 降级2：返回基础数据（只保留必要字段）
    log.Warn("用户服务异常，返回降级数据")
    return &UserInfo{
        ID:   userID,
        Name: "用户" + strconv.Itoa(userID), // 兜底显示
    }, nil
}
```

---

## 4. 服务雪崩

### 什么是服务雪崩

```
服务A依赖服务B，服务B变慢
→ A的请求积压，线程/goroutine耗尽
→ A也变慢，C依赖A也跟着崩
→ 整个调用链全部崩溃
```

### 超时隔离

```go
// 每个下游调用都设置独立超时，不能让一个慢服务拖垮整体
func callWithIsolation(ctx context.Context, serviceName string, fn func() error) error {
    // 不同服务不同超时（SLA 越低，超时越短）
    timeouts := map[string]time.Duration{
        "payment":    500 * time.Millisecond,
        "inventory":  200 * time.Millisecond,
        "user":       300 * time.Millisecond,
        "recommend":  100 * time.Millisecond, // 非核心服务超时更短
    }

    timeout, ok := timeouts[serviceName]
    if !ok {
        timeout = 300 * time.Millisecond
    }

    ctx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()

    done := make(chan error, 1)
    go func() {
        done <- fn()
    }()

    select {
    case err := <-done:
        return err
    case <-ctx.Done():
        return fmt.Errorf("%s 调用超时", serviceName)
    }
}
```

### goroutine 信号量隔离

```go
// 限制每个服务的并发调用数，防止某个慢服务占满所有 goroutine
type Semaphore struct {
    ch chan struct{}
}

func NewSemaphore(capacity int) *Semaphore {
    return &Semaphore{ch: make(chan struct{}, capacity)}
}

func (s *Semaphore) Acquire(timeout time.Duration) bool {
    select {
    case s.ch <- struct{}{}:
        return true
    case <-time.After(timeout):
        return false // 超时未获取到，快速失败
    }
}

func (s *Semaphore) Release() {
    <-s.ch
}

// 使用
var paymentSem = NewSemaphore(50) // 支付服务最多50个并发

func callPayment(req PaymentReq) (*PaymentResp, error) {
    if !paymentSem.Acquire(100 * time.Millisecond) {
        return nil, errors.New("支付服务繁忙，请稍后重试")
    }
    defer paymentSem.Release()

    return doCallPayment(req)
}
```

---

## 5. 流量不均衡/热点问题

### 面试官考察意图
负载均衡是系统设计基础，热点问题考察候选人对真实流量分布的理解。

### 常见负载均衡策略对比

| 策略 | 原理 | 适用场景 |
|------|------|---------|
| 轮询（Round Robin） | 依次分配 | 服务性能均等 |
| 加权轮询 | 按权重分配 | 服务器性能差异大 |
| 最少连接 | 选当前连接最少的 | 请求处理时长差异大 |
| 一致性哈希 | 按请求 key hash | 需要会话保持（如缓存） |

### 一致性哈希解决缓存热点

```go
import "github.com/stathat/consistent"

// 一致性哈希：扩缩容时只有少量 key 迁移
var ring = consistent.New()

func init() {
    ring.Add("redis-node-1:6379")
    ring.Add("redis-node-2:6379")
    ring.Add("redis-node-3:6379")
}

func getRedisNode(key string) string {
    node, _ := ring.Get(key)
    return node
}

// 访问时自动路由到对应节点
func getCacheValue(key string) (string, error) {
    node := getRedisNode(key)
    rdb := getRedisClient(node)
    return rdb.Get(context.Background(), key).Result()
}
```

### 热点路由（将热点请求路由到专用实例）

```go
// 识别热点：滑动窗口统计访问频率
type HotKeyDetector struct {
    mu      sync.RWMutex
    counter map[string]int64
    window  time.Duration
    thresh  int64
}

func (d *HotKeyDetector) IsHot(key string) bool {
    d.mu.RLock()
    defer d.mu.RUnlock()
    return d.counter[key] >= d.thresh
}

func (d *HotKeyDetector) Record(key string) {
    d.mu.Lock()
    d.counter[key]++
    d.mu.Unlock()
}

// 路由：热点 key 打到专用的高性能实例
func getSmartRoute(key string) *redis.Client {
    detector.Record(key)
    if detector.IsHot(key) {
        return hotKeyRedis // 专门处理热 key 的实例
    }
    return normalRedis
}
```

---

## 高频追问汇总

| 问题 | 核心答案 |
|------|---------|
| 缓存雪崩、击穿、穿透的区别？ | 雪崩=大量key同时失效；击穿=单个热key失效；穿透=查询不存在的key（缓存和DB都没有）。 |
| 缓存穿透怎么解决？ | 布隆过滤器（提前过滤不存在的key）；或把"不存在"也缓存起来（空值缓存，TTL设短）。 |
| 熔断器三个状态怎么转换？ | 关闭→打开：失败率超阈值；打开→半开：等待超时；半开→关闭：试探请求成功；半开→打开：试探请求失败。 |
| 限流和降级的区别？ | 限流：拒绝超额请求，保护服务不被压垮；降级：服务出问题时返回简化/默认结果，保证部分可用。 |
| 如何做故障演练？ | Chaos Engineering：在生产（或预发）随机注入故障（网络延迟、服务宕机），验证系统韧性；工具：Chaos Monkey、ChaosBlade。 |
