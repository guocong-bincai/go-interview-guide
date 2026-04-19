# 并发编程实战问题

> 考察频率：★★★★★  优先级：P0

---

## 1. goroutine 池设计

### 面试官考察意图

考察候选人有没有实际设计过并发资源控制机制。初级工程师会直接 `go func()` 无限制创建协程，高级工程师知道资源是有限的，必须池化管理。这道题考察的是**生产级并发控制能力**，以及有没有踩过协程泄漏/资源耗尽的坑。

### 核心答案（30秒版）

goroutine 池的核心思路是**复用**：预先创建固定数量的 worker 协程，从任务队列取任务执行，避免无限制创建协程导致的 OOM 和调度开销。Go 生态推荐 `ants`、`tun` 这类库，生产环境直接用，不要自己造轮子。

### 深度展开

#### 为什么需要 goroutine 池

Go 的协程很轻（2KB 栈），但不是免费的：

```go
// ❌ 无限制创建协程 — 生产禁用
for i := 0; i < 100000; i++ {
    go func() {
        doWork() // 如果 doWork 占用内存，10万协程直接 OOM
    }()
}
```

一个真实踩坑案例：
- 场景：处理 10 万个文件上传，每个上传开一个 goroutine 调第三方接口
- 问题：10 万协程同时存在，内存从 500MB 飙升到 8GB，GC STW 时间达到秒级
- 根因：goroutine 数量 = 并发请求数，第三方接口慢导致协程堆积
- 解决：限制最大并发数为 CPU 核数的 2~4 倍，用 semaphore 或 worker pool

```go
// ✅ 用 semaphore 限制并发数
var (
    sema    = make(chan struct{}, 100) // 最多100并发
    wg      sync.WaitGroup
)

for i := 0; i < 100000; i++ {
    sema <- struct{}{}    // 获取信号量，满了就阻塞
    wg.Add(1)
    go func() {
        defer func() { <-sema }() // 释放信号量
        defer wg.Done()
        doWork()
    }()
}
wg.Wait()
```

#### 核心结构

```
┌─────────────────────────────────────────────┐
│              goroutine 池架构                │
├─────────────────────────────────────────────┤
│                                             │
│   Task Queue (channel)                      │
│   ┌────┬────┬────┬────┬────┬────┐         │
│   │ T1 │ T2 │ T3 │ T4 │ T5 │ .. │         │
│   └────┴────┴────┴────┴────┴────┘         │
│          ▲         │                        │
│          │         │                        │
│   ┌──────┴─────────┴────────┐              │
│   │      N 个 Worker          │              │
│   │  ┌────┐  ┌────┐  ┌────┐ │              │
│   │  │ G1 │  │ G2 │  │ G3 │ │              │
│   │  └──┬─┘  └──┬─┘  └──┬─┘ │              │
│   └─────┼───────┼───────┼───┘              │
│         ▼       ▼       ▼                  │
│       doWork()  work()  task()            │
│                                             │
└─────────────────────────────────────────────┘
```

#### 手写一个简单 goroutine 池

```go
type Pool struct {
    workers  int
    tasks    chan func()            // 任务队列
    wg       sync.WaitGroup
    quit     chan struct{}
}

func NewPool(workers int, queueLen int) *Pool {
    p := &Pool{
        workers: workers,
        tasks:   make(chan func(), queueLen),
        quit:    make(chan struct{}),
    }
    p.wg.Add(workers)
    for i := 0; i < workers; i++ {
        go p.worker(i)
    }
    return p
}

func (p *Pool) worker(id int) {
    defer p.wg.Done()
    for {
        select {
        case task := <-p.tasks:
            task()
        case <-p.quit:
            fmt.Printf("worker %d shutdown\n", id)
            return
        }
    }
}

// Submit 提交任务，非阻塞，超queueLen直接panic
func (p *Pool) Submit(task func()) {
    p.tasks <- task
}

// Close 等待所有任务执行完然后关闭
func (p *Pool) Close() {
    close(p.quit)
    p.wg.Wait()
}

// 使用示例
func main() {
    pool := NewPool(10, 1000) // 10个worker，队列最多1000
    defer pool.Close()

    for i := 0; i < 100; i++ {
        pool.Submit(func() {
            // 实际任务
        })
    }
}
```

#### ants 库核心设计

