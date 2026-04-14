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
