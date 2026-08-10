[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 并发编程](../README.md)

---

# sync 原语：Mutex / RWMutex / Once / Pool

## 面试官考察意图

考察候选人对 Go 同步原语的掌握深度。
初级只能说出"Mutex 加锁解锁"，高级要能讲清楚 **Mutex 的自旋机制、饥饿模式与公平性保证、RWMutex 读写分离的实现代价、以及 sync.Pool 的 GC 清空问题**，并结合生产中的死锁、活锁、性能问题给出分析。

---

## 核心答案（30 秒版）

Go sync 包核心原语：

| 原语 | 用途 | 关键特性 |
|------|------|----------|
| `Mutex` | 保护临界区 | 自旋等待、饥饿模式、不可重入 |
| `RWMutex` | 读多写少场景 | 写锁优先、写锁阻塞所有读者 |
| `sync.Once` | 单次执行 | 双重检查、只执行一次保证 |
| `sync.Pool` | 高频对象复用 | **GC 时清空**、无保证、不适合同步 |

---

## 深度展开

### 1. Mutex 实现：自旋 + 饥饿 + 公平

```go
// src/sync/mutex.go（核心逻辑）
type Mutex struct {
    state int32  // 低位：锁状态 | 第1位：饥饿 | 第2位：唤醒
    sema  uint32 // 信号量
}

// 锁状态位
const (
    mutexLocked = 1 << iota  // 1: 已加锁
    mutexWoken               // 2: 唤醒标志
    mutexStarving             // 4: 饥饿模式
    mutexWaiterShift = iota   // 3: 等待者计数偏移
)
```

**加锁流程（Fast Path）：**

```
Lock()
    │
    ├─ CAS(state, 0, 1)     ← 尝试直接加锁（无竞争）
    │     成功 → 返回        ← 极快，~25ns
    │
    └─ 失败（已被锁定）
           │
           ├─ 自旋 4 次（CPU 空转）
           │   （条件：GOMAXPROCS>1、处于多核、running P）
           │
           ├─ 尝试 CAS 加锁（可能被其他自旋释放）
           │     成功 → 获得锁
           │
           └─ 否则：进入等待队列
                  设置 mutexStarving（饥饿模式）
                  runtime.SemaVC(&m.sema) → park goroutine
```

**解锁流程：**

```go
func (m *Mutex) Unlock() {
    // Fast path: 直接释放锁
    if atomic.AddInt32(&m.state, -mutexLocked) >= 0 {
        return
    }
    // Slow path: 处理等待队列
    unlockSlow()
}
```

**饥饿模式 vs 正常模式：**

| 模式 | 触发条件 | 策略 | 优点 |
|------|----------|------|------|
| **正常模式** | 新来的等待者自旋抢锁 | 自旋 ~4 次后入队 | 性能最优，LRO（Latest Release First） |
| **饥饿模式** | 等待超过 1ms | 锁直接给等待队列队头 | 防止 goroutine 饿死（饿死问题：长等待队列导致新请求永远无法竞争过队头） |

```
饥饿模式解决的是"尾部队尾goroutine永远等不到锁"的问题
（队头goroutine释放锁时，新来的goroutine可能通过自旋抢走）
```

### 2. RWMutex：读多写少的锁

```go
type RWMutex struct {
    w           Mutex
    writerSem   uint32
    readerSem   uint32
    readerCount atomic.Int32  // 当前读者数量
    readerWait  atomic.Int32  // 写锁等待时，已经获取读锁的读者数量
}
```

**加锁逻辑：**

```
写锁 Lock():
    w.Lock()              ← 互斥写锁本身
    readerCount.Add(-rwmutexMaxReaders)  ← 把 readerCount 变成负数
                                    ← 所有后续 reader 都会被挡在锁外面

写锁 Unlock():
    readerCount.Add(rwmutexMaxReaders)  ← 恢复 readerCount
    w.Unlock()
```

```
读锁 Lock():
    if atomic.AddInt32(&readerCount, 1) < 0:
        // readerCount 是负数（有写者在等）
        // park 当前读者，等 writer 信号
        runtime.SemaVC(&readerSem)

读锁 Unlock():
    if atomic.AddInt32(&readerCount, -1) == -rwmutexMaxReaders:
        // 我是最后一个读者，唤醒等待的写者
        runtime.SemaVC(&writerSem)
```

