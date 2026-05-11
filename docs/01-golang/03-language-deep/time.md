[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [💎 语言机制](../README.md)

---

# time.Timer / time.Ticker：常见陷阱与正确用法

> 考察频率：★★★☆☆  优先级：P1（高频面试点）
> 关键词：Timer、Ticker、内存泄漏、stop、reset、精度问题、时区处理

---

## 面试官考察意图

考察候选人对 Go time 包的掌握深度，以及在生产环境中的正确使用。
初级只知道"Timer 是延时，Ticker 是周期"，高级要能讲清楚 **Timer/Ticker 的底层实现（C runtime）、Stop/Reset 的正确姿势、Ticker 的内存泄漏风险、以及在 HTTP/Gateway 场景中如何避免定时器堆积导致的 OOM**。

---

## 核心答案（30 秒版）

| 原语 | 用途 | 特点 |
|------|------|------|
| `time.Timer` | 单次延时 | 触发一次后自动停止 |
| `time.Ticker` | 周期执行 | **必须手动 Stop**，否则内存泄漏 |
| `time.After` | 一次性延时（超时）| 每次调用创建新 Timer，要配合 select 用 |
| `time.Sleep` | 阻塞延时 | 最简单，但无法中断 |

**最常考的坑：** `time.Ticker` 如果忘记 `Stop()`，goroutine 和 timer 会永久泄漏。

---

## 深度展开

### 1. time.Timer 底层原理

Timer 的实现依赖 runtime timer：

```go
// Timer 本质是一个 runtime timer，不占用独立的 goroutine
timer := time.NewTimer(5 * time.Second)
select {
case <-timer.C:
    fmt.Println("5 seconds passed")
}
```

**NewTimer vs time.After：**

```go
// ✅ time.After：每次调用创建新 Timer（适用于 select 超时）
select {
case <-ch:
    fmt.Println("received")
case <-time.After(5 * time.Second):  // ❌ 但这个 timer 不会被 GC（泄漏风险）
    fmt.Println("timeout")
}
// 问题：<-time.After(d) 每次创建新 timer，触发后会等待 GC

// ✅ 正确超时写法（推荐）：
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
select {
case <-ch:
    fmt.Println("received")
case <-ctx.Done():
    fmt.Println("timeout")
}

// ✅ time.After 的正确用法（短期程序）：
timer := time.NewTimer(5 * time.Second)
defer timer.Stop()  // 主动停止，释放资源
select {
case <-ch:
    fmt.Println("received")
case <-timer.C:
    fmt.Println("timeout")
}
```

### 2. Timer.Stop() 和 Timer.Reset()（易错）

#### 2.1 Stop()：停止 Timer

```go
// ❌ 常见错误：Stop 后不检查返回值，不知道 timer 是否已触发
timer := time.NewTimer(3 * time.Second)
// ... 一些逻辑 ...
timer.Stop()  // 只停止，还不知道有没有触发过

// ✅ 正确做法：检查返回值
timer := time.NewTimer(3 * time.Second)
stopped := timer.Stop()
if stopped {
    fmt.Println("timer stopped before firing")
    // timer.C 永远不会被发送数据
} else {
    fmt.Println("timer already fired")
    // timer.C 已经被发送数据
}
```

#### 2.2 Reset()：重置 Timer（高频坑）

```go
// ❌ time.After 的常见误用：每次循环都创建新 timer
for {
    select {
    case <-time.After(1 * time.Second):  // ❌ 每次迭代创建新 timer，上一个泄漏
        doWork()
    }
}

// ✅ 正确写法：用 NewTimer + Reset 复用
func workerWithReset() {
    timer := time.NewTimer(1 * time.Second)
    defer timer.Stop()

    for {
        timer.Reset(1 * time.Second)  // 复用同一个 timer
        select {
        case <-jobChan:
            doWork()
        case <-timer.C:
            fmt.Println("timeout")
        }
    }
}

// ✅ 更好的方式：context 超时（推荐）
func workerWithContext() {
    for {
        ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
        select {
        case <-jobChan:
            cancel()
            doWork()
        case <-ctx.Done():
            fmt.Println("timeout")
        }
    }
}
```

### 3. time.Ticker：内存泄漏的头号凶手

```go
// ❌ 内存泄漏：goroutine + timer 永久泄漏
func leakyTicker() {
    ticker := time.NewTicker(1 * time.Second)
    go func() {
        for range ticker.C {  // ticker 永远不被停止，goroutine 永远在
            doWork()
        }
    }()
    // 函数退出，ticker 没有 Stop()，timer 和 goroutine 永久泄漏
}

// ✅ 正确做法：必须 Stop
func correctTicker() {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()  // 关键：退出前必须 Stop
    go func() {
        for range ticker.C {
            doWork()
        }
    }()
    // 或在外层逻辑完成后调用 ticker.Stop()
}

// ✅ 另一个正确做法：使用 context 取消
func tickerWithContext() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    go func() {
        for {
            select {
            case <-ticker.C:
                doWork()
            case <-ctx.Done():
                return
            }
        }
    }()
}
```

### 4. 精度问题：time.Sleep 不是高精度定时

```go
// ❌ 不要依赖 time.Timer 做高精度定时（Go 不保证精度）
// Go 的 timer 精度受限于调度器和系统时钟分辨率

// ✅ 高精度场景：直接用 runtime.nanotime 或硬件定时器
// Go timer 的精度通常在 1ms~10ms 级别，不适合微秒级需求
func highPrecisionDemo() {
    // 不适合：定时 100 微秒级别任务
    // 适合：心跳检测、健康检查（秒级）
}

// ✅ time.Now() 用于时间戳，性能优先用 time.Since() 而非 time.Now().Sub()
start := time.Now()
// ... 业务逻辑 ...
elapsed := time.Since(start)  // ✅ 比 time.Now().Sub(start) 更快（少一次 syscall）
fmt.Printf("took %v\n", elapsed)
```

### 5. 时区处理（易错）

```go
// ❌ 常见时区错误：time.LoadLocation 失败导致程序崩溃
loc, err := time.LoadLocation("Asia/Shanghai")
if err != nil {
    fmt.Println("failed to load timezone:", err)
    // 不要 panic，生产环境要有 fallback
    loc = time.UTC  // fallback
}

// ✅ 正确做法：内置时区
func timeInBeijing() time.Time {
    loc := time.FixedZone("CST", 8*60*60)  // UTC+8 固定时区
    return time.Now().In(loc)
}

// ✅ 跨时区存储：用 Unix 时间戳而非字符串
// 数据库/消息队列存 UTC，解析时再转本地时区
unixTs := time.Now().UTC().Unix()  // 存储
readBack := time.Unix(unixTs, 0)   // 读取
```

### 6. 生产环境案例：定时任务泄漏排查

```go
// 问题代码：HTTP handler 中创建 Ticker
func leakyHandler() {
    ticker := time.NewTicker(1 * time.Second)
    go func() {
        for range ticker.C {
            checkHealth()
        }
    }()
    // handler 结束，ticker 没有 Stop → 泄漏
}

// ✅ 正确做法：在全局 init 或单独 goroutine 中管理
type Scheduler struct {
    ticker *time.Ticker
    stop   chan struct{}
}

func NewScheduler(interval time.Duration) *Scheduler {
    s := &Scheduler{
        ticker: time.NewTicker(interval),
        stop:   make(chan struct{}),
    }
    go s.run()
    return s
}

func (s *Scheduler) run() {
    for {
        select {
        case <-s.ticker.C:
            checkHealth()
        case <-s.stop:
            s.ticker.Stop()
            return
        }
    }
}

func (s *Scheduler) Stop() {
    close(s.stop)
}
```

---

## 高频追问

**Q：Timer 和 Ticker 底层用的是什么机制？**
> Go runtime 维护一个全局 timer wheel（2^30 ns ~ 1s 精度），不依赖 OS 线程。Timer/Ticker 的 goroutine 阻塞在 channel 接收（`timer.C`），由 runtime 的 `sysmon` 协程触发 timer 并向 channel 发送数据。goroutine 本身不消耗线程资源（等待时 park）。

**Q：大量短生命周期 Timer 会影响性能吗？**
> 会。每个 `time.After()` 都在 runtime timer wheel 中注册一个 timer，大量短期 timer 会加重 sysmon 负担。**建议用 context 超时代替 `time.After`**：context 超时是链表管理，不依赖全局 timer wheel，开销更小。

**Q：Ticker.Stop() 和 Ticker=nil 有什么区别？**
> `ticker.Stop()` 停止定时器，释放资源，但 channel 和 goroutine 仍然存在（goroutine 等待 channel）。`ticker = nil` 只是清空引用，timer 仍在运行，继续泄漏。

---

**[← 上一篇：io.Reader/Writer](./io-reader-writer.md)** · **[目录](../README.md)** · **[下一篇：Closure 闭包](./19-closure.md)**
