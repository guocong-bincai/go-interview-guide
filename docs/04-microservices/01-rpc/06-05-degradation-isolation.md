# 服务降级与隔离：舱壁模式（Bulkhead）

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：服务降级、降级开关、舱壁模式、线程池隔离、信号量隔离、兜底数据、雪崩防护

---

## 面试官考察意图

熔断、限流之后，面试官必问的第三板斧就是**降级与隔离**。初级候选人多停留在"降级就是返回兜底数据"的表面理解；高级候选人要能讲清楚：

1. **降级的分类**：主动降级（开关）vs 被动降级（熔断触发）
2. **隔离的两种实现**：线程池隔离 vs 信号量隔离，各自优劣与适用场景
3. **舱壁模式（Bulkhead）** 为什么能防止"一个慢服务拖垮整个进程"
4. 降级/熔断/限流/超时四者如何配合构成完整的容错体系

这是微服务防雪崩的核心考点，几乎每家大厂后端面试都会涉及。

---

## 核心答案（30 秒版）

- **降级**：系统资源紧张或依赖故障时，**主动牺牲非核心功能**，保证核心链路可用（如推荐失败返回默认列表、支付失败走余额兜底）。
- **隔离**：把不同依赖的调用资源（线程/信号量/连接池）**物理隔开**，一个依赖耗尽资源不影响其他依赖。
- **舱壁模式** 类比轮船的防水舱：一个舱进水不会沉船。对应到服务，就是按依赖**分池隔离**，互不干扰。

| 容错手段 | 作用 | 触发时机 |
|---------|------|---------|
| 限流 | 控制进入的流量 | 事前，主动 |
| 熔断 | 快速失败，停止调用故障依赖 | 事中，被动 |
| **降级** | 返回兜底结果，保证主流程 | 事中，主动/被动 |
| **隔离** | 资源分舱，防止级联耗尽 | 事前，主动 |

---

## 深度展开

### 1. 服务降级的分类

#### 1.1 按触发方式分

| 类型 | 说明 | 例子 |
|------|------|------|
| **主动降级**（开关降级） | 运维/配置中心手动打开降级开关，提前应对大促 | 双11前关闭"猜你喜欢"个性化，返回热门列表 |
| **被动降级**（异常降级） | 依赖超时/熔断后自动返回兜底 | 库存服务超时 → 返回"库存充足"的乐观兜底 |

#### 1.2 按降级粒度分

- **接口级降级**：整个接口返回 fallback（默认值/空数据/缓存数据）
- **功能级降级**：关闭非核心功能模块（评论、点赞、推荐）
- **数据级降级**：实时数据 → 缓存数据 → 默认数据，逐级降级

#### 1.3 降级开关设计（高频追问）

降级开关必须**独立于代码发布**，通过配置中心动态下发：

```go
// 降级开关（配置中心热更新，如 Apollo/Nacos/etcd watch）
type DegradeSwitch struct {
    mu        sync.RWMutex
    recommend bool // 推荐功能开关
    comment   bool // 评论功能开关
    ratio     int  // 降级比例 0-100
}

// 配置中心回调里更新
func (s *DegradeSwitch) Update(cfg Config) {
    s.mu.Lock()
    s.recommend = cfg.RecommendOn
    s.comment = cfg.CommentOn
    s.ratio = cfg.DegradeRatio
    s.mu.Unlock()
}

func (s *DegradeSwitch) RecommendEnabled() bool {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.recommend
}
```

**开关设计要点**：
- 开关默认关闭（不降级），只有出现故障/预案时才打开
- 支持**按比例降级**（先放 10% 流量试水，观察核心指标）
- 开关状态要**可观测**（降级了多少流量、影响了哪些用户，都要有指标）
- 必须支持**一键恢复**

### 2. 舱壁模式（Bulkhead）：线程池隔离 vs 信号量隔离

#### 2.1 为什么需要隔离？

```text
❌ 无隔离（共用线程池）：
                 ┌─────────────┐
 请求 →→→→→→→→→→→ 共享线程池(200) │← 慢依赖A 占满 200 个线程
                 └─────────────┘
 依赖B 的请求全部排队等线程 → 整个服务雪崩

✅ 舱壁隔离（按依赖分池）：
        ┌──────────┐
 依赖A →│ 线程池A(50)│  ← A 慢了只耗尽 A 的池
        └──────────┘
        ┌──────────┐
 依赖B →│ 线程池B(50)│  ← B 正常响应，互不影响
        └──────────┘
```

#### 2.2 线程池隔离（Hystrix 默认模式）

每个依赖分配独立线程池，线程池满则快速失败（走降级）。

```go
// 为每个下游依赖建立独立 goroutine 池（简易实现）
type DependencyPool struct {
    name     string
    sem      chan struct{} // 信号量池
    timeout  time.Duration
}

func NewDependencyPool(name string, size int, timeout time.Duration) *DependencyPool {
    return &DependencyPool{name: name, sem: make(chan struct{}, size), timeout: timeout}
}

func (p *DependencyPool) Call(ctx context.Context, fn func(ctx context.Context) (any, error)) (any, error) {
    // 尝试获取"舱位"，拿不到说明该依赖的池已满 → 快速失败走降级
    select {
    case p.sem <- struct{}{}:
    default:
        return nil, ErrPoolExhausted // 池满：不等待，立即降级
    }
    defer func() { <-p.sem }()

    done := make(chan result, 1)
    go func() {
        v, err := fn(ctx)
        done <- result{v, err}
    }()
    select {
    case r := <-done:
        return r.v, r.err
    case <-time.After(p.timeout):
        return nil, ErrTimeout
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

**优点**：彻底隔离（线程数、排队队列都独立），慢依赖不拖垮整体。
**缺点**：线程/goroutine 有开销，上下文切换成本高，线程数需要调优。

#### 2.3 信号量隔离（Sentinel / Resilience4j 默认模式）

不新起 goroutine，只限制**并发调用数**，超限直接拒绝。

```go
// 信号量隔离：只限制并发数，不隔离线程
type SemaphoreIsolation struct {
    sem     chan struct{}
    timeout time.Duration
}

