# Blocking Profile 阻塞分析：定位 Goroutine 锁竞争瓶颈

> 考察频率：★★★☆☆  优先级：P1（Senior 加分）
> 关键词：block profile、mutex profile、goroutine contention、pcontended、sync.Mutex

---

## 面试官考察意图

这道题考察候选人对 **Go 并发性能调优的深度**。初级只会看 CPU profile 找热点函数；高级能利用 **blocking profile 精准定位 goroutine 因为等待锁或 channel 而产生的阻塞**，并据此优化同步策略。在生产环境中，CPU profile 显示"瓶颈在某处"但无法解释"为什么慢"，而 blocking profile 可以精确到哪个锁导致了延迟。

---

## 核心答案（30 秒版）

Go 的 **blocking profile** 记录的是 **goroutine 被阻塞的原因和位置**：

| Profile 类型 | 捕获什么 | 适用场景 |
|-------------|---------|---------|
| **Block Profile** | Goroutine 因等待锁/channel 而阻塞的时间 | 并发瓶颈排查 |
| **Mutex Profile** | 锁竞争次数与持有时间 | 锁粒度和公平性分析 |

启用方式：`runtime.SetBlockProfileRate(n)` —— n 为采样的概率（n > 0 时生效）。

**核心用法：**
```bash
go tool pprof -sample_index block profile.pb
```
输出会显示 goroutine 等待锁的平均时间和总阻塞时间，帮助判断是锁粒度太粗还是锁争用太激烈。

---

## 深度展开

### 1. Block Profile 的工作原理

```go
// src/runtime/profile.go
func SetBlockProfileRate(rate int) {
    // rate == 0   → 关闭
    // rate == -1  → 所有事件都采样
    // rate > 0    → 每次阻塞事件以 rate/n 概率采样
}
```

**内部机制：** Go runtime 在每个 sync.Mutex.Lock() 调用前插入一个检查点。当 goroutine 在该锁上等待超过阈值（约 1μs），会记录：
- goroutine ID
- 堆栈追踪
- 阻塞时长
- 阻塞原因（mutex、channel、timer 等）

### 2. Block Profile 的三种阻塞分类

| 分类 | 常量名 | 触发场景 |
|------|--------|---------|
| `sync_mutex` | `syncMutexProfile` | 等待 `sync.Mutex` 锁 |
| `sync_channel` | `syncChannelProfile` | Channel send/receive |
| `sync_timer` | `syncTimerProfile` | 等待 Timer/AfterFunc |

### 3. 实战：从 Block Profile 到优化决策

#### 步骤 1：生成 Profile

```go
import _ "net/http/pprof"

// 在 HTTP handler 或其他入口挂载
go func() {
    http.ListenAndServe(":6060", nil) // 暴露 pprof 端点
}()

// 访问 http://localhost:6060/debug/pprof/block?debug=2
```

#### 步骤 2：采样与分析

```bash
# 采样率设为 1%（默认值足够）
# 生产环境可用 setBlockProfileRate(1)

# 抓取 profile
curl http://localhost:6060/debug/pprof/block?debug=2 > block.pb

# 分析
go tool pprof -http=:8081 block.pb
```

**典型输出：**

```
(pprof) top 10
Showing nodes accounting for 2.34s, 94.75% of 2.47s total
Dropped 39 nodes (cum <= 0.01s)
      flat  flat%   sum%        cum   cum%
   1.82s 73.68% 73.68%    1.82s 73.68%  sync.runtime_SemacquireMutex
   0.34s 13.76% 87.45%    0.34s 13.76%  sync.(*Mutex).Lock
   0.12s  4.86% 92.31%    0.46s 18.62%  main.(*Cache).Get
   0.05s  2.02% 94.33%    0.05s  2.02%  sync.runtime_procPin
```

**解读：**
- `sync.runtime_SemacquireMutex` 占 73.68% —— 大量 goroutine 在等锁
- `main.(*Cache).Get` 累计消耗 18.62% —— Cache.Get 内部用了重锁
- **优化方向**：考虑将全局 Cache 改为分片锁或无锁结构

#### 步骤 3：查看详细堆栈

```bash
(pprof) list Cache.Get
Total: 2.47s

ROUTINE ======================== main.(*Cache).Get in /app/cache.go
   0.12s    0.46s (flat, cum) 18.62% of Total
         .          .     42:func (c *Cache) Get(key string) []byte {
         .          .     43:    c.mu.RLock()
   0.12s    0.46s     44:    defer c.mu.RUnlock()
         .          .     45:    val, ok := c.store[key]
         .          .     46:    ...
         .          .     47:}
```

### 4. Mutex Profile：另一种视角

```go
// 启用 mutex profile（不同于 block profile）
runtime.SetMutexProfileFraction(1)

// 采样 1% 的锁获取事件
```

**两种 Profile 的区别：**

| | Block Profile | Mutex Profile |
|---|--------------|---------------|
| 关注点 | goroutine 花多少时间等待 | 多少次尝试获得锁 |
| 数据维度 | 阻塞时长 | 竞争次数 |
| 最佳用途 | "谁被堵得最久" | "哪些锁最容易打架" |

**典型 Mutex Profile 输出：**

