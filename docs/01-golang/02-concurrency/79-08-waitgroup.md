[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 并发编程](../README.md)

---

# WaitGroup：协程同步的"计数器"

> 考察频率：★★★★★  难度：★★★☆☆
> 关键词：Add/Done/Wait、计数器机制、重入陷阱、goroutine 泄漏、错误传播

## 🎯 面试官考察意图

`sync.WaitGroup` 是 Go 面试中几乎必考的并发原语。面试官想确认：

1. 是否理解 Add/Done/Wait 三个方法的正确调用顺序（Add 必须在启动 goroutine 之前）
2. 是否知道常见的使用陷阱（Done 被跳过、重复 Add/Done、WaitGroup 复制问题）
3. 能否结合实际场景（任务分发、结果汇总）设计正确的同步方案
4. 进阶候选人能否谈到 WaitGroup 的实现原理和替代方案

---

## ⚡ 核心答案（30秒版）

**`sync.WaitGroup` 内部维护一个 64bit 计数器，高 32bit 记录等待计数（wg），低 32bit 记录 Add 增量。**

核心三件套：`Add(delta)` 在启动 goroutine 前增加计数 → `Done()`（本质 Add(-1)）在 goroutine 结束时调用 → `Wait()` 阻塞直到计数归零。

**三个致命坑**：① Done() 因 panic 没执行导致永远阻塞；② Add() 在 goroutine 内调用导致 race；③ WaitGroup 不能复制（拷贝后行为未定义）。生产环境务必用 `defer wg.Done()`，配合 recover 兜底。

---

## 🔬 深度展开

### 1. 基本用法与正确姿势

```go
func fetchURL(ctx context.Context, urls []string) ([]byte, error) {
    var wg sync.WaitGroup
    results := make([]byte, 0, len(urls))
    mu := sync.Mutex{} // 保护共享数据
    
    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done() // ✅ 确保无论正常退出还是 panic 都能 +1
            
            data, err := http.Get(u)
            if err != nil {
                log.Printf("error fetching %s: %v", u, err)
                return
            }
            
            mu.Lock()
            results = append(results, data...)
            mu.Unlock()
        }(url) // ✅ 立即传值，避免闭包捕获循环变量
    }
    
    wg.Wait() // 阻塞等待所有 goroutine 完成
    return results, nil
}
```

### 2. WaitGroup 的内部实现原理

```go
// src/sync/waitgroup.go（简化版结构体）
type WaitGroup struct {
    noCopy noCopy           // 禁止复制
    statep *uint64        // 指向动态分配的 64bit 状态
    sema   uint32         // 信号量，Wait() 阻塞时用到
}

// state 的 64bit 布局：
// ┌─────────────────┬─────────────────┐
// │   高 32 bit     │   低 32 bit     │
// │   wg (等待数)    │   delta (增量)   │
// └─────────────────┴─────────────────┘

// Add 操作
func (wg *WaitGroup) Add(delta int) {
    state := atomic.LoadUint64(wg.statep)
    wg := uint32(state >> 32)      // 读取高32位
    dl := uint32(state)            // 读取低32位
    // ... 检查并更新 ...
}

// Done 本质上就是 Add(-1)
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}

// Wait 阻塞直到 wg 归零
func (wg *WaitGroup) Wait() {
    for {
        state := atomic.LoadUint64(wg.statep)
        wg := uint32(state >> 32)
        if wg == 0 {
            return // 没有待完成的 goroutine，直接返回
        }
        // 使用 syscall 级 wait 阻塞，不被 sysmon 抢占
        runtime_Semacquire(wg.sema)
    }
}
```

关键点：
- **WaitGroup 对象不固定大小**：首次调用 Add/Done/Wait 时，会在堆上分配一个 `uint64` 状态块，statep 指向它
- **sema 是 OS 信号量**：Wait() 内部使用 `runtime_Semacquire`，走的是操作系统级别的睡眠，而不是忙等待
- **Add(-1) 下溢检测**：如果计数器已经是 0 又调用 Done()，会触发 panic："sync: negative WaitGroup counter"

### 3. 四个高频陷阱

#### 陷阱一：Done() 因 panic 被跳过

```go
// ❌ 错误写法
for i := 0; i < n; i++ {
    wg.Add(1)
    go func(idx int) {
        process(idx)  // 如果这里 panic，Done() 永远不会执行！
        wg.Done()     // 💥 可能导致 WaitGroup 永远不为 0
    }(i)
}

// ✅ 正确写法
for i := 0; i < n; i++ {
    wg.Add(1)
    go func(idx int) {
        defer wg.Done()
        process(idx)  // defer 保证即使 panic 也会执行
    }(i)
}
```

#### 陷阱二：Add 在 goroutine 内部调用（race）