**RWMutex 的坑：**

```go
// 坑1：读写锁不支持重入！
var mu sync.RWMutex
func read() {
    mu.RLock()
    defer mu.RUnlock()
    write() // ❌ 如果 write() 也需要写锁 → 死锁！
}

// 坑2：写锁饥饿
// 高并发读场景下，写者可能永远抢不到锁（不断有新读者插入队头）
// 解决方案：降低 reader 持有锁的时间，或换用更细粒度的锁
```

### 3. sync.Once：单次执行

```go
type Once struct {
    done atomic.Int32  // 0=未执行，1=已执行
    m    Mutex
}

func (o *Once) Do(f func()) {
    // Fast path：已经执行过
    if o.done.Load() == 1 {
        return
    }
    // Slow path：加锁，双重检查
    o.m.Lock()
    defer o.m.Unlock()
    if o.done.Load() == 0 {
        f()
        o.done.Store(1)
    }
}
```

**为什么需要双重检查？**

```go
// 如果没有 done.Load() 的第一次检查：
// 两个 goroutine 同时调用 Do()：
//   G1: Lock() → f() → Store(1) → Unlock()
//   G2: Lock() → (G1 Unlock 后才到这里)
//       再次执行 f() ！← 违背单次执行语义

// 第一次 Load() 检查避免了已执行后仍要竞争锁
```

#### Go 1.21 新增：`OnceFunc / OnceValue / OnceValues`

Go 1.21 在 `sync.Once` 基础上新增了三个便捷工厂函数，解决了 `Once.Do()` 无法传参和获取返回值的问题：

```go
// OnceFunc：返回可多次调用、只执行一次的函数
func OnceFunc(f func()) func() {
    once := new(Once)
    return func() {
        once.Do(f)
    }
}

// OnceValue：返回可多次调用、只执行一次并缓存返回值的函数
func OnceValue[T any](f func() T) func() T {
    once := new(Once)
    var value T
    var onceVal atomic.Value
    onceVal.Store(&value)
    return func() T {
        if once.done.Load() == 0 {
            once.Do(func() {
                onceVal.Store(f())
            })
        }
        return *onceVal.Load().(*T)
    }
}

// OnceValues：返回可多次调用、只执行一次并缓存多个返回值的函数
func OnceValues[T1, T2 any](f func() (T1, T2)) func() (T1, T2) {
    once := new(Once)
    var onceVals atomic.Value
    return func() (T1, T2) {
        if once.done.Load() == 0 {
            once.Do(func() {
                onceVals.Store(f())
            })
        }
        v := onceVals.Load().([2]any)
        return v[0].(T1), v[1].(T2)
    }
}
```

**使用示例：**

```go
// 传统写法（繁琐，需要封装）
var doInit func()
once := sync.Once{}
doInit = func() { initConfig(); doInit = nil }

// Go 1.21+ 简洁写法
getConfig := sync.OnceFunc(initConfig)
getConfig() // 第一次执行 initConfig，之后直接返回

// 获取返回值
getDB := sync.OnceValue(func() *sql.DB {
    return connectDB()
})
db := getDB() // 返回缓存的数据库连接

// 获取多返回值
getUser := sync.OnceValues(func() (string, error) {
    return fetchUser()
})
name, err := getUser() // 第一次调用 fetchUser，之后返回缓存值
```

**面试追问：**

> Q：`sync.Once` vs `OnceFunc`，选哪个？

| 场景 | 推荐 |
|------|------|
| 不需要返回值，只需要执行一次副作用 | `sync.Once.Do()` |
| 需要缓存单次执行结果（泛型）| `sync.OnceValue` |
| 需要缓存多返回值 | `sync.OnceValues` |
| 不想自己管理 Once 实例 | `sync.OnceFunc` |

`OnceFunc/OnceValue/OnceValues` 本质是对 `sync.Once` 的封装，线程安全与 `Once.Do()` 一致。