生产环境推荐直接用 `ants`（一个 Go 协程池库）：

```go
import "github.com/panjf2000/ants/v2"

func main() {
    defer ants.Release()

    // 提交10万个任务，只用10个协程执行
    var wg sync.WaitGroup
    for i := 0; i < 100000; i++ {
        wg.Add(1)
        ants.Submit(func() {
            defer wg.Done()
            doWork()
        })
    }
    wg.Wait()
}

// 自定义池大小
p, _ := ants.NewPoolWithFunc(10, func(i interface{}) {
    defer wg.Done()
    doWork(i)
})
defer p.Release()
```

ants 的核心优势：
- 基于 sync.Pool 复用 worker，减少创建/销毁开销
- 提供 PoolWithFunc（任务有返回值）和 Pool（任务无返回值）
- 支持动态调整池大小（SetCap）
- 非阻塞 Submit，队列满返回 error

#### 何时需要 goroutine 池

| 场景 | 是否需要 | 原因 |
|------|---------|------|
| 高并发 HTTP 请求处理 | 不需要 | 每个请求一个 goroutine，Go 调度器处理 |
| 批量调用第三方接口（100+并发）| **需要** | 防止协程爆炸 + 限流 |
| 异步任务队列消费者 | **需要** | 固定 worker 数，控制消费速率 |
| 一次性并发任务（10~100）| 不需要 | 直接 go func() 即可 |

### 高频追问

**Q：goroutine 池和信号量限流的区别？**
> 两者都能限制并发数，但 goroutine 池还负责**任务队列缓冲**和**worker 复用**。信号量只是计数器，不负责存储任务。如果任务来源速率不稳定（生产者快、消费者慢），必须用池+队列；如果只是限制同时发起多少个请求，信号量就够了。

**Q：任务执行时间差异很大怎么办？**
> 如果有的任务执行 1ms，有的执行 10s，固定池大小会导致短任务饿死长任务。解决方案：1）长短任务分池；2）用加权公平队列（priority queue）；3）使用 worker pool 的动态调度变体（如 `tun` 库）。

---

## 2. 并发安全的本地缓存

### 面试官考察意图

考察候选人对 Go 并发安全的理解深度：什么时候需要加锁、什么时候用 sync.Map、什么时候应该用分片锁。这道题没有标准答案，关键是能够**根据读写比例选择最优方案**，并说出 trade-off。

### 核心答案（30秒版）

sync.Map 不是银弹：它适合读多写少、key 稳定的场景。写多场景下 `map+RWMutex` 性能更好。如果对并发要求极高，用分片锁（sharding map）把锁粒度降到 1/16。生产中最推荐的是根据业务场景选型。

### 深度展开

#### 为什么原生 map 不是并发安全的

```go
// ❌ 竞态条件，数据可能丢失
var m = make(map[string]int)

func write() {
    m["key"] = 1 // 不是原子操作
}

func read() {
    _ = m["key"] // 可能读到半个写入
}

// ✅ 编译报错：fatal error: concurrent map read and map write
```

Go 的 map 内部有 resize 机制，并发读写会导致 data race 检测器直接 panic。

#### sync.Map 原理

```go
// sync.Map 核心是双 map + 原子操作
type Map struct {
    read   atomic.Value // 存储 readOnly，读取不加锁
    dirty  map[string]*entry // 脏数据，需要锁
    misses int // 读miss计数，超过阈值提升dirty
}

// readOnly 中 entry 可能被标记为 deleted
type entry struct {
    p unsafe.Pointer // *interface{} 或 nil 或 expunged
}
```

读流程（无锁）：
1. 从 read.map 查，找到了检查是否 deleted
2. 没找到，锁住，从 dirty.map 查（同时 miss++）
3. miss 超过阈值，把 dirty 提升为新的 read.map

写流程（加锁）：
1. 写 lock
2. 写 dirty.map
3. unlock

#### sync.Map vs map+RWMutex 性能对比

```go
// 基准测试：100%读，1000个key
// sync.Map:   ~300ns/op
// RWMutex+map: ~80ns/op（读完全无锁优势）

// 基准测试：50%读50%写，1000个key
// sync.Map:   ~800ns/op（dirty提升开销大）
// RWMutex+map: ~400ns/op（写锁粒度更可控）
```

