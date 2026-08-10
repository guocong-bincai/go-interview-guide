# time 包深度解析：Timer/Ticker 正确用法与陷阱

> 考察频率：★★★★☆  优先级：P2
> 关键词：time.Timer、time.Ticker、内存泄漏、Reset、精度问题、时区处理、Go 1.26 定时器改进

---

## 面试官考察意图

考察候选人对 Go time 包的深度理解，特别是定时器的正确用法。
初级只知道 `time.Sleep`，高级要能讲清楚 **Timer/Ticker 的底层实现区别、内存泄漏根因、Reset 的正确使用时机、定时器精度问题、以及时区处理的最佳实践**，并能结合生产环境给出正确的代码写法。

---

## 核心答案（30 秒版）

Timer 和 Ticker 的三个高频陷阱：

1. **Goroutine 泄漏**：未 Stop 的 Ticker 会导致其 goroutine 永远阻塞在 `for range ticker.C`
2. **Timer.Stop() 配合 `<-timer.C` 使用**：Stop() 只是阻止下次触发，已经进入 channel 的值需要显式读取
3. **Reset 时机错误**：Reset 只应在 timer 停止或已过期后调用，运行时调用行为未定义

```
正确写法：
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop() // 必须 Stop，否则 goroutine 泄漏
for range ticker.C {
    // 处理
}
```

---

## 深度展开

### 1. Timer vs Ticker 本质区别

| 特性 | Timer（一次性） | Ticker（周期性） |
|------|---------------|-----------------|
| 触发次数 | 一次 | 无限次，直到 Stop() |
| 底层结构 | `timer` 结构体 | 内部复用 Timer |
| 适用场景 | 延迟执行、超时控制 | 定期任务、轮询 |

两者底层都依赖 **runtime timer** 实现，区别在于 Timer 只触发一次后自动失效，而 Ticker 循环重置自身。

### 2. Ticker 正确用法

#### 常见泄漏写法（❌）

```go
// 错误：goroutine 永远阻塞，因为 channel 永远不会被关闭
go func() {
    for {
        <-ticker.C  // 没有退出条件
        fmt.Println("tick")
    }
}()
```

#### 正确写法（✅）

```go
ticker := time.NewTicker(500 * time.Millisecond)
defer ticker.Stop() // 关键：函数退出前必须 Stop

go func() {
    for range ticker.C {  // for range 自动监听 channel 关闭
        fmt.Println("tick")
    }
    fmt.Println("ticker goroutine exited")
}()

time.Sleep(2 * time.Second)
// defer ticker.Stop() 在这里触发，for range 自动退出
```

**为什么 `for range ticker.C` 不会泄漏？**
因为 `ticker.Stop()` 会 **close(ticker.C)**，close 后 for range 会立即退出。

#### 手动 select 写法

```go
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()

for {
    select {
    case <-ticker.C:
        fmt.Println("tick")
    case <-done:
        return  // 外部信号退出
    }
}
```

### 3. Timer.Stop() 的正确理解

```go
timer := time.NewTimer(5 * time.Second)

// 场景一：还没触发，想提前取消
if stopped := timer.Stop(); stopped {
    fmt.Println("timer was stopped before firing")
}
// 此时 timer.C 中的值已被消费（如果有的话）
// Stop() 返回 true 表示 timer 还没触发

// 场景二：想继续用 reset
timer.Reset(3 * time.Second) // 正确：timer 已停止，可以 Reset
<-timer.C                    // 等待新的 3 秒
```

**Timer.Stop() 的语义**：
- 阻止 timer 未来触发
- 如果 channel 中已有未读值，Stop() **不会消费**这些值
- 返回 true 表示"阻止成功"（即 timer 本来会触发但被你阻止了）

### 4. Timer.Reset 的正确时机

> **⚠️ Go 1.15 之前 Reset 是有问题的，1.15+ 才明确定义**

```go
// ✅ 正确：在 timer 停止或过期后 Reset
timer := time.NewTimer(5 * time.Second)
<-timer.C        // 等待触发
timer.Reset(3)    // OK：已过期，可以 Reset

// 或者
timer := time.NewTimer(5 * time.Second)
timer.Stop()      // 提前停止
timer.Reset(3)    // OK：已停止，可以 Reset

// ❌ 错误：在 timer 运行时 Reset（行为未定义）
timer := time.NewTimer(5 * time.Second)
time.Sleep(1 * time.Second)
timer.Reset(3)    // 危险：timer 还在运行中，Reset 行为未定义
```

### 5. Timer 底层原理： scheduler timer

Timer 依赖 Go runtime 的 timer goroutine 实现：