### 4. sync.Pool：对象复用池

```go
// 生产级使用示例：减少 JSON 解析的内存分配
var jsonPool = sync.Pool{
    New: func() interface{} {
        return &json.Decoder{}
    },
}

func parseJSON(r io.Reader) (*json.Decoder, error) {
    dec := jsonPool.Get().(*json.Decoder)
    dec.Reset(r)
    defer func() {
        dec.Reset(nil)
        jsonPool.Put(dec)
    }()
    return dec, nil
}
```

**sync.Pool 的致命问题：GC 时会清空**

```go
// 官方注释明确说明：
// "Pool's implementation evicts all objects when the garbage collector runs"

// 真实测试：
runtime.GC()
pool.Put(make([]byte, 1024))
val := pool.Get() // nil ← 对象已被清空！

// 适用场景：高频临时分配（JSON buffer、string builder）
// 不适用场景：需要长期持有的缓存、连接池
```

**sync.Pool 内部结构：**

```
sync.Pool {
    local   []poolLocal   // per-P 本地池（减少竞争）
    global  *poolChain    // 全局回收池
}
```

```
Get() 流程：
1. 先从当前 P 的 poolLocal 取
2. 空 → 从其他 P 偷（Steal）
3. 空 → 从 global 取
4. 空 → 调用 New()

Put() 流程：
1. 放入当前 P 的本地池
2. 本地满 → 移入 global
```

### 5. 生产问题与排查

**问题 1：Mutex 不可重入导致死锁**

```go
// 错误示例：
func (s *Store) Update(key string, val int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.write(key, val)  // 内部又调用 Update → 死锁
}

func (s *Store) write(key string, val int) {
    s.mu.Lock()  // ❌ 已持有锁，再次加锁 → 永久阻塞
    defer s.mu.Unlock()
    // ...
}

// 正确做法：拆分成锁内/锁外两层
func (s *Store) Update(key string, val int) {
    s.mu.Lock()
    s.data[key] = val
    s.mu.Unlock()  // 不在内部调用带锁方法
}
```

**问题 2：RWMutex 读者饥饿**

```go
// 高并发读、写操作少的场景 → 写者可能永远等不到
// 实测：1000 个并发 reader，每 10s 有一个 writer
// writer 可能 10 分钟都无法获得写锁

// 解决方案：
// 1. 降低读锁持有时间（不要在锁内做 IO）
// 2. 使用 sync.Map（适用于读多写少、无复合操作）
// 3. 考虑分段锁（分片 map）
```

**问题 3：sync.Pool 误用导致数据不一致**

```go
// 错误：把 Pool 对象借给其他 goroutine
buf := jsonPool.Get().(*bytes.Buffer)
go func() {
    buf.Reset()
    buf.Write(...)
}()
// buf 可能被 Put() 回收，另一个 Get() 拿到的是部分写入的数据

// 正确：每次使用完必须 Put()
```

---

### 5. sync.Map：并发安全 Map（Go 1.21 新增方法）

`sync.Map` 是专为**读多写少**场景优化的并发安全 Map，核心思想是 `read` 和 `dirty` 双字典设计，实现无锁读、有锁写。

```go
type Map struct {
    read atomic.Value // readOnly（只读快照）
    dirty map[any]any // 实际数据（有锁）
    misses int        // read 未命中计数
}

type readOnly struct {
    m       map[any]entry
    amended bool // dirty 与 read 是否一致（true=有额外 key）
}
```

**Load / Store / Delete（基础操作）：**

```go
m := sync.Map{}

// 读取（无锁，极快）
v, ok := m.Load("key")

// 写入（有锁）
m.Store("key", "value")

// 删除（有锁）
m.Delete("key")
```

#### Go 1.21 新增方法（高频考点）

Go 1.21 为 `sync.Map` 新增了 6 个方法，解决了旧 API 功能不足的问题：