结论：读多写少（9:1以上）用 sync.Map；否则用 RWMutex+map。

#### 分片锁实现

```go
type ShardedMap struct {
    shards []*shard // 16个分片
}

type shard struct {
    mu  sync.RWMutex
    m   map[string]interface{}
}

func NewShardedMap(nShards int) *ShardedMap {
    shards := make([]*shard, nShards)
    for i := 0; i < nShards; i++ {
        shards[i] = &shard{m: make(map[string]interface{})}
    }
    return &ShardedMap{shards: shards}
}

func (m *ShardedMap) shard(key string) *shard {
    // 字符串哈希定位分片
    h := fnv32(key)
    return m.shards[h%uint32(len(m.shards))]
}

func (m *ShardedMap) Get(key string) (interface{}, bool) {
    s := m.shard(key)
    s.mu.RLock()
    defer s.mu.RUnlock()
    val, ok := s.m[key]
    return val, ok
}

func (m *ShardedMap) Set(key string, val interface{}) {
    s := m.shard(key)
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = val
}

func fnv32(key string) uint32 {
    h := fnv.New32a()
    h.Write([]byte(key))
    return h.Sum32()
}
```

分片锁性能（16分片 vs 单锁）：
- 16并发读：分片锁 ~1.6M ops/s，单锁 ~400K ops/s（4倍差距）
- 16并发写：分片锁 ~800K ops/s，单锁 ~200K ops/s

#### 带 TTL 的缓存实现

```go
type TTLCache struct {
    mu    sync.RWMutex
    items map[string]*cacheItem
}

type cacheItem struct {
    value      interface{}
    expireTime time.Time
}

// 定期清理过期数据（启动一个后台goroutine）
func NewTTLCache(ttl time.Duration, cleanupInterval time.Duration) *TTLCache {
    c := &TTLCache{items: make(map[string]*cacheItem)}
    go c.cleanup(ttl, cleanupInterval)
    return c
}

func (c *TTLCache) cleanup(ttl, interval time.Duration) {
    ticker := time.NewTicker(interval)
    for range ticker.C {
        c.mu.Lock()
        now := time.Now()
        for k, v := range c.items {
            if now.After(v.expireTime) {
                delete(c.items, k)
            }
        }
        c.mu.Unlock()
    }
}

func (c *TTLCache) Set(key string, val interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = &cacheItem{
        value:      val,
        expireTime: time.Now().Add(5 * time.Minute), // 默认5分钟TTL
    }
}

func (c *TTLCache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    item, ok := c.items[key]
    if !ok || time.Now().After(item.expireTime) {
        return nil, false
    }
    return item.value, true
}
```

### 高频追问

**Q：sync.Map 的 dirty 提升过程会发生什么？**
> 当 read 中找不到key时（miss），会锁住查 dirty。如果 misses 超过 len(dirty) 的阈值，就把整个 dirty 复制一份作为新的 read，并清空 dirty。这个过程叫"提升"。提升后新请求从 read 读就不用加锁了。但如果频繁写，提升会很慢，导致性能抖动。

**Q：本地缓存和分布式缓存（如 Redis）怎么选？**
> 本地缓存：极低延迟（ns级）、无网络开销，但不支持跨实例共享、有不一致风险。
> 分布式缓存：多实例一致、容量大，但有网络延迟。
> 实际方案通常是两级缓存：Redis 做分布式缓存层，本地 GCache 做一级缓存（热点数据）。需要解决的是：如何让多个实例的本地缓存保持相对新鲜——常用 TTL + 变更时主动失效。

---

## 3. 优雅关闭（Graceful Shutdown）

### 面试官考察意图

考察候选人有没有真正在生产环境部署过服务。K8s 滚动更新、灰度发布都依赖 SIGTERM 优雅关闭。如果候选人只能说 `kill -9`，说明没有线上发布经验。这道题考察的是**服务稳定性意识**和**信号处理能力**。

### 核心答案（30秒版）

优雅关闭的核心是**收场**：收到 SIGTERM 后，停止接收新请求、等待正在处理的请求完成、最后再退出。Go 标准库提供了 `http.Server.Shutdown(context)`，配合 `sync.WaitGroup` 等待请求处理完。超时强制退出防止僵死。

### 深度展开

#### K8s 滚动更新原理

