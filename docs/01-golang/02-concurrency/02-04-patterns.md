# Go 并发模式：Pipeline、Fan-out/Fan-in、errgroup

> 考察频率：★★★★★  优先级：P0
> 关键词：Pipeline、Fan-out/Fan-in、errgroup、select、超时控制

## 为什么并发模式重要

5-8 年 Go 工程师的核心区分度之一就是**能否正确设计高并发流水线**。面试官不仅看你会不会用 goroutine，更看你能否组织出一个**高效、稳定、可控**的并发模型。

---

## 1. Pipeline（流水线）模式

### 核心思想

将一个复杂任务拆分为**多个串行阶段（Stage）**，每个阶段由独立的 goroutine 处理，通过 channel 连接上下游。

```
数据流：
Source → Stage1(ch) → Stage2(ch) → Stage3(ch) → Sink
         goroutine   goroutine   goroutine
```

### 典型案例：文件处理流水线

```go
// Stage 函数签名：接收输入 channel，返回输出 channel
func pipelineStage(in <-chan string) <-chan string {
    out := make(chan string, 100)
    go func() {
        defer close(out)
        for s := range in {
            // 处理逻辑
            processed := strings.ToUpper(s)
            out <- processed
        }
    }()
    return out
}

// 使用 Pipeline
func main() {
    // 1. 数据源（生产者）
    source := generateStrings(1000)

    // 2. 三个处理阶段串联
    stage1 := pipelineStage(source)          // 转大写
    stage2 := filterNonEmpty(stage1)          // 过滤空字符串
    stage3 := addPrefix(stage2)               // 加前缀

    // 3. 消费结果
    for result := range stage3 {
        fmt.Println(result)
    }
}

func generateStrings(n int) <-chan string {
    out := make(chan string)
    go func() {
        defer close(out)
        for i := 0; i < n; i++ {
            out <- fmt.Sprintf("item-%d", i)
        }
    }()
    return out
}
```

### Pipeline 的关键原则

1. **每个 Stage 负责关闭自己的输出 channel**（上游关闭，下游遍历完）
2. **消费者（下游）负责通知上游停止**（通过 close 输入 channel）
3. ** channel 要有缓冲**：避免上下游速度不匹配时阻塞

```go
// 错误：没有缓冲的 channel 可能导致死锁
// stage1 处理慢，stage2 处理快 → stage2 阻塞等待 stage1
stage1 := make(chan int)        // 无缓冲！
stage2 := make(chan int, 100)   // 有缓冲！

// 正确：根据下游消费速度设置合理缓冲
// 一般 buffer 大小 = 并行度 × 每批次大小
stage1 := make(chan int, runtime.GOMAXPROCS(0)*10)
```

---

## 2. Fan-out / Fan-in（多路复用）

### Fan-out：多个 goroutine 处理同一 channel