```go
// Clear：原子清空所有数据（Go 1.21+）
m.Clear() // 等价于遍历删除，但更高效

// CompareAndSwap：CAS 替换，仅当 key 存在且值匹配时才替换（Go 1.21+）
ok := m.CompareAndSwap("key", "old", "new")

// CompareAndDelete：CAS 删除，仅当 key 存在且值匹配时才删除（Go 1.21+）
ok := m.CompareAndDelete("key", "old")

// LoadAndDelete：原子读取并删除，返回旧值（Go 1.21+）
old, loaded := m.LoadAndDelete("key")

// Swap：原子交换，返回旧值（Go 1.21+）
old, loaded := m.Swap("key", "new")

// LoadOrStore：读取或插入（Go 1.21+ 更完善）
actual, loaded := m.LoadOrStore("key", "default")
```

**使用场景对比：**

| 方法 | 场景 | 示例 |
|------|------|------|
| `Load` | 纯读 | 缓存查找 |
| `Store` | 写 | 动态更新配置 |
| `LoadOrStore` | 先读后写 | 懒加载缓存 |
| `Swap` | 覆盖写入，需要旧值 | 计数器重置 |
| `CompareAndSwap` | 乐观锁更新 | 并发安全的计数器 |
| `CompareAndDelete` | 条件删除 | 删除特定版本数据 |
| `LoadAndDelete` | 读取并移除 | 消费后删除 |
| `Clear` | 重置 | 全量刷新 |

**生产级使用示例：**

```go
// 场景：配置中心，动态更新，热加载
type ConfigMap struct {
    sync.Map
}

func (c *ConfigMap) Update(k, v string) {
    c.Store(k, v)
}

func (c *ConfigMap) Get(k string) string {
    if v, ok := c.Load(k); ok {
        return v.(string)
    }
    return ""
}

// CompareAndSwap 实现乐观锁更新
func (c *ConfigMap) CompareUpdate(k, old, new string) bool {
    return c.CompareAndSwap(k, old, new)
}
```

**性能对比（高频追问）：**

| 场景 | sync.Map | RWMutex + map |
|------|----------|---------------|
| 纯读（>99%）| ✅ 无锁，极快 | ❌ 每次需加读锁 |
| 读多写少 | ✅ 优化路径 | ⚠️ 可接受 |
| 写多/复合操作 | ❌ 仍需锁，比普通 map 慢 | ✅ 完全可控 |
| 需要遍历 | ❌ Range 无保证一致性 | ✅ 一致性可保证 |

---

### 4. sync.WaitGroup：并发任务等待

#### 4.1 核心用法

```go
var wg sync.WaitGroup

wg.Add(1)
go func() {
    defer wg.Done() // 必须 defer，确保即使 panic 也会调用
    // 任务逻辑
}()
wg.Wait() // 阻塞直到所有任务完成
```

**三个核心方法：**

| 方法 | 作用 |
|------|------|
| `Add(n)` | 增加计数（可正可负） |
| `Done()` | 减一（等价于 `Add(-1)`） |
| `Wait()` | 阻塞直到计数归零 |

#### 4.2 Go 1.25 新增：`WaitGroup.Go()`

Go 1.25 为 `sync.WaitGroup` 新增了 `Go()` 方法，简化"启动 goroutine + 自动 Done"的常见模式：

```go
// 老写法（需配对 Add/Done）
wg.Add(1)
go func() {
    defer wg.Done()
    // 任务
}()

// Go 1.25 新写法（一行搞定，自动 Add + defer Done）
wg.Go(func() {
    // 任务
})
```

**源码实现（伪代码）：**

```go
func (wg *WaitGroup) Go(f func()) {
    wg.Add(1)
    go func() {
        defer wg.Done()
        f()
    }()
}
```

**对比：传统写法 vs Go() 的常见错误：**

```go
// ❌ 常见错误 1：Add 漏掉
go func() { /* 任务 */ }() // 没 Add，Wait 永远不会结束
wg.Wait()

// ❌ 常见错误 2：Done 漏掉
wg.Add(1)
go func() {
    // 任务（可能 panic）
    // 没 defer wg.Done()，Wait 永远不会结束
}()

// ❌ 常见错误 3：Add/Done 不配对（多 goroutine 场景）
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        // 任务
    }()
}

// ✅ Go() 彻底避免这些问题
for i := 0; i < 10; i++ {
    wg.Go(func() {
        // 任务，自动 Add+defer Done
    })
}
```

