[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 并发编程](../README.md)

---

# atomic 与无锁编程：CAS、atomicValue、lock-free

## 面试官考察意图

考察候选人对 Go 原子操作和底层内存模型的掌握深度。
初级只能说出"atomic 用于计数"，高级要能讲清楚 **CAS 原理、 ABA 问题、atomic.Value 泛型读写的实现、以及如何在生产中用无锁结构避免 Mutex 竞争**，并结合 CPU 缓存架构（MESI）和伪共享给出性能分析。

---

## 核心答案（30 秒版）

Go sync/atomic 提供了六类操作：

| 类别 | 操作 | 说明 |
|------|------|------|
| **Add** | `AddInt64/AddUint64` 等 | 原子加减，返回**新值** |
| **Compare-And-Swap** | `CompareAndSwapInt64` | CAS，成功返回 true |
| **Load** | `LoadInt64/LoadPointer` 等 | 原子读取 |
| **Store** | `StoreInt64/StorePointer` 等 | 原子写入 |
| **Swap** | `SwapInt64` 等 | 原子交换，返回**旧值** |
| **Value** | `Load/Store` | 泛型读写接口 |

核心语义：**acquire/release** — 保证可见性但不保证顺序（除非配合内存屏障）。

---

## 深度展开

### 1. CAS：一切原子操作的基础

#### 什么是 CAS

CAS（Compare-And-Swap）是 CPU 提供的指令：
```
CAS(addr, old, new):
    if *addr == old:
        *addr = new
        return true
    else:
        return false
```
一条 CPU 指令完成，天然原子性，不存在竞态。

#### Go 中的 CAS

```go
// 伪代码实现自旋锁
func (m *Mutex) Lock() {
    for {
        if atomic.CompareAndSwapInt32(&m.state, 0, 1) {
            return // 加锁成功
        }
        // 没抢到，空转（自旋）
        runtime.Gosched()
    }
}
```

#### CAS 的三大问题

**问题 1：ABA 问题**

```
线程A: 读取 x = 100
线程B: CAS(x, 100, 200) → 成功，x = 200
线程B: CAS(x, 200, 100) → 成功，x = 100
线程A: CAS(x, 100, 300) → 成功！（但实际上 x 已经被改过了）
```

**ABA 问题的危害场景：** 栈的 push/pop、链表操作、垃圾回收标记。

**解决方案：**

```go
// 方案1：版本号（最常用）
type TaggedPtr struct {
    ptr uint64
    ver uint64  // 版本号，每次修改+1
}

// 方案2：Go 的 sync/atomic 不直接支持，用第三方库
// go.uber.org/atomic 的 Int64 封装了版本号

// 方案3：用 Double CAS（DCAS）
// Go 不支持 DCAS，勉强用 Mutex
```

**问题 2：自旋开销**

```go
// 错误：无限自旋
for atomic.CompareAndSwapInt64(&addr, old, new) == false {
    // CPU 全速空转，极度浪费
}

// 正确：有限自旋 + backoff
var backoff int
for !atomic.CompareAndSwapInt64(&addr, old, new) {
    for i := 0; i < 1<<backoff; i++ {
        runtime.Gosched()
    }
    if backoff < 10 {
        backoff++
    }
}
```

**问题 3：只适合单个变量**

CAS 只能保证单个变量的原子性，多变量需要 Mutex 或专门的 lock-free 数据结构。

### 2. atomic.Add 的陷阱：返回值是**新值**不是旧值

```go
var counter int64

// 错误理解：counter == 1
atomic.AddInt64(&counter, 1)
fmt.Println(atomic.LoadInt64(&counter)) // 打印 1，正确

// 但如果你想：
// "只有 counter 当前是 0 时才加 1"
atomic.AddInt64(&counter, 1) // ❌ 这不是 CAS，是无条件 Add
// 应该用：
if atomic.CompareAndSwapInt64(&counter, 0, 1) {
    // 只有 counter==0 时才改成 1
}
```

### 3. atomic.Value：泛型只读存储

`atomic.Value` 是 Go 1.17+ 提供的泛型只读原子存储，用于**读多写少**的无锁场景。

```go
type Config struct {
    Timeout time.Duration
    MaxConn int
}

var config atomic.Value

// 初始化
config.Store(&Config{Timeout: 5 * time.Second, MaxConn: 100})

// 读取（永远安全，不需要锁）
func GetConfig() *Config {
    return config.Load().(*Config)
}

// 更新（写操作需要额外同步）
var mu sync.Mutex
func UpdateConfig(newCfg *Config) {
    mu.Lock()
    config.Store(newCfg)
    mu.Unlock()
}
```

#### atomic.Value 内部实现

```go
// src/sync/atomic/value.go
type Value struct {
    v any  // interface{}
}

// 核心：
// - Store 写入时用 CAS 设置 v 指针
// - Load 用 atomic load 读取 v 指针
// - 关键保证：写入对所有 reader 可见（通过 Go 的safe pointer）
```

**atomic.Value 的限制：**

```go
// 坑1：不能存指针指向的内容被修改的对象
type Node struct{ val int }
var v atomic.Value
n := &Node{val: 1}
v.Store(n)
n.val = 100  // ❌ data race！其他 reader 可能看到部分构造
v.Store(&Node{val: 100}) // ✅ 新对象，OK

// 坑2：第一次 Store 之前 Load 是未定义行为
var v atomic.Value
v.Load() // ❌ 可能 panic

// 解决：用 sync.Once 初始化
var v atomic.Value
var once sync.Once
func GetV() *Node {
    once.Do(func() { v.Store(&Node{}) })
    return v.Load().(*Node)
}
```

### 4. 实战：lock-free 计数器

#### 场景：高频更新、低频读取（日志计数器、Metrics）