```go
// ❌ 危险：竞态条件
var wg sync.WaitGroup
for i := 0; i < n; i++ {
    go func() {
        wg.Add(1)  // 💥 可能 Wait() 已经检查过计数器了
        process(i)
        wg.Done()
    }()
}
wg.Wait()

// ✅ 正确：Add 在 goroutine 外部调用
var wg sync.WaitGroup
for i := 0; i < n; i++ {
    wg.Add(1)
    go func() {
        process(i)
        wg.Done()
    }()
}
wg.Wait()
```

#### 陷阱三：复制 WaitGroup 导致未定义行为

```go
// ❌ 编译通过但运行出错
func handleRequests(reqs []Request) {
    var wg sync.WaitGroup
    for _, req := range reqs {
        handler := wg  // 💥 拷贝了 WaitGroup！
        go func(r Request) {
            handler.Add(1)
            serve(r)
            handler.Done()
        }(req)
    }
    wg.Wait()  // 💥 上面拷贝的 wg 和本地 wg 是两个独立的状态
}

// ✅ 始终传递指针或直接在原地调用
func handleRequests(reqs []Request) {
    var wg sync.WaitGroup
    for _, req := range reqs {
        wg.Add(1)
        go func(r Request) {
            defer wg.Done()
            serve(r)
        }(req)
    }
    wg.Wait()
}
```

#### 陷阱四：无缓冲 channel + WaitGroup 导致的 goroutine 泄漏

```go
// ❌ 泄漏风险：当某个 goroutine panic 且没有 recover
func worker(ids []int, resultCh chan Result) {
    var wg sync.WaitGroup
    for _, id := range ids {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            resultCh <- process(id)  // 如果 ch 已满且没人读，永久阻塞
        }(id)
    }
    wg.Wait()
    close(resultCh)
}
```

### 4. WaitGroup 的错误传播方案

WaitGroup 本身不支持错误传递，常用以下三种方案：

**方案 A：channel 收集错误**

```go
func runTasks(tasks []Task) error {
    var wg sync.WaitGroup
    errChan := make(chan error, len(tasks))
    
    for _, task := range tasks {
        wg.Add(1)
        go func(t Task) {
            defer wg.Done()
            if err := t.Execute(); err != nil {
                errChan <- fmt.Errorf("task %s failed: %w", t.Name(), err)
            }
        }(task)
    }
    
    wg.Wait()
    close(errChan)
    
    // 第一个错误就是我们要报告的错误
    for err := range errChan {
        return err
    }
    return nil
}
```

**方案 B：errgroup（推荐）**

```go
import "golang.org/x/sync/errgroup"

func runTasks(ctx context.Context, tasks []Task) error {
    g, ctx := errgroup.WithContext(ctx)
    
    for _, task := range tasks {
        task := task // 闭包变量捕获
        g.Go(func() error {
            return task.Execute()
        })
    }
    
    return g.Wait() // 第一个错误就返回，其他 goroutine 自动取消
}
```

**方案 C：AtomicError 原子保存最后一个错误**

```go
type AtomicError struct{ v atomic.Value }

func (a *AtomicError) Store(err error) {
    a.v.Store(err)
}

func (a *AtomicError) Load() error {
    err, _ := a.v.Load().(error)
    return err
}
```

### 5. WaitGroup 能复用吗？

**可以，但有前提**。复用前必须确认 Wait() 已经返回（所有 goroutine 已完成）：

```go
var wg sync.WaitGroup

// 第一轮
wg.Add(2)
go func() { defer wg.Done(); doWork1() }()
go func() { defer wg.Done(); doWork2() }()
wg.Wait()

// ✅ 可以复用：此时所有 goroutine 已结束
// 第二轮
wg.Add(3)
go func() { defer wg.Done(); doWork3() }()
go func() { defer wg.Done(); doWork4() }()
go func() { defer wg.Done(); doWork5() }()
wg.Wait()
```

---

## 🗣️ 面试话术

- **初级**："WaitGroup 用于等待一组 goroutine 完成，核心是 Add/Done/Wait 三件套。"
- **中级**："WaitGroup 基于 64bit 原子计数器，高32位存等待数、低32位存增量。关键坑是 Done 要用 defer、不能在 goroutine 内 Add、不要复制 WaitGroup。"
- **高级**："WaitGroup 底层用 runtime_Semacquire 做系统级睡眠而非忙等待。生产环境建议用 errgroup 替代，自带上下文取消和错误聚合能力。"

---

## 🔗 关联阅读

- [Channel 底层原理](./38-01-channel.md)
- [sync 原语：Mutex / RWMutex / Once / Pool](./40-02-sync.md)
- [单飞（SingleFlight）：请求合并去重](./13-06-singleflight.md)
- [并发模式：Pipeline / Fan-out/Fan-in](./02-04-patterns.md)
