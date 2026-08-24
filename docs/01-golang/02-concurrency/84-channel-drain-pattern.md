# Channel 安全排空模式：Drain、Close 与生命周期管理

> 考察频率：★★★★☆  优先级：P1
> 关键词：channel drain、range、close、goroutine lifecycle、select default panic

---

## 面试官考察意图

这道题考察候选人对 **channel 生命周期管理的实战能力**。初级只知道 `close(ch)` 关闭 channel；高级要能讲清楚 **安全的 drain 策略、select + range 的性能对比、panic 的触发场景，以及在复杂并发模式下如何避免 goroutine 泄漏**。这是实际工程中极其常见但面试很少系统覆盖的实践考点。

---

## 核心答案（30 秒版）

**Channel Drain（安全排空）**是将 channel 中所有剩余数据取出的一种操作：

```go
// 推荐方式：for range
for range ch { /* 处理每个值 */ }

// 快速排空（不在乎值）
func drain(ch <-chan T) {
    for range ch {}
}
```

**关键规则：**
- `close()` 后仍有数据的 channel 可以继续 `range` 直到清空
- 向已 close 的 channel **发送** → panic
- 从已 close 且**无数据**的 channel **接收** → 返回零值和 `ok = false`
- `select` 中的 `default` 在通道操作不阻塞时立即执行

---

## 深度展开

### 1. 三种 Drain 模式

#### 模式 A：for range（推荐，最安全）

```go
func processAll(ch chan int) {
    count := 0
    for v := range ch {
        fmt.Println(v)
        count++
    }
    // 当 channel 被 close 且队列为空时，range 循环自然退出
    fmt.Printf("processed %d items\n", count)
}
```

**工作原理：** `for range` 会自动重复调用 `<-ch`，每次返回 `(value, ok)`。当 `ok == false`（channel closed 且 drained），循环终止。

#### 模式 B：select + default（快速排空）

```go
// 不关心值的快速排空
func fastDrain(ch chan int) {
    for {
        select {
        case _, ok := <-ch:
            if !ok {
                return
            }
        default:
            return  // channel 已空但没 close，也退出
        }
    }
}
```

**注意：** 这种模式有争议 —— `default` 分支在没有数据但仍开放时会提前退出，可能导致数据遗漏。适合"想要立刻拿到的就拿，来不及的不强求"的场景。

#### 模式 C：drain helper 函数（可复用）

```go
// 标准库风格的 drain helper
func drain(ch <-chan any) {
    for range ch {}
}

// 带超时保护的 drain
func drainWithTimeout(ch <-chan any, timeout time.Duration) bool {
    done := make(chan struct{})
    go func() {
        defer close(done)
        drain(ch)
    }()
    
    select {
    case <-done:
        return true   // 成功排空
    case <-time.After(timeout):
        return false  // 超时
    }
}
```

### 2. Close 后行为对照表

| 操作 | Channel 状态 | 结果 |
|------|-------------|------|
| `<-ch` | open, has data | 返回值，无 error |
| `<-ch` | open, empty | 阻塞等待 |
| `<-ch` | closed, has data | 返回值（ FIFO ） |
| `<-ch` | closed, empty | 返回零值 + `ok = false` |
| `ch <- v` | open | 正常发送 |
| `ch <- v` | closed | **panic: send on closed channel** |
| `close(ch)` | already closed | **panic: close of closed channel** |
| `close(ch)` | nil channel | **panic: close of nil channel** |

### 3. 经典 goroutine 泄漏陷阱

#### 陷阱 1：生产者跑了但没人消费

```go
func produce(ch chan<- int) {
    for i := 0; i < 1000; i++ {
        ch <- i  // 如果没有人接收，goroutine 会永远阻塞
    }
}

func main() {
    ch := make(chan int, 10)
    go produce(ch)
    // 没有人消费 → goroutine 泄漏！
}
```

#### 陷阱 2：消费者只读了部分

```go
func consume(ch <-chan int) {
    for i := 0; i < 5; i++ {
        <-ch  // 只读 5 个就退出
    }
}

func main() {
    ch := make(chan int, 10)
    for i := 0; i < 1000; i++ {
        ch <- i  // 第 11 个开始阻塞！
    }
    // 消费者只读了 5 个，剩余 995 个全部堆积 → goroutine 泄漏
}
```

#### 陷阱 3：没有 close 导致 range 永不结束

```go
func worker(tasks <-chan int, results chan<- int) {
    for task := range tasks {
        results <- task * 2
    }
    // tasks 永远不会 close → worker goroutine 永远不会退出
}

func main() {
    tasks := make(chan int)
    results := make(chan int)
    
    go worker(tasks, results)
    
    for i := 0; i < 5; i++ {
        tasks <- i
    }
    // 忘记 close(tasks) → worker 永远等下一个任务
}
```

### 4. 正确的 Producer-Consumer 生命周期

```go
func runPipeline(ctx context.Context) {
    tasks := make(chan int, 100)
    results := make(chan int, 100)
    
    // Worker 协程（消费者）
    workers := make([]workerResult, numWorkers)
    wg := sync.WaitGroup{}
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for task := range tasks {
                results <- task * 2
            }
        }(i)
        workers[i] = workerResult{id: id}
    }
    
    // 主流程
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    
    go func() {
        for i := 0; i < 1000; i++ {
            select {
            case tasks <- i:
            case <-ctx.Done():
                return
            }
        }
        close(tasks)  // ✅ 发送完成后关闭 channel
    }()
    
    // 等待所有 worker 完成
    go func() {
        wg.Wait()
        close(results)  // ✅ 所有 worker 结束后关闭结果 channel
    }()
    
    // 消费结果
    for result := range results {
        _ = result
    }
}
```

### 5. 实际工程：Worker Pool 完整示例

```go
type Job struct {
    ID     int
    Data   []byte
}

type Result struct {
    JobID int
    Output []byte
    Err error
}

func workerPool(ctx context.Context, jobs []Job, concurrency int) []Result {
    jobChan := make(chan Job, len(jobs))
    resultChan := make(chan Result, len(jobs))
    
    // 启动 worker
    var wg sync.WaitGroup
    for i := 0; i < concurrency; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobChan {
                res := process(job)
                select {
                case resultChan <- res:
                case <-ctx.Done():
                    return
                }
            }
        }()
    }
    
    // 发送 jobs
    for _, j := range jobs {
        jobChan <- j
    }
    close(jobChan)  // 告诉 workers 不再有新任务
    
    // 等待 workers 完成后再关闭 resultChan
    go func() {
        wg.Wait()
        close(resultChan)
    }()
    
    // 收集结果
    var results []Result
    for r := range resultChan {
        results = append(results, r)
    }
    return results
}
```

**关键点：**
1. `jobChan` 由发送方负责关闭（生产完即 close）
2. `resultChan` 由 WaitGroup 保护后关闭（workers 都 exit 才 close）
3. `ctx.Done()` 用于优雅中断
4. 双重 buffer 防止反压死锁

---

## 🗣️ 面试话术

**一句话记住**：Channel 的生命周期三定律 —— 谁创建谁发送、谁发送完谁 close、谁最后做完谁关结果。`for range` 是最安全的 drain 方式，`select + default` 快但有遗漏风险。

---

## 🔗 延伸阅读

- [Go Blog: Go Concurrency Patterns](https://go.dev/blog/concurrency-patterns)
- [Effective Go: Channels](https://go.dev/doc/effective_go#channels)
- [Rob Pike: Go Slices](https://blog.golang.org/slices)