```
K8s 滚动更新：
1. 发送 SIGTERM 到旧 Pod
2. 等待就绪探针失败，新流量切走
3. 旧 Pod 内部执行优雅关闭
4. Pod 真正退出

用户视角：0  downtime（如果优雅关闭 < 30s）
```

如果优雅关闭没做好：Pod 收到 SIGTERM 后立即退出，正在处理的请求直接 reset，用户看到 502。

#### Go 优雅关闭实现

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: mux}

    // 创建 goroutine 启动服务
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %v\n", err)
        }
    }()

    // 等待 SIGTERM/SIGINT
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    sig := <-quit
    log.Printf("收到信号 %v，开始优雅关闭...", sig)

    // 创建有超时的 context
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Shutdown：停止接收新连接 + 等待现有连接处理完
    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("优雅关闭超时，强制退出: %v", err)
    }

    log.Println("服务已关闭")
}
```

#### 结合 WaitGroup 追踪所有请求

```go
type Server struct {
    srv    *http.Server
    active int64 // 正在处理的请求数
    wg     sync.WaitGroup
    mu     sync.Mutex
}

func (s *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 请求开始
    s.mu.Lock()
    s.active++
    s.mu.Unlock()

    defer func() {
        s.mu.Lock()
        s.active--
        s.mu.Unlock()
        s.wg.Done()
    }()

    // 业务处理
    s.handle(w, r)
}

func (s *Server) handle(w http.ResponseWriter, r *http.Request) {
    // 处理请求...
}

// 等待所有活跃请求完成
func (s *Server) WaitForZeroActive() {
    for {
        s.mu.Lock()
        active := s.active
        s.mu.Unlock()
        if active == 0 {
            break
        }
        time.Sleep(100 * time.Millisecond)
    }
}
```

#### 实际踩坑案例

**案例：第三方接口超时导致优雅关闭卡住**

```go
// 问题：某个请求处理中调第三方接口，等了60s
// Shutdown给了30s超时，导致请求被强制中断，数据不一致

// ✅ 正确做法：所有外部调用都要有超时控制
func handle(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    // 这个请求最多等5秒
    resp, err := httpClient.Do(req.WithContext(ctx))
    if err != nil {
        // 处理超时错误
        http.Error(w, "timeout", 504)
        return
    }
    // 继续处理...
}
```

**案例：数据库连接未释放导致连接池泄漏**

```go
// 问题：请求处理中 panic，defer 没执行，连接未归还

// ✅ 正确做法：recover 保护
func handle(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if err := recover(); err != nil {
            log.Printf("panic recovered: %v", err)
            http.Error(w, "internal error", 500)
        }
    }()
    // 业务逻辑，可能panic
}
```

### 高频追问

**Q：SIGTERM 和 SIGINT 的区别？**
> SIGINT（Ctrl+C）是用户主动中断，SIGTERM 是 K8s/系统发送的终止信号。两者处理逻辑相同：优雅关闭。永远不要忽略 SIGTERM。

**Q：服务有长连接（WebSocket/GRPC stream）怎么办？**
> 长连接不能简单用 WaitGroup，要维护一个活跃连接集合：1）连接建立时注册，关闭时注销；2）Shutdown 时主动关闭所有活跃连接（close 所有 stream）；3）给连接关闭一个超时。

---

## 4. 并发任务编排

### 面试官考察意图

考察候选人对 Go 并发原语的掌握程度，以及能否根据业务场景选择合适的编排模式。这道题会延伸到 context 取消、errgroup 错误传播、semaphore 并发控制，是区分高级工程师的关键题目。

### 核心答案（30秒版）

Go 并发任务编排有三板斧：`errgroup`（并发执行+错误传播）、`semaphore`（限制并发数）、`pipeline`（多阶段流水线）。选择依据：任务是否需要等待全部完成、是否需要取消、是否需要限制并发数。

### 深度展开

#### errgroup：并发执行，任一失败全部取消

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx) // 任意一个err，ctx cancel
    results := make([]string, len(urls))

    for i, url := range urls {
        i, url := i, url // 捕获循环变量
        g.Go(func() error {
            resp, err := http.Get(url) // 如果ctx cancel，自动取消
            if err != nil {
                return fmt.Errorf("fetch %s: %w", url, err)
            }
            body, _ := io.ReadAll(resp.Body)
            results[i] = string(body)
            return nil
        })
    }

    // 等待所有 goroutine 完成，或任一出错
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

`errgroup.WithContext` 核心：当某个 goroutine 返回非 nil 错误，其他 goroutine 会通过 `ctx cancel` 被通知取消。

#### semaphore：控制最大并发数

```go
import "golang.org/x/sync/semaphore"