```go
// runtime/time.go 简化逻辑
type timer struct {
    when     int64      // 触发时间（纳秒）
    period   int64      // 重复间隔（Ticker 用）
    callback func(*g, unsafe.Pointer) // 回调
    arg      unsafe.Pointer
}
```

当 timer 触发时：
1. runtime timer goroutine 调用 `ready(g, now)` 
2. 将 G 标记为 `_Grunnable`，放入 P 的可运行队列
3. 下次 P 调度到这个 G 时执行

**这意味着 Timer/Ticker 的精度取决于 goroutine 调度时机**，不是硬实时。

### 6. 定时器精度问题

#### 最低精度 ~10ms

Go Timer 的调度精度不是纳秒级，**受 scheduler 影响**：

| 定时器间隔 | 实际精度 | 原因 |
|---------|---------|------|
| 1ms | ±1~5ms | scheduler tick |
| 10ms | ±1~3ms | 较好 |
| 100ms+ | ±1ms 内 | 足够精确 |

#### 高精度定时任务：使用 ticker + 计数

```go
// 场景：需要每 10ms 执行一次（高要求）
ticker := time.NewTicker(10 * time.Millisecond)
defer ticker.Stop()

start := time.Now()
count := 0
for range ticker.C {
    count++
    elapsed := time.Since(start)
    if elapsed >= time.Second {
        fmt.Printf("实际频率: %d Hz\n", count)
        break
    }
}
```

#### Go 1.26 Timer 精度改进

Go 1.26 改进了 timer 的唤醒机制，减少了 timer goroutine 的唤醒延迟：

```
Go 1.25 之前：timer goroutine 可能延迟唤醒
Go 1.26+：    使用更精确的 timer bucket 调度
```

### 7. 时区处理

#### 常见时区问题

```go
// ❌ 错误：时区不一致
t := time.Parse("2006-01-02 15:04:05", "2024-03-15 09:00:00")
fmt.Println(t) // 默认 UTC，不是本地时区！

// ✅ 正确：指定时区
loc, _ := time.LoadLocation("Asia/Shanghai")
t, _ := time.ParseInLocation("2006-01-02 15:04:05", "2024-03-15 09:00:00", loc)
fmt.Println(t) // Asia/Shanghai 时区
```

#### 生产环境最佳实践

```go
// 统一使用 UTC 存储，Asia/Shanghai 显示
constTZ = "Asia/Shanghai"

func FormatForUser(t time.Time) string {
    loc, _ := time.LoadLocation(constTZ)
    return t.In(loc).Format("2006-01-02 15:04:05")
}

// 或者用 Unix 时间戳存储
unix := time.Now().Unix()      // UTC 秒
storedAt := time.Unix(unix, 0) // 任意时区都能还原
```

---

## 高频追问

### Q1：为什么 time.Sleep 可以实现精确延时？

`time.Sleep` 调用 `runtime.entersyscall` 让当前 G 绑定到 M，然后由 scheduler 在定时器到期后唤醒。核心在于 Go runtime 维护了一个 **timer wheel**，每个 P 有一个 timer bucket，到期时精确唤醒。

### Q2：Ticker 和 Timer 哪个性能更好？

两者都是基于同一个 timer 机制，**性能相近**。选择依据是使用场景：
- 需要**重复执行** → Ticker
- 只需要**一次延迟** → Timer（省去 Stop 调用）

### Q3：如何实现高精度定时任务（<1ms）？

Go 的 timer 不适合亚毫秒级精度。如需高精度：
1. **使用外部库**：如 `github.com/decred/dcrd/dcrutil/v4/asig`
2. **busy loop + sleep** 混合
3. **硬件定时器**（需要 CGO 调用 OS API）

```go
// 伪代码：高精度混合写法
for {
    start := time.Now()
    // busy loop 等待到接近目标时间
    for time.Since(start) < targetDuration-time.Millisecond {
        // spin
    }
    time.Sleep(targetDuration - time.Since(start)) // 最后用 sleep 精确到点
    doSomething()
}
```

### Q4：time.Now() 的性能瓶颈？

`time.Now()` 在 x86 上使用 `rdtsc` 指令，约 **25ns**，性能足够好。但在极高频调用（百万次/秒）时可以用 `time.Since()` 缓存或改用 `sync/atomic` 时间戳。

---

## 延伸阅读

- [Go time package source](https://github.com/golang/go/tree/master/src/time)
- [Go 1.26 release notes - timer improvements](https://go.dev/doc/go1.26)
- [Rob Pike - "Spaghetti" timer notes](https://groups.google.com/g/golang-dev/c/S7fIzAkZ_6I)
