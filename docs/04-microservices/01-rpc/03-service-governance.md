# 服务治理：超时、重试、负载均衡

> 考察频率：★★★★☆  关键词：超时传递、指数退避、连接复用、金丝雀

## 一、超时控制

### 为什么重要
- 防止请求无限等待（依赖服务挂了）
- 及时释放资源（goroutine、连接）
- 避免级联故障（一个服务慢拖垮所有依赖它的服务）

### Go 中的超时控制

#### Context 超时

```go
// 服务端：设置 deadline
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 300*time.Millisecond)
    defer cancel()

    result, err := fetchData(ctx, "some-data")
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            http.Error(w, "timeout", 504)
            return
        }
        http.Error(w, "server error", 500)
        return
    }
    json.NewEncoder(w).Encode(result)
}
```

#### HTTP Client 超时

```go
client := &http.Client{
    Timeout: 5 * time.Second,
}

// 精细化超时：Transport 层
tr := &http.Transport{
    DialContext: (&net.Dialer{
        Timeout:   30 * time.Second,  // 建立连接超时
    }).DialContext,
    ResponseHeaderTimeout: 5 * time.Second, // 读取响应头超时
    IdleConnTimeout:       90 * time.Second, // 空闲连接超时
}
```

#### gRPC 超时

```go
// 客户端调用时指定超时
ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
defer cancel()

resp, err := client.SendPayment(ctx, req)
if ctx.Err() == context.DeadlineExceeded {
    // 超时处理
}
```

### 超时设计原则

| 原则 | 说明 |
|------|------|
| 客户端超时 > 服务端处理时间 | 留 buffer 给网络 RTT |
| 不同层级超时可叠加 | 用 context 传递，不要各自独立 |
| 超时要有监控告警 | 比例超过阈值要报警 |
| 超时后要幂等 | upstream 超时，downstream 可能已经执行了 |

---

## 二、重试机制

### 重试三要素
1. **什么情况下重试**：网络错误、超时（5xx 错误）、连接失败
2. **重试多少次**：通常 2-4 次
3. **间隔多久重试**：指数退避 + 抖动

### 指数退避（Exponential Backoff）

```go
func backoff(attempt int) time.Duration {
    base := 100 * time.Millisecond
    maxDelay := 5 * time.Second

    // 指数退避：100ms * 2^attempt，上限 5s
    delay := base * time.Duration(1<<attempt)
    if delay > maxDelay {
        delay = maxDelay
    }

    // 加随机抖动，避免惊群
    jitter := time.Duration(rand.Int63n(int64(delay / 2)))
    return delay + jitter
}
```

### Go gRPC 重试示例

```go
// gRPC 重试策略（服务端）
grpcServer := grpc.NewServer(
    grpc.UnaryInterceptor(func(ctx context.Context, req any,
        info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        return retryUnary(ctx, req, info, handler)
    }),
)

func retryUnary(ctx context.Context, req any,
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    const maxRetries = 3
    for attempt := 0; attempt <= maxRetries; attempt++ {
        if attempt > 0 {
            time.Sleep(backoff(attempt - 1))
        }
        resp, err := handler(ctx, req)
        if err == nil {
            return resp, nil
        }
        // 只对可重试错误重试
        if !isRetryable(err) {
            return nil, err
        }
    }
    return nil, err
}

func isRetryable(err error) bool {
    // 超时、临时性网络错误可重试
    // 业务逻辑错误（参数错误）不可重试
    return status.Code(err) == codes.Unavailable ||
           status.Code(err) == codes.DeadlineExceeded
}
```

### 重试的坑

| 坑 | 后果 | 解决方案 |
|---|------|---------|
| 无限制重试 | 雪崩，服务挂了还在疯狂重试 | 限制最大重试次数 |
| 无抖动 | 大量请求同时重试（惊群）| 加随机 jitter |
| 非幂等操作重试 | 数据重复 | 读操作可重试，写操作需幂等 |
| 超时传递没做好 | 下游超时后上游重试，实际已执行 | 幂等键、查询后再操作 |

---

## 三、负载均衡

### 常见策略

| 策略 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| Round Robin | 轮询 | 简单、均匀 | 不考虑负载 |
| 加权轮询 | 按权重轮询 | 性能差异 | 需要配置权重 |
| 最少连接 | 选连接数最少的 | 适合长连接 | 需要维护连接数 |
| 一致性哈希 | 哈希环 | 缓存友好 | 负载不均 |
| 随机 | 随机选择 | 实现简单 | 不均匀 |