// 限制同时只有3个goroutine执行
var sem = semaphore.NewWeighted(3)

func limitedTask(ctx context.Context) error {
    if err := sem.Acquire(ctx, 1); err != nil {
        return err // ctx取消或超时
    }
    defer sem.Release(1)

    return doWork()
}

// 并发执行100个任务，最多3个同时运行
func main() {
    ctx := context.Background()
    for i := 0; i < 100; i++ {
        go func(i int) {
            if err := limitedTask(ctx); err != nil {
                log.Printf("任务%d失败: %v", i, err)
            }
        }(i)
    }
    time.Sleep(10 * time.Second)
}
```

#### pipeline 模式：多阶段流水线

```go
// 阶段1：生成数据
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// 阶段2：平方
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// 阶段3：过滤（保留偶数）
func filter(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
        close(out)
    }()
    return out
}

// 使用
func main() {
    // 生成 → 平方 → 过滤
    in := generate(1, 2, 3, 4, 5)
    sq := square(in)
    even := filter(sq)

    for n := range even {
        fmt.Println(n) // 4, 16
    }
}
```

Pipeline 的优势：各阶段解耦、可以并行执行（square 的多个 worker 可以并发读同一个 channel）、天然背压（channel满了发送方阻塞）。

#### Fan-out/Fan-in 模式

```go
// Fan-out：多个worker从同一个channel读
// Fan-in：多个channel合并成一个

func fanOutIn(nums <-chan int, workers int) <-chan int {
    in := make(chan int, 100)

    // Fan-out: 启动N个worker
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for n := range in {
                in <- doWork(n)
            }
        }()
    }

    // Fan-in: 合并结果
    out := make(chan int)
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

#### 生产级 Worker Pool 实现

```go
func WorkerPool(ctx context.Context, jobs <-chan Job, workers int) error {
    g, ctx := errgroup.WithContext(ctx)

    for i := 0; i < workers; i++ {
        i := i // 避免闭包捕获
        g.Go(func() error {
            for {
                select {
                case job, ok := <-jobs:
                    if !ok {
                        return nil
                    }
                    if err := job.Process(); err != nil {
                        return fmt.Errorf("worker%d job%d: %w", i, job.ID, err)
                    }
                case <-ctx.Done():
                    return ctx.Err()
                }
            }
        })
    }

    return g.Wait()
}
```

### 高频追问

**Q：errgroup 里的 goroutine panic 了怎么办？**
> 默认 errgroup 不会捕获 panic，会导致整个程序崩溃。生产环境建议用 `errgroup.Group.Go` 时用 `defer recover()` 包一层，防止 panic 扩散。如果需要捕获 panic，可以用 `golang.org/x/sync/errgroup` 的变体或自己封装。

**Q：context 取消和 semaphore 限流有什么区别？**
> context 取消是**协作式取消**：调用 `cancel()` 后，goroutine 需要主动检查 `ctx.Done()` 并退出。semaphore 限流是**阻塞式限流**：Acquire 会阻塞直到拿到信号量。两者可以组合：`semaphore.NewWeighted(3).Acquire(ctx, 1)`，当 context 取消时，Acquire 会立即返回错误。

**Q：pipeline 中某个阶段出错了，之前阶段的数据怎么处理？**
> 用 errgroup.WithContext：任意阶段出错，ctx cancel，所有阶段停止处理。对于已经 buffer 在 channel 里的数据，如果 channel 缓冲区满，发送方会阻塞。可以给 channel 设置适当 buffer 大小，或用 context 控制丢弃。

---

## 延伸阅读

- [ants - Goroutine pool for Go](https://github.com/panjf2000/ants)
- [golang.org/x/sync - errgroup, semaphore](https://pkg.go.dev/golang.org/x/sync)
- [Go graceful shutdown 官方博客](https://pkg.go.dev/net/http#Server.Shutdown)
- [Go Concurrency Patterns: Pipeline](https://go.dev/blog/pipelines)