**Go() 的优势：**

| 维度 | 传统写法 | `wg.Go()` |
|------|----------|------------|
| 代码量 | 3 行（Add + go + Done） | 1 行 |
| Add/Done 配对错误 | 容易漏 | 不可能 |
| panic 安全 | 需配合 defer | 内置 defer，安全 |
| 可读性 | 模板代码干扰 | 意图清晰 |

#### 4.3 常见坑点

**坑 1：WaitGroup 是一次性的**

```go
// ❌ 错误：Wait 之后重置
wg.Wait()
wg.Add(10) // panic: sync: negative WaitGroup counter

// ✅ 正确：需要新实例
wg := new(sync.WaitGroup)
wg.Add(1)
wg.Wait()
```

**坑 2：在 goroutine 内调用 Wait**

```go
// ✅ 正确：在主 goroutine Wait
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        // 任务
    }()
}
wg.Wait() // 主 goroutine 等待

// ❌ 危险：在每个 goroutine 内部等待（可能导致所有 goroutine 互相等待）
```

**坑 3：Add 负数**

```go
wg.Add(-1) // 合法，等价于 Done()
// 但如果计数已经为 0：panic: sync: negative WaitGroup counter
```

**坑 4：与 errgroup 的取舍**

```go
import "golang.org/x/sync/errgroup"

var eg errgroup.Group
for i := 0; i < 10; i++ {
    eg.Go(func() error {
        return doSomething()
    })
}
if err := eg.Wait(); err != nil {
    // 有错误处理
}

// WaitGroup 没有错误传递，如果需要错误收集，用 errgroup
```

#### 4.4 生产实战：Worker Pool

```go
func processJobs(ctx context.Context, jobs []Job) []Result {
    var wg sync.WaitGroup
    results := make([]Result, len(jobs))

    for i, job := range jobs {
        idx := i // 闭包变量捕获
        wg.Add(1)
        wg.Go(func() {
            defer wg.Done()
            select {
            case <-ctx.Done():
                return // context 取消则提前退出
            default:
                results[idx] = process(job)
            }
        })
    }

    wg.Wait()
    return results
}
```

---

## 高频追问

**Q：Mutex 自旋的意义是什么？**

自旋让 goroutine 在用户态"忙等"，避免了进入内核态（park）切换的成本（约 200-500ns）。如果锁很快释放，自旋等待比 park 更高效。条件是多核 + 有其他可运行 goroutine。

**Q：Go 1.17 之前的 Mutex 实现和现在有什么不同？**

Go 1.17 之前：饥饿模式触发阈值是 30 秒，且饥饿模式下不保证 FIFO。
Go 1.17+：饥饿模式阈值改为 1ms，且严格保证 FIFO（防止等待者饿死）。

**Q：sync.Map 和 RWMutex + map 比，哪个好？**

| 场景 | 推荐 |
|------|------|
| 读多写少（读>90%），无复合操作 | `sync.Map`（无锁读） |
| 写多，或需要原子复合操作（Read+Write） | RWMutex + map |

`sync.Map` 的代价：写操作（Store/Delete）仍然需要锁，且比普通 map 更慢。

**Q：WaitGroup 为什么不能中途 Reset？**

WaitGroup 一旦计数器归零就不能重置使用。设计上，WaitGroup 用于"等待一组任务完成"，是**一次性**工具。如果需要"循环等待"，应该用新的 WaitGroup 实例，或考虑 channel/errgroup。

**Q：Go 1.25 `wg.Go()` 和手动 `Add + goroutine + Done` 有什么区别？**

本质上没有区别，`Go()` 只是便捷封装。但手动模式容易犯错（Add 漏掉、Done 漏掉、panic 时没 defer），`Go()` 内置 defer 保证了 panic 安全，且代码更简洁。

**Q：WaitGroup 和 errgroup.Group 怎么选？**

- 需要错误传递和收集 → `errgroup.Group`
- 只关心"完成"，无需错误 → `WaitGroup`（或 Go 1.25 的 `wg.Go()`）
- 需要 context 取消传播 → `errgroup.WithContext`

