# Goroutine 泄漏

> 考察频率：★★★★☆  难度：★★★★☆

## 什么是 Goroutine 泄漏

Goroutine 有自己的栈和调度器，理论上执行完毕会自动释放。但若：

1. **阻塞在 channel**（无人读取/关闭）
2. **阻塞在 mutex/sync.WaitGroup**（未 signal/wait）
3. **死循环**（业务逻辑错误）

就会导致 goroutine 永远无法退出，累积占用内存。

## 如何识别泄漏

### 1. pprof goroutine profile

```bash
# 采集 goroutine profile
curl http://localhost:6060/debug/pprof/goroutine > goroutine.prof

# 查看当前 goroutine 数量
curl http://localhost:6060/debug/pprof/goroutine?debug=1
```

### 2. 监控 runtime.metrics

```go
var m struct {
    goroutineMetric runtime.MemStats
}

// 每分钟检查
go func() {
    ticker := time.NewTicker(time.Minute)
    for range ticker.C {
        n := runtime.NumGoroutine()
        if n > threshold {
            log.Printf("WARNING: too many goroutines: %d", n)
        }
    }
}()
```

### 3. 压测观察

```bash
# 使用 hey 或 wrk 压测
hey -n 10000 -c 100 http://localhost:8080/endpoint

# 同时监控 goroutine 数量变化
watch -n 0.5 'curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | grep -c "^goroutine"'
```

---

## 常见泄漏场景

### 场景 1：channel 无人读取

```go
// 泄漏
func producer() {
    ch := make(chan int)
    for i := 0; i < 100; i++ {
        ch <- i
    }
    // ch 没人读取，发送方阻塞
}

// 修复
func producerFixed() {
    ch := make(chan int, 100) // 带缓冲
    for i := 0; i < 100; i++ {
        ch <- i
    }
    close(ch) // 关闭 channel
}
```

### 场景 2：context 超时未处理

```go
// 泄漏
func request() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

    result := make(chan string)

    go func() {
        // 模拟慢操作
        time.Sleep(5 * time.Second)
        result <- "done"
    }()

    select {
    case r := <-result:
        fmt.Println(r)
    case <-ctx.Done():
        // context 超时后，goroutine 仍在运行
    }
}

// 修复：确保 goroutine 可退出
func requestFixed() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

    done := make(chan string, 1)

    go func() {
        time.Sleep(5 * time.Second)
        select {
        case done <- "done":
        case <-ctx.Done():
            // 响应 ctx 取消信号
        }
    }()

    select {
    case r := <-done:
        fmt.Println(r)
    case <-ctx.Done():
        fmt.Println("timeout")
    }
}
```

### 场景 3：goroutine 泄露在 select 中

```go
// 泄漏：default 分支导致死循环
func leaky() {
    ch := make(chan int)
    go func() {
        for {
            select {
            case v := <-ch:
                fmt.Println(v)
            default: // 无消息时死循环
            }
        }
    }()
}

// 修复
func fixed() {
    ch := make(chan int)
    done := make(chan struct{})

    go func() {
        for {
            select {
            case v := <-ch:
                fmt.Println(v)
            case <-done:
                return
            }
        }
    }()

    // 需要退出时
    close(done)
}
```

### 场景 4：WaitGroup 未 Done

```go
// 泄漏
func process() {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            doWork(i)
            // 忘记 wg.Done()
        }()
    }
    wg.Wait()
}

// 修复
func processFixed() {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            doWork(i)
        }(i)
    }
    wg.Wait()
}
```

---

## 泄漏检测工具

### go-leak-check

```bash
go install github.com/uber-go/go-leak-check@latest

# 运行测试时检测泄漏
go-leak-check ./...
```

### 代码审查要点

```go
// 1. channel 要么关闭，要么有接收方
// 2. select 一定有 default 或 <-done
// 3. WaitGroup 一定要 defer wg.Done()
// 4. Context 要传播 cancel
// 5. for-range channel 一定会被 close
```

---

## 修复模式

### Worker Pool（推荐）

```go
func workerPool(jobs <-chan int, workers int) {
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                process(job)
            }
        }()
    }
    wg.Wait()
}
```

### 带退出信号的 Worker

```go
func workerWithSignal(jobs <-chan int, done <-chan struct{}) {
    for {
        select {
        case job := <-jobs:
            process(job)
        case <-done:
            return
        }
    }
}
```

---

## 总结：Goroutine 泄漏排查

