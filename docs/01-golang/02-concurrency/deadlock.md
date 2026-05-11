[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 并发编程](../README.md)

---

# Go 死锁：四条件、常见场景与生产级排查

> 考察频率：★★★★☆  优先级：P0（必考知识点）
> 关键词：死锁、Coffman 条件、channel 死锁、Mutex 死锁、活锁、go-deadlock、pprof

---

## 面试官考察意图

死锁是 Go 并发编程中的致命问题，考察候选人**对并发安全的深层理解**。
初级只能说出"死锁就是两个 goroutine 互相等待"，高级要能讲清楚 **死锁的四个必要条件（Coffman 条件）、Go 中最常见的死锁场景（channel 阻塞、Mutex 重入、select 悬停）、以及如何用 go-deadlock、pprof、runtime trace 工具在生产环境定位死锁**。

---

## 核心答案（30 秒版）

死锁的四个必要条件（Coffman 条件）：

| 条件 | 含义 | 破环方式 |
|------|------|----------|
| **互斥** | 资源每次只能被一个 goroutine 持有 | 无法破坏（锁的本质）|
| **持有并等待** | goroutine 持有资源同时等待其他资源 | 先拿全部再操作 |
| **不可抢占** | 资源不能被强制释放 | tryLock、设置超时 |
| **循环等待** | goroutine 循环等待链 | 按顺序加锁、层级化资源 |

Go 最常见的死锁场景：**无缓冲 channel 双向阻塞、sync.Mutex 不可重入、select 没有 default 走不出去**。

---

## 深度展开

### 1. 死锁的四个必要条件（Coffman 条件）

死锁必须同时满足以下四个条件，破环任一即可：

```go
// 条件1：互斥 — channel / Mutex 同一时刻只能被一个 goroutine 使用
ch := make(chan int)
mu := &sync.Mutex{}

// 条件2：持有并等待 — goroutine 持有锁的同时等待 channel
func holdAndWait() {
    mu.Lock()           // 持有锁
    ch <- 1            // 等待 channel（如果没人接收，永远阻塞）
    mu.Unlock()
}

// 条件3：不可抢占 — Go 的锁和 channel 都不支持强制释放
// 没有 "force unlock" 或 "cancel channel send" 的标准手段

// 条件4：循环等待 — goroutine A 等待 goroutine B，B 又在等待 A
func circularWait() {
    go func() {
        mu.Lock()
        <-ch    // 等待 goroutine B 给我发数据
        mu.Unlock()
    }()
    go func() {
        ch <- 1  // goroutine B 等待 goroutine A 释放锁
        mu.Lock()
        mu.Unlock()
    }()
}
```

### 2. Go 最常见的死锁场景

#### 2.1 无缓冲 channel 双向阻塞（最常见）

```go
// ❌ 死锁：无人接收
func deadlockNoReceiver() {
    ch := make(chan int)
    ch <- 42  // 永久阻塞
    <-ch      // 永远不会执行
}

// ❌ 死锁：无人发送
func deadlockNoSender() {
    ch := make(chan int)
    <-ch     // 永久阻塞
    <-ch     // 永远不会执行
}

// ❌ 死锁：相互等待
func deadlockMutualWait() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go func() {
        <-ch1         // 等 ch1 有数据
        ch2 <- 123    // 等待 goroutine B 接收 ch2
    }()

    go func() {
        <-ch2         // 等 ch2 有数据
        ch1 <- 456    // 等待 goroutine A 接收 ch1
    }()
}

// ✅ 修复1：有缓冲 channel
func fixedWithBuffer() {
    ch1 := make(chan int, 1) // 缓冲容量1
    ch2 := make(chan int, 1)
    ch1 <- 1                 // 不阻塞
    ch2 <- 2
}

// ✅ 修复2：用 select default 避免阻塞
func fixedWithSelect() {
    ch := make(chan int)
    select {
    case ch <- 42:
        fmt.Println("sent")
    default:
        fmt.Println("ch blocked, do something else")
    }
}
```

#### 2.2 sync.Mutex 不可重入（Go 1.18+ 会直接 panic）