```go
// Fan-out：启动多个 worker 并行处理同一 channel
func fanOut(in <-chan Job, workers int) <-chan Result {
    out := make(chan Result, workers*10)

    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for job := range in {
                out <- processJob(job) // worker id 可用于负载追踪
            }
        }(i)
    }

    // 等待所有 worker 完成后关闭 out
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

**适用场景**：
- 多个独立任务可以并行处理（CPU 密集或 IO 密集）
- worker 之间无状态共享

### Fan-in：合并多个 channel 到一个

```go
// Fan-in：把多个 channel 合并成一个
func merge(channels ...<-chan string) <-chan string {
    out := make(chan string)

    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan string) {
            defer wg.Done()
            for s := range c {
                out <- s
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

// 使用：三个源合并消费
ch1 := source("A")
ch2 := source("B")
ch3 := source("C")

for v := range merge(ch1, ch2, ch3) {
    fmt.Println(v)
}
```

### Fan-out/Fan-in 组合使用

```go
// 完整的并行 Pipeline：Source → Fan-out(3 workers) → Fan-in → Sink
func parallelPipeline(items []int) []int {
    // Stage 1: Source
    in := make(chan int, len(items))
    go func() {
        for _, item := range items {
            in <- item
        }
        close(in)
    }()

    // Stage 2: Fan-out（3 个 worker 并行处理）
    workerCount := 3
    results := make([]<-chan int, workerCount)
    for i := 0; i < workerCount; i++ {
        results[i] = processWorker(in)
    }

    // Stage 3: Fan-in（合并结果）
    merged := fanInInt(results...)

    // Stage 4: 收集结果
    var output []int
    for v := range merged {
        output = append(output, v)
    }
    return output
}

func processWorker(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for item := range in {
            out <- item * 2 // 模拟处理
        }
    }()
    return out
}

func fanInInt(channels []<-chan int) <-chan int {
    out := make(chan int)
    go func() {
        var wg sync.WaitGroup
        for _, ch := range channels {
            wg.Add(1)
            go func(c <-chan int) {
                defer wg.Done()
                for v := range c {
                    out <- v
                }
            }(ch)
        }
        wg.Wait()
        close(out)
    }()
    return out
}
```

---

## 3. errgroup（错误传播）

### 原生 errgroup 用法

`golang.org/x/sync/errgroup` 是最常用的并发错误处理工具。

```go
import "golang.org/x/sync/errgroup"

func fetchAllURLs(urls []string) ([]string, error) {
    g, ctx := errgroup.WithContext(context.Background())

    results := make([]string, len(urls))
    for i, url := range urls {
        i, url := i, url // 捕获循环变量
        g.Go(func() error {
            resp, err := http.Get(url)
            if err != nil {
                return fmt.Errorf("请求 %s 失败: %w", url, err)
            }
            defer resp.Body.Close()

            body, err := io.ReadAll(resp.Body)
            if err != nil {
                return fmt.Errorf("读取 %s 响应失败: %w", url, err)
            }
            results[i] = string(body)
            return nil
        })
    }

    // 等待所有 goroutine 完成，或者任一出错时取消其他
    if err := g.Wait(); err != nil {
        return nil, err // 返回第一个错误
    }
    return results, nil
}
```

### errgroup 的三大特性

| 特性 | 说明 |
|------|------|
| **WithContext** | 任一 goroutine 返回错误，context 被取消，其他 goroutine 收到信号立即退出 |
| **Wait** | 阻塞等待所有 goroutine 完成，返回第一个非 nil 错误 |
| **Go** | 启动 goroutine，类似 go func()，但自动与 group 生命周期绑定 |

### 限制并发数的 Fan-out

```go
// 限制最多 5 个并发
func limitedFetch(urls []string, limit int) error {
    g, ctx := errgroup.WithContext(context.Background())

    // 信号量：控制并发数
    sem := make(chan struct{}, limit)

    for _, url := range urls {
        url := url // 捕获
        g.Go(func() error {
            // 获取信号量（阻塞直到有空闲槽位）
            sem <- struct{}{}
            defer func() { <-sem }()

            select {
            case <-ctx.Done():
                return ctx.Err() // 如果其他 goroutine 已出错，直接退出
            default:
            }

            return fetch(url)
        })
    }
    return g.Wait()
}
```

### errgroup + 超时控制

```go
func fetchWithTimeout(urls []string, timeout time.Duration) ([]string, error) {
    g, ctx := errgroup.WithContext(context.Background())

    ctx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()

    results := make([]string, len(urls))
    for i, url := range urls {
        i, url := i, url
        g.Go(func() error {
            req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
            resp, err := http.DefaultClient.Do(req)
            if err != nil {
                return fmt.Errorf("url %s: %w", url, err)
            }
            defer resp.Body.Close()
            body, _ := io.ReadAll(resp.Body)
            results[i] = string(body)
            return nil
        })
    }
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

---

## 4. 并发模式进阶：半选择（Semaphore）控制并发

### 场景：控制数据库连接池使用

```go
import "golang.org/x/sync/semaphore"

func processItems(items []Item, maxConcurrent int) error {
    // 最多同时处理 maxConcurrent 个任务
    sem := semaphore.NewWeighted(int64(maxConcurrent))

    var wg sync.WaitGroup
    for _, item := range items {
        item := item
        wg.Add(1)

        // 获取信号量
        if err := sem.Acquire(context.Background(), 1); err != nil {
            wg.Done()
            return err
        }

        go func() {
            defer wg.Done()
            defer sem.Release(1) // 释放信号量

            if err := process(item); err != nil {
                // 记录错误，但不中断其他任务
                log.Printf("处理 %v 失败: %v", item.ID, err)
            }
        }()
    }
    wg.Wait()
    return nil
}
```

---

## 5. 真实线上场景：爬虫 Pipeline

```go
// 完整案例：并发爬虫，URL 去重，控制并发 + 错误处理

type Crawler struct {
    client   *http.Client
    limiter  *rate.Limiter       // 限速
    sem      *semaphore.Weighted  // 并发控制
    seen     sync.Map             // URL 去重
}

func (c *Crawler) Crawl(ctx context.Context, startURL string) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx)

    urls := make(chan string, 100)
    results := make(chan string, 100)

    // 生产者
    g.Go(func() error {
        defer close(urls)
        urls <- startURL
        return nil
    })

    // 消费者：Fan-out（5个worker并发爬取）
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        g.Go(func() error {
            defer wg.Done()
            for url := range urls {
                select {
                case <-ctx.Done():
                    return ctx.Err()
                default:
                }

                c.sem.Acquire(ctx, 1)
                page, err := c.fetch(ctx, url)
                c.sem.Release(1)

                if err != nil {
                    log.Printf("爬取失败 %s: %v", url, err)
                    continue
                }

                results <- page

                // 发现新 URL，加入队列
                for _, newURL := range extractLinks(page) {
                    if _, loaded := c.seen.LoadOrStore(newURL, true); !loaded {
                        select {
                        case urls <- newURL:
                        default:
                        }
                    }
                }
            }
            return nil
        })
    }

    // 等待完成后关闭 results
    go func() {
        wg.Wait()
        close(results)
    }()

    var pages []string
    for p := range results {
        pages = append(pages, p)
    }

    return pages, g.Wait() // 返回结果或错误
}
```

---

## 面试高频追问

**Q1: Fan-out 什么时候用？什么时候不用？**
→ 用：任务可并行、独立无依赖（爬虫、批量 API 调用）。不用：任务有顺序依赖（必须 A→B→C）、资源竞争激烈（数据库连接池不足）、任务非常轻量（goroutine 开销大于任务本身）。

**Q2: errgroup 和 sync.WaitGroup 的区别？**
→ WaitGroup 只等待，不传播错误。errgroup 的 WithContext 更强大：任一出错时自动取消其他 goroutine（通过 context），避免资源浪费。

**Q3: channel 缓冲大小怎么定？**
→ 经验公式：`cap = workers * batch_size`（防止最坏情况下阻塞）。太小：上游阻塞；太大：内存占用高、延迟增加。IO 密集型任务可以适当调大。

**Q4: Pipeline 出现慢消费者怎么办？**
→ ① 监控 channel 长度，超阈值告警；② 设置 channel 缓冲 + select 超时退出；③ 使用 semaphore 控制生产速度；④ 慢消费者单独扩容或降级。