### Go gRPC 负载均衡

```go
// 方案1：gRPC 内置 round_robin
conn, err := grpc.Dial(
    "passthrough:///consul://localhost:8500?health=true",
    grpc.WithBalancerName("round_robin"),
)

// 方案2：自己实现 picker
import "google.golang.org/grpc/balancer"
import "google.golang.org/grpc/balancer/base"
```

### 一致性哈希

```go
type ConsistentHash struct {
    hashFunc func([]byte) uint32
    virtualNodes int
    ring        map[uint32]string // hash -> 真实节点
    sortedKeys   []uint32         // 已排序的哈希值
}

func NewConsistentHash(nodes []string, virtualNodes int) *ConsistentHash {
    ch := &ConsistentHash{
        hashFunc:    crc32.ChecksumIEEE,
        virtualNodes: virtualNodes,
        ring:        make(map[uint32]string),
        sortedKeys:  make([]uint32, 0),
    }
    for _, node := range nodes {
        ch.Add(node)
    }
    return ch
}

func (ch *ConsistentHash) Add(node string) {
    for i := 0; i < ch.virtualNodes; i++ {
        hash := ch.hashFunc([]byte(fmt.Sprintf("%s#%d", node, i))))
        ch.ring[hash] = node
        ch.sortedKeys = append(ch.sortedKeys, hash)
    }
    sort.Slice(ch.sortedKeys, func(i, j int) bool {
        return ch.sortedKeys[i] < ch.sortedKeys[j]
    })
}

func (ch *ConsistentHash) Get(key string) string {
    if len(ch.ring) == 0 {
        return ""
    }
    hash := ch.hashFunc([]byte(key))
    // 二分找第一个 >= hash 的节点
    idx := sort.Search(len(ch.sortedKeys), func(i int) bool {
        return ch.sortedKeys[i] >= hash
    })
    if idx == len(ch.sortedKeys) {
        idx = 0 // 环，回到第一个
    }
    return ch.ring[ch.sortedKeys[idx]]
}
```

---

## 四、熔断器（Circuit Breaker）

> 详见 `02-circuit-breaker/02-circuit-breaker.md`，这里补充服务治理层面的实践。

### 熔断与超时/重试的关系

```
超时：防止等太久 → 快速失败
重试：临时失败再试 → 增加成功率
熔断：失败率太高不再试 → 防止雪崩
```

三者组合：超时（保底）+ 重试（容错）+ 熔断（保护）。

---

## 五、实战：如何设计一个靠谱的 RPC 调用

```go
func callRPC(ctx context.Context, client pb.PaymentClient,
    req *pb.PaymentRequest) (*pb.PaymentResponse, error) {

    // 1. 设置超时（防止等太久）
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    // 2. 重试（指数退避，最多 3 次）
    var lastErr error
    for attempt := 0; attempt < 3; attempt++ {
        if attempt > 0 {
            time.Sleep(backoff(attempt))
        }

        resp, err := client.Pay(ctx, req)
        if err == nil {
            return resp, nil
        }

        // 不可重试的错误直接返回
        if !isRetryable(err) {
            return nil, err
        }
        lastErr = err

        // 熔断：context 已超时就不再重试
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        default:
        }
    }
    return nil, lastErr
}
```

### 服务治理 Checklist

- [ ] 每个外部调用都设置了合理的超时
- [ ] 重试加了指数退避 + jitter
- [ ] 重试只针对可重试的错误码
- [ ] 关键路径有熔断保护
- [ ] 幂等操作才能安全重试
- [ ] 超时、重试、熔断都有监控指标
- [ ] 用 context 传递超时，而不是创建新的 timeout

---

## 面试高频追问

1. **超时设置的依据？**
   → 压测 P99 延迟 × 2 + 预留 30% buffer + 网络 RTT

2. **重试会带来幂等问题怎么办？**
   → 幂等键（每次请求带唯一 ID），服务端用 Redis 记录已处理过的 ID

3. **熔断后的流量怎么处理？**
   → 直接返回降级数据，或走备份链路（fallback）

4. **服务发现怎么做的？**
   → Consul/etcd/Nacos，注册中心 + 健康检查 + 客户端缓存