```go
// ❌ 死锁：同一个 goroutine 重复加锁
type Counter struct {
    mu sync.Mutex
    n  int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    c.mu.Lock()  // ❌ Go 1.18+ panic: sync: unlock of unlocked mutex
    defer c.mu.Unlock()
    c.n++
}

// ✅ 修复：重构代码，避免在持锁区域内重复加锁
func (c *Counter) SafeIncrement() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.increment()  // 在锁内调用的是普通方法，不是 Lock() 本身
}

func (c *Counter) increment() {
    c.n++  // 不需要再加锁
}
```

**为什么 Go 的 Mutex 不可重入？**
> 不可重入设计是刻意的——Mutex 的解锁语义假设"谁加的锁谁解"，可重入会引入复杂的递归锁问题。如果需要可重入，用 `sync.RWMutex`（读锁可重入，读转写时降级锁）或自己封装一层。

#### 2.3 select 永久阻塞（没有 default 且所有 case 都阻塞）

```go
// ❌ 死锁：select 永远选不出可执行的 case
func selectDeadlock() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    select {
    case <-ch1:
        fmt.Println("ch1 received")
    case <-ch2:
        fmt.Println("ch2 received")
    // 没有 default，也没有超时，永远阻塞
    }
}

// ✅ 修复1：加 default（走默认分支）
func selectWithDefault() {
    ch := make(chan int)
    select {
    case <-ch:
        fmt.Println("received")
    default:
        fmt.Println("no data available")
    }
}

// ✅ 修复2：加超时（最常见做法）
func selectWithTimeout() {
    ch := make(chan int)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    select {
    case <-ch:
        fmt.Println("received")
    case <-ctx.Done():
        fmt.Println("timeout:", ctx.Err())
    }
}
```

#### 2.4 嵌套 Channel + select 悬停

```go
// ❌ 死锁：select 的 case 都阻塞，但外层 goroutine 永远等不到
func nestedSelectDeadlock() {
    outer := make(chan int)
    inner := make(chan int)

    go func() {
        select {
        case <-outer:
            inner <- 1
        case <-inner:
            outer <- 1
        // 两个都阻塞 → 永久等待
        }
    }()

    // 主 goroutine 也没有任何发送动作
    <-make(chan struct{}) // 永久阻塞在这里
}

// ✅ 修复：给 select 加超时或 default
func nestedSelectFixed() {
    outer := make(chan int)
    inner := make(chan int)
    done := make(chan struct{})

    go func() {
        select {
        case <-outer:
            inner <- 1
        case <-inner:
            outer <- 1
        case <-time.After(3 * time.Second): // 超时退出
            done <- struct{}{}
        }
    }()

    <-done
}
```

### 3. 活锁（Live Lock）

活锁不是死锁，但同样让程序无法推进——goroutine 在不断行动但永远无法完成。

```go
// 典型活锁：两个 goroutine 互相礼让，但永远碰不上
func liveLock() {
    ch1 := make(chan int, 1)
    ch2 := make(chan int, 1)
    ch1 <- 1
    ch2 <- 1

    for i := 0; i < 100; i++ {
        select {
        case <-ch1:
            fmt.Println("goroutine 1 got ch1, giving ch2")
            ch2 <- 1
        case <-ch2:
            fmt.Println("goroutine 2 got ch2, giving ch1")
            ch1 <- 1
        }
        // 每次都礼让对方，没有实质性推进
    }
}

// ✅ 修复：随机退避或设置最大重试次数
func liveLockFixed() {
    ch1 := make(chan int, 1)
    ch2 := make(chan int, 1)
    ch1 <- 1

    for i := 0; i < 100; i++ {
        select {
        case <-ch1:
            ch2 <- 1
            return // 实质性完成
        case <-ch2:
            ch1 <- 1
            return
        case <-time.After(time.Duration(rand.Intn(100)) * time.Millisecond):
            // 随机退避，打破对称性
        }
    }
}
```

### 4. 生产级死锁排查工具

#### 4.1 go-deadlock（推荐）

```bash
go get github.com/sasha-s/go-deadlock
```