```
(pprof) top -cum -nodecount 10
Showing nodes accounting for 1250, 98.03% of 1275 total
Dropped 19 nodes (cum <= 6.38)
      flat  flat%   sum%        cum   cum%
    820 64.31% 64.31%     1150 90.20%  main.processRequest
     200 15.69% 80.00%      250 19.61%  (*DB).Query
     100  7.84% 87.84%      100  7.84%  sync.(*RWMutex).RLock
      50  3.92% 91.76%       50  3.92%  sync.(*RWMutex).Unlock
      30  2.35% 94.12%       50  3.92%  sync.(*RWMutex).lockSlow
...
```

### 5. 实战案例：从 Profile 到修复

#### 问题场景：HTTP 服务高延迟

```go
// ❌ 问题代码：全局大锁保护小操作
var mu sync.Mutex
var cache = make(map[string][]byte)

func GetData(key string) []byte {
    mu.Lock()
    defer mu.Unlock()
    
    // 这里只读一个小操作却被大锁保护
    if val, ok := cache[key]; ok {
        return val
    }
    // 模拟 IO 读取文件（~1ms）
    time.Sleep(time.Millisecond)
    val := readFile(key)
    cache[key] = val
    return val
}
```

**使用 pprof 分析：**

```bash
# CPU profile 显示 GetData 是高频调用 ✓
# Block profile 显示大量等待在 mu.Lock() ✓
# → 结论：锁粒度过粗，IO 时间在临界区内
```

**修复方案：**

```go
// ✅ 方案1：读写锁 + 缩小临界区
var mu sync.RWMutex
var cache = make(map[string][]byte)

func GetData(key string) []byte {
    // 先读
    mu.RLock()
    if val, ok := cache[key]; ok {
        mu.RUnlock()
        return val
    }
    mu.RUnlock()
    
    // 写操作（独占）
    mu.Lock()
    defer mu.Unlock()
    
    // Double-check after acquiring write lock
    if val, ok := cache[key]; ok {
        return val
    }
    
    time.Sleep(time.Millisecond)  // IO 不在读锁范围内
    val := readFile(key)
    cache[key] = val
    return val
}

// ✅ 方案2：分片锁（更高并发）
const numShards = 32
type Shard struct {
    mu   sync.RWMutex
    data map[string][]byte
}
type ShardedCache struct {
    shards [numShards]Shard
}

func (sc *ShardedCache) GetData(key string) []byte {
    shard := &sc.shards[hash(key)%numShards]
    
    shard.mu.RLock()
    val, ok := shard.data[key]
    shard.mu.RUnlock()
    if ok {
        return val
    }
    
    shard.mu.Lock()
    defer shard.mu.Unlock()
    
    if val, ok := shard.data[key]; ok {
        return val
    }
    
    val = readFile(key)
    shard.data[key] = val
    return val
}
```

### 6. 常见误用与坑

#### 误区 1：Block Profile 不是 CPU Profile

```
Block Profile 告诉你："goroutine 在等什么"
CPU Profile 告诉你："goroutine 在做什么"

两者互补但不等价！
```

#### 误区 2：默认采样率可能漏掉低频阻塞

```go
// 默认 rate = 100，意味着每 100 次阻塞事件采样 1 次
// 如果阻塞很频繁但时间短，可能采不到
runtime.SetBlockProfileRate(10)  // 提高采样率到 10%
```

#### 误区 3：忽略 channel 操作的阻塞

```go
// ch <- x 也会产生 block profile 记录
// 很多人只注意 mutex，忽略了 channel 也是主要阻塞源

// 比如 worker pool 中 result chan 满了会导致 worker 阻塞
for _, job := range jobs {
    results <- process(job)  // ← results 满了就阻塞在这里
}
```

#### 误区 4：把阻塞时间等同于计算时间

```go
// ⚠️ 关键认知：
// Block profile 的累积时间是 "等待时间"，不是"实际花费时间"
// 100 个请求各自等了 50ms 锁 = 总阻塞 5s
// 但这 5s 中 CPU 没做任何计算！
// → 优化目标是减少这 5s 的无效等待
```

### 7. 结合其他 Profile 的综合调优

```go
// 同时采集多种 profile
import (
    "net/http/pprof"
    _ "net/http/pprof"
)

func main() {
    go func() {
        http.ListenAndServe(":6060", nil)
    }()
    
    // 常用组合：
    // :6060/debug/pprof/profile   — CPU profile（30s 默认）
    // :6060/debug/pprof/heap      — Memory heap profile
    // :6060/debug/pprof/goroutine — Goroutine stack traces
    // :6060/debug/pprof/block     — Block profile
    // :6060/debug/pprof/mutex     — Mutex profile
    // :6060/debug/pprof/trace     — Execution trace
    
    // 用 tracer 做细粒度分析
    f, _ := os.Create("trace.out")
    defer f.Close()
    trace.Start(f)
    defer trace.Stop()
    
    runBenchmarks()
}
```

---

## 🗣️ 面试话术

**一句话记住**：CPU profile 告诉你"在哪花时间"，block profile 告诉你"在等什么浪费时间"。两者配合才能完整诊断并发性能问题。

---

## 🔗 延伸阅读

- [Go Blog: Introducing the Go Race Detector](https://go.dev/blog/introducing-the-go-race-detector)（包含 profiling 概述）
- [Go pprof 文档](https://pkg.go.dev/net/http/pprof)
- [Uber Go Style Guide: Profiling](https://github.com/uber-go/guide/blob/master/style.md#performance)