---

## 延伸阅读

- [Go Mutex 源码解析](https://github.com/golang/go/blob/master/src/sync/mutex.go)（官方实现）
- [sync.WaitGroup 官方文档](https://pkg.go.dev/sync#WaitGroup)
- [Go 1.25 WaitGroup.Go() 博客文章](https://appliedgo.net/spotlight/go-1.25-waitgroup-go/)
- [Mutex vs RWMutex: When to use what](https://pkg.go.dev/sync#Mutex)（官方文档）
- [ArdanLabs: Synchronization Patterns in Go](https://www.ardanlabs.com/blog/2018/10/synchronization-primitives-in-go-part-1.html)
- [Go 1.17 Mutex 改进](https://groups.google.com/g/golang-dev/c/M-m5cZCm2kM)（官方讨论）

---

**[← 上一篇：Channel 底层原理](./01-channel.md)** · **[下一篇：atomic 与无锁 →](./03-atomic.md)**

---

## 附录：Go 1.24 sync.Mutex lock2 自旋优化（spinbit）

> **Q：Go 1.24 runtime lock2 自旋优化（spinbit）是什么？**

Go 1.24 引入了 runtime lock2 的自旋优化（spinbit），核心解决的是**高并发下 mutex 的性能悬崖**问题。

**问题背景：**

在 ChanContended 基准测试中，当 GOMAXPROCS 从 1 → 4 → 8 → 12 时，吞吐量每次都减半。在 GOMAXPROCS=20 时，20 个线程在 lock2 调用中总共消耗 27.74 秒 CPU 时间，其中 70% 消耗在 `procyield`（自旋）上。**所有等待线程都在自旋，导致严重的 CPU 浪费和性能退化。**

**核心优化（spinbit 机制）：**

原设计：所有等待线程同时自旋竞争 → CPU 严重浪费。

```
Go 1.24 之前的问题：
线程1 ──自旋──> [争抢 mutex] ──> 线程2 ──自旋──> [争抢 mutex] ──> ...
                    ↑ 20个线程同时自旋，CPU 争用严重
```

新设计：引入 `spinbit` 状态位，只有持 spinbit 的线程自旋，其他线程直接 park。

```
Go 1.24 spinbit 优化：
                              ┌── 持有 spinbit 的线程自旋（只有1个）
                              ↓
解锁 ──→ 检测 spinbit ──→ 存在 → 避免唤醒，直接让自旋线程 CAS 抢锁
                              ↓ 不存在
                              唤醒等待线程（sema）
```

**设计要点：**
- 扩展 mutex 状态字，加入 `spinning` 标志位
- 只有持有 spinning 位的线程才能自旋重试，其他等待者直接进入睡眠
- 解锁时发现有 spinning 线程存在 → 跳过 wakeup，直接让自旋线程抢锁
- futex（Linux）和 Xchg8 两套实现，Linux 用 futex 性能更好
- **向后兼容：导出 API 没有变化**，Go 1.24 默认开启

**性能收益（官方基准测试）：**

| 场景 | 优化效果 |
|------|---------|
| GOMAXPROCS=20，lock2 高并发 | 吞吐量提升最高可达 ~70%（减少无效自旋） |
| CPU 争用 | lock2 调用中的 CPU 消耗从 27.74s → 大幅下降 |

**面试加分点：**

1. **不只是 mutex**：runtime lock2 优化影响所有使用 `runtime.Lock` 的场景，包括 channel 的 mutex
2. **spinbit vs 无自旋**：不是简单的"有自旋 vs 无自旋"，而是"只有1个自旋 vs 所有线程都自旋"
3. **与 Mutex 自旋的关系**：runtime lock2 是底层运行时锁，与 sync.Mutex 的用户态锁自旋机制不同，但都服务于减少 park 切换

**延伸阅读：**
- [Proposal: Improve scalability of runtime.lock2 (spinbit)](https://github.com/golang/go/issues/68312)（Go 1.24 lock2 优化提案）
- [Go1.24 新特性：自旋互斥 lock2 优化，性能有一定提高！](https://blog.csdn.net/eddycjy/article/details/145272722)