func NewSemaphoreIsolation(maxConcurrent int, timeout time.Duration) *SemaphoreIsolation {
    return &SemaphoreIsolation{sem: make(chan struct{}, maxConcurrent), timeout: timeout}
}

func (s *SemaphoreIsolation) Call(ctx context.Context, fn func() error) error {
    select {
    case s.sem <- struct{}{}:
        defer func() { <-s.sem }()
    default:
        return ErrSemaphoreExhausted // 并发数超限，快速失败
    }
    done := make(chan error, 1)
    go func() { done <- fn() }() // 注意：实际生产常用调用方 goroutine 直接执行
    select {
    case err := <-done:
        return err
    case <-time.After(s.timeout):
        return ErrTimeout
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

**优点**：轻量、无线程切换开销，性能好。
**缺点**：调用方线程可能被慢依赖阻塞（如果同步调用），隔离性弱于线程池。

#### 2.4 如何选型？（面试高频追问）

| 维度 | 线程池隔离 | 信号量隔离 |
|------|-----------|-----------|
| 隔离强度 | 强（线程/队列都独立） | 弱（只限并发数） |
| 性能开销 | 高（线程切换） | 低（无切换） |
| 适用场景 | 延迟敏感、依赖差异大 | 低延迟、高频调用（网关/内部RPC） |
| 典型实现 | Hystrix | Sentinel、Resilience4j |

> 面试话术：**线程池隔离适合"慢依赖"场景（延迟差异大），信号量隔离适合"快依赖"高并发场景（追求低开销）。Go 里 goroutine 很廉价，通常用信号量 + goroutine 组合模拟线程池隔离，成本可控。**

### 3. 降级 + 熔断 + 限流 + 超时：完整容错链路

```text
请求进入
  │
  ▼
① 限流：QPS 超限？ → 拒绝（返回 429）
  │
  ▼
② 熔断：依赖处于 Open？ → 直接降级（不发起调用）
  │
  ▼
③ 隔离：该依赖池/信号量满？ → 快速失败降级
  │
  ▼
④ 超时：调用超过 deadline？ → 取消并降级
  │
  ▼
⑤ 成功返回
```

**关键认知**：四者层层递进——限流在最前面挡流量，熔断和隔离保护自身资源，降级是最后兜底保证"有响应而不是挂掉"。

### 4. 降级兜底策略实战

```go
// 推荐位接口：三级降级
func (s *Service) GetRecommend(ctx context.Context, uid int64) ([]Item, error) {
    // 第一级：降级开关判断（主动降级）
    if !s.switchs.RecommendEnabled() {
        return s.hotItems(ctx) // 兜底：热门列表（读缓存）
    }

    items, err := s.recall(ctx, uid) // 个性化召回（调用重依赖）
    if err != nil {
        // 第二级：异常降级
        log.Warn("recall failed, fallback to hot", "err", err)
        if hot, err2 := s.hotItems(ctx); err2 == nil {
            return hot, nil
        }
        // 第三级：最终兜底（静态默认数据，保证有响应）
        return defaultItems, nil
    }
    return items, nil
}
```

**降级设计三原则**：
1. **主链路优先**：降级永远先牺牲非核心（推荐、评论），保住核心（下单、支付）
2. **兜底必须有**：任何外部调用都要有 fallback（默认值/缓存/静态数据），不能裸调
3. **可观测**：降级次数、降级比例、降级原因都要有指标和告警（降级是"有损"的，必须有感知）

---

## 高频追问

### Q1：熔断和降级的区别？
> 熔断是**触发机制**（依赖故障达到阈值后停止调用），降级是**处理策略**（返回什么兜底结果）。熔断后必然走降级，但降级不一定要熔断（主动降级）。一句话：**熔断是"不调了"，降级是"调不了给什么"。**

### Q2：降级和限流的区别？
> 限流是**挡外部流量**（保护系统不被打垮），降级是**内部取舍**（牺牲非核心保核心）。限流是"少干活"，降级是"挑着干活"。

### Q3：Go 里 goroutine 这么便宜，还需要线程池隔离吗？
> 需要。隔离的本质不是"省资源"而是"限制资源占用上限"——防止某个依赖的故障导致 goroutine 无限堆积（内存暴涨、GC 压力）。Go 里常用 **信号量（chan struct{}）** 限流 + 超时控制实现轻量舱壁，成本远低于 Java 线程池。

### Q4：降级开关放配置中心，配置中心挂了怎么办？
> 本地缓存 + 默认值兜底：进程启动时拉取全量配置缓存到本地，配置中心不可用时用本地缓存，本地也没有就用代码内置默认值（通常默认不降级）。

---

## 延伸阅读

- 本模块：[熔断与限流](01-02-circuit-breaker.md)（三态熔断 + 令牌桶/漏桶/滑动窗口）
- 本模块：[服务治理：超时、重试、负载均衡](05-03-service-governance.md)
- 分布式模块：[限流熔断降级体系](../../03-distributed/04-service-mesh/17-04-rate-limiter-circuit-breaker.md)
- 项目实战：[服务雪崩与可用性保障](../../10-real-problems/04-04-availability-problems.md)