```go
import "github.com/sasha-s/go-deadlock"

// 替换 sync.Mutex → deadlock.Mutex
type Counter struct {
    mu deadlock.Mutex  // 自动在 30s 后检测死锁并报告 goroutine 栈
    n  int
}

// 输出示例（30s 后）:
// Potentially busy/locked lock:
// goroutine 12345: lock
//   ./main.go:42 Counter.Increment()
// goroutine 12346: trying to lock
//   ./main.go:55 Counter.Increment()
```

**生产环境部署注意：**
```go
// 只在 debug 模式开启
if os.Getenv("DEBUG") == "1" {
    go deadlock.Opts.OnPotentialDeadlock = func() {
        // 发告警，不要直接 os.Exit()（会中断服务）
        alertSlack("deadlock detected")
    }
}
```

#### 4.2 pprof goroutine profile（在线排查）

```bash
# 抓取当前所有 goroutine 栈
go tool pprof http://localhost:6060/debug/pprof/goroutine?debug=1
```

**典型死锁的 pprof 特征：**
```
goroutine profile: total 100
50 @...channelreceive
30 @...channelsend
20 @...sync.Mutex.Lock
```
大量 goroutine 阻塞在 channel/Mutex 上，且状态不变 → 高度怀疑死锁。

#### 4.3 runtime trace（Go 1.11+）

```bash
# 开启 trace
trace.Start(os.Stderr)
defer trace.Stop()

// 访问 http://localhost:6060/debug/pprof/trace?seconds=10
// 在 trace 视图中找 "Goroutine states" 面板
// 如果大量 G 长期处于 "GC assist wait" 或 "semacquire" 状态 → 死锁
```

#### 4.4 go version >= 1.18 runtime 改进

Go 1.18+ 在 `sync.Mutex` 不可重入时会直接 panic：
```go
// Go 1.18+ 会 panic:
// "sync: unlock of unlocked mutex"
c.mu.Lock()
c.mu.Lock()  // → panic
```

### 5. 预防死锁的最佳实践

```go
// 原则1：按固定顺序获取多把锁（打破循环等待）
type Account struct {
    mu   sync.Mutex
    id   int
    balance float64
}

func Transfer(a, b *Account, amount float64) {
    // 始终按 id 顺序加锁，避免 ABBA 死锁
    if a.id < b.id {
        a.mu.Lock()
        b.mu.Lock()
    } else {
        b.mu.Lock()
        a.mu.Lock()
    }
    defer func() {
        b.mu.Unlock()
        a.mu.Unlock()
    }()
    // 转账逻辑...
}

// 原则2：用 tryLock 避免永久等待
func tryLockDemo(m *sync.Mutex) bool {
    if m.TryLock() {
        fmt.Println("got lock")
        defer m.Unlock()
        return true
    }
    fmt.Println("lock busy, retry later")
    return false
}

// 原则3：channel 操作设置超时（避免永久阻塞）
func withTimeout() {
    ch := make(chan int)
    select {
    case <-ch:
        fmt.Println("received")
    case <-time.After(5 * time.Second):
        fmt.Println("timeout, give up")
    }
}
```

---

## 高频追问

**Q：Go 的 channel 死锁和 Java 的 wait/notify 死锁有什么区别？**
> 本质相同（互相等待），但触发方式不同。Java 死锁通常是多把锁+synchronized 嵌套；Go 死锁更常见的是无缓冲 channel 的天然同步特性（发送和接收必须配对）。Go 的 channel 是"发送者阻塞直到有接收者"，如果没有接收者就会死锁——这是 Go CSP 模型的设计代价。

**Q：sync.Map 有死锁问题吗？**
> sync.Map 的读写分离设计（readOnly + dirty）避免了锁竞争热点，但 `LoadOrStore` 在 key 不存在时需要加锁写入，仍然存在低概率的锁竞争。sync.Map 的优势是**读多写少**场景的无锁读取（readOnly 快照），不是解决死锁。

**Q：已经有 go-deadlock 了，为什么还要用 pprof trace？**
> go-deadlock 依赖定期检测，有 ~30s 的盲区；pprof 可以**实时抓取**所有 goroutine 栈，不需要重启服务；runtime trace 可以看到**调度层面的阻塞**（不仅仅是锁，还包括 channel、系统调用等）。

---

**[← 上一篇：Race Condition](./race-condition.md)** · **[目录](../README.md)** · **[下一篇：sync 原语](./02-sync.md)**