```go
// ❌ 低效：Mutex 方案
type Counter struct {
    mu sync.Mutex
    n  int64
}
func (c *Counter) Inc() {
    c.mu.Lock()
    c.n++
    c.mu.Unlock()
}

// ✅ 高效：atomic 方案
type Counter struct {
    n atomic.Int64
}
func (c *Counter) Inc() {
    c.n.Add(1) // 一次 CPU 指令，无锁
}
func (c *Counter) Value() int64 {
    return c.n.Load()
}
```

#### 性能对比（单 goroutine 1000万次操作）

| 方案 | 耗时 | 每操作 |
|------|------|--------|
| Mutex | ~800ms | ~80ns |
| atomic.Add | ~120ms | ~12ns |
| 提升 | 6.7x | — |

### 5. 实战：lock-free 队列（Michael-Scott Queue）

```go
type node struct {
    val  any
    next atomic.Pointer[node]
}

type Queue struct {
    head atomic.Pointer[node]
    tail atomic.Pointer[node]
}

func NewQueue() *Queue {
    n := &node{}
    return &Queue{head: atomic.Pointer[node]{}, tail: atomic.Pointer[node]{}}
}

// Enqueue：无锁入队
func (q *Queue) Enqueue(v any) {
    n := &node{val: v}
    for {
        tail := q.tail.Load()
        next := tail.next.Load()
        if tail == q.tail.Load() { // 再次检查
            if next == nil {
                // 尝试追加
                if tail.next.CompareAndSwap(next, n) {
                    // 尝试更新 tail
                    q.tail.CompareAndSwap(tail, n)
                    return
                }
            } else {
                // tail 落后，移动 tail
                q.tail.CompareAndSwap(tail, next)
            }
        }
    }
}

// Dequeue：无锁出队
func (q *Queue) Dequeue() (any, bool) {
    for {
        head := q.head.Load()
        tail := q.tail.Load()
        next := head.next.Load()
        if head == q.head.Load() {
            if head == tail {
                if next == nil {
                    return nil, false // 空队列
                }
                // tail 落后
                q.tail.CompareAndSwap(tail, next)
            } else {
                v := next.val
                if q.head.CompareAndSwap(head, next) {
                    return v, true
                }
            }
        }
    }
}
```

### 6. 伪共享与缓存行对齐

CPU 缓存以**缓存行**（通常 64 字节）为单位。多个变量在同一缓存行时，一个核修改会导致其他核的缓存行失效。

```go
// ❌ 伪共享：Counter1 和 Counter2 在同一缓存行
type BadCounters struct {
    C1 int64  // 偏移 0
    C2 int64  // 偏移 8 —— 同一缓存行！
}

// G1 修改 C1 → G2 的 C2 缓存行失效
// G2 修改 C2 → G1 的 C1 缓存行失效
// 即使两个字段完全不相关，也会被迫同步

// ✅ 解决方案：缓存行对齐
type PaddedCounter struct {
    C1 int64
    _  [8]int64  // 填满当前缓存行，后面 C2 落到下一行
    C2 int64
}

// 或者用 go.uber.org/atomic 的封装
```

### 7. CPU 缓存架构与原子操作性能

```
CPU Core 0  ←→  L1 Cache (32KB)  ←→  L2 Cache (256KB)  ←→  L3 Cache (共享)  ←→  主存
                                                                                     |
核心操作延迟：                                                                       |
  L1: ~1ns                                                                           |
  L2: ~4ns                                                                           |
  L3: ~15ns                                                                          |
  主存: ~100ns                                                                       |
  atomic.Add: ~15ns（在 L3 缓存内）                                                  |
  Mutex Lock+Unlock: ~50-200ns（含内核切换）                                           |
```

---

## 高频追问

**Q：atomic 的 Load/Store 能保证什么？**

`atomic.Load` 是 **acquire 语义** — 之后的读写不能重排到它之前。
`atomic.Store` 是 **release 语义** — 之前的读写不能重排到它之后。
这保证了在 synchronized-with 关系建立时，writer 的写入对 reader 可见。

**Q：atomic.AddInt64 返回的是新值还是旧值？**

返回**新值**（修改后的值）。如果要判断是否发生变化，必须用 CAS。

**Q：Go 的 atomic 能否保证 64 位对齐？**

32 位系统上，64 位变量的 atomic 操作（LoadInt64、StoreInt64）要求 8 字节对齐。Go 编译器保证了 struct 字段按声明顺序 8 字节对齐，但如果用 `unsafe.Pointer` 自行转换，需要注意对齐问题。

**Q：什么场景下 atomic 比 Mutex 更好？**

- 计数器、flags、配置值等简单变量的高频更新
- 读多写少（读用 atomic，写用 Mutex 或 sync.Once）
- 对延迟极其敏感（Mutex 的 park/unpark 切换 ~200ns）
- 不需要复杂的复合操作（需要的话 Mutex 更合适）

**Q：atomic 能替代 Mutex 吗？**

不能。atomic 只保证单个变量的操作原子性，无法保护由多个变量组成的复合操作（检查-然后-行动）。

---

## 延伸阅读

- [Go atomic 包官方文档](https://pkg.go.dev/sync/atomic)（官方实现）
- [Google Talks: Understanding the Real Cost of Lock Contention](https://www.youtube.com/watch?v=W8tN_q3kJPA)
- [DDD with Go: Lock-free Data Structures](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part3.html)
- [Uber: go.uber.org/atomic](https://github.com/uber-go/atomic)（更易用的 atomic 封装）
- [MESI 缓存一致性协议](https://en.wikipedia.org/wiki/MESI_protocol)

---

**[← 上一篇：sync 原语](./02-sync.md)** · **[下一篇：并发模式 →](./04-patterns.md)**