| 步骤 | 方法 |
|------|------|
| 发现 | pprof goroutine profile、NumGoroutine() 监控 |
| 定位 | 对比 goroutine 堆栈变化 |
| 修复 | channel 关闭、context 取消、WaitGroup Done |
| 预防 | Worker Pool、统一 goroutine 管理 |

### 泄漏检查代码模式

```go
// 确保 goroutine 可退出
func safeGoroutine(fn func(ctx context.Context)) func(context.Context) {
    return func(ctx context.Context) {
        go func() {
            fn(ctx)
        }()
    }
}
```

---

## 五、Go 1.26 实验性功能：Goroutine Leak Profile

> Go 1.26 新增实验性功能，通过 GC 协作检测**永久阻塞型 goroutine 泄漏**，无需手动埋点。

### 5.1 什么是永久阻塞型泄漏

传统 pprof goroutine profile 只能看到"当前有多少 goroutine"，但无法判断哪些是**永久无法唤醒**的。Goroutine Leak Profile 使用 GC 协作检测，一旦 GC 发现某个 goroutine 阻塞在某个同步原语上、且该同步原语从任何可达路径都无法被解锁，则判定为泄漏。

**典型泄漏场景：**

```go
func processWorkItems(ws []workItem) ([]workResult, error) {
    ch := make(chan result) // 无缓冲 channel
    for _, w := range ws {
        go func(item workItem) {
            res, err := processWorkItem(item)
            ch <- result{res, err} // 如果 early return，这些 goroutine 永久阻塞
        }(w)
    }
    var results []workResult
    for range len(ws) {
        r := <-ch
        if r.err != nil {
            return nil, r.err // ⚠️ 提前返回，未读取完的 goroutine 泄漏
        }
        results = append(results, r.res)
    }
    return results, nil
}
```

### 5.2 启用方式

**编译/运行时刻启用：**

```bash
# 方式1：设置 GOEXPERIMENT
GOEXPERIMENT=goroutineleakprofile go run .

# 方式2：采集 profile
curl -o goroutineleak.prof 'http://localhost:6060/debug/pprof/goroutineleak?debug=1'

# 方式3：通过 net/http/pprof
# 启用后自动注册 /debug/pprof/goroutineleak 端点
```

**代码中启用：**

```go
import _ "net/http/pprof" // 注册 pprof handler
// 访问：http://host:port/debug/pprof/goroutineleak?debug=1
```

### 5.3 检测原理（面试可讲）

**GC 协作检测算法：**

1. **标记阻塞点**：当 goroutine G 阻塞在同步原语 P（channel、Mutex、Cond 等）时，记录 (G, P) 对
2. **GC 传播可达性**：GC 从所有 runnable goroutine 出发，追踪它们直接或间接能触达的同步原语集合 Reach(P)
3. **泄漏判定**：如果 P 不在 Reach(P) 中（没有任何 runnable goroutine 能解锁 P），则 G 永久阻塞 → 泄漏
4. **局限性**：如果 P 通过全局变量或 runnable goroutine 的局部变量可达，GC 无法检测

**技术来源**：Vlad Saioc (Uber)，发表在 [PLDI 2025](https://dl.acm.org/doi/pdf/10.1145/3676641.3715990)。

### 5.4 与传统方法对比

| 方法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| pprof goroutine profile | 堆栈采样 | 通用、工具成熟 | 无法区分活跃/泄漏 |
| `runtime.NumGoroutine()` 监控 | 计数 | 简单、告警容易 | 无法定位泄漏点 |
| **Goroutine Leak Profile** | GC 协作检测 | 自动发现永久阻塞泄漏 | 实验阶段，无法检测全局变量间接引用 |
| go-leak-check (uber) | 测试框架 hook | CI 集成、精确 | 仅测试环境、需提前埋点 |

### 5.5 生产使用建议

```bash
# 生产环境定期采集（建议：每 5 分钟一次，随机选一个实例）
# 通过 CI/监控平台自动对比泄漏 goroutine 数量变化

# 查看泄漏 goroutine 堆栈
curl 'http://localhost:6060/debug/pprof/goroutineleak?debug=1' | grep -A30 "goroutine leak"
```

**计划：** Go 1.27 计划默认启用此 profile，届时无需设置 `GOEXPERIMENT`。

### 5.6 高频追问

**Q：GC 协作检测会增加 GC 开销吗？**

仅在主动采集 profile 时有少量开销（记录阻塞点）。不启用则零开销。

**Q：为什么需要 GOEXPERIMENT？**

API（profile 类型名、pprof 端点）可能在 Go 1.27 正式版中调整。当前是收集生产反馈的实验阶段。
