[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚡ 性能调优](../README.md)

---

# 内存泄漏排查：goroutine 泄漏、map 泄漏、timer 泄漏、sync.Pool 陷阱

## 面试官考察意图

考察候选人对 Go 内存泄漏问题的排查能力。初级只会说"看看 pprof"，高级要能讲清楚 **goroutine 泄漏的根因（channel 阻塞、for-range 误用）、map 持续写入导致过期数据累积、timer vs ticker 误用、sync.Pool GC 清空导致的无效分配**，并能用 pprof + trace 组合拳定位根因。

---

## 核心答案（30 秒版）

Go 内存泄漏的四大根因：

| 类型 | 根因 | 典型场景 | 排查工具 |
|------|------|----------|----------|
| **goroutine 泄漏** | channel 发送无接收、无缓冲 channel 发送阻塞 | for-range 误用、context 未取消 | `pprof goroutine`、`trace` |
| **map 泄漏** | 持续写入，从不删除 | 缓存 map 无限增长 | `pprof heap` |
| **timer/ticker 泄漏** | timer 用作 ticker、stop 后未清理 | setTimeout 误用 | `pprof goroutine` |
| **sync.Pool 泄漏** | GC 清空后大量重新分配 | 高频对象池化 | `pprof goroutine` |

**内存泄漏 ≠ OOM**：Go 的 RSS 是渐进增长的，GC 会清理垃圾，但泄漏对象如果被根引用，GC 无法回收，最终 OOM。

---

## 深度展开

### 1. goroutine 泄漏（最常见）

#### 泄漏模式一：channel 无接收者

```go
// ❌ 泄漏：goroutine 阻塞在 channel 发送
func leak1() {
    ch := make(chan int)
    go func() {
        ch <- 1  // 永远没人接收，goroutine 阻塞在此
    }()
    // 函数返回，goroutine 仍在运行
}

// ✅ 修复：使用带缓冲的 channel
func fixed1() {
    ch := make(chan int, 1)  // 缓冲至少 1 个
    go func() {
        ch <- 1  // 发送后立即返回
    }()
}

// ✅ 修复：使用 context 取消
func fixed2(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case ch <- 1:
        case <-ctx.Done():
            return  // context 取消时退出
        }
    }()
}
```

#### 泄漏模式二：for-range 误用

```go
// ❌ 泄漏：for-range 遍历 channel 时，如果 channel 不关闭
//        遍历会一直等待，永不返回
func leak2() {
    ch := genChannel()  // 返回一个不会关闭的 channel
    go func() {
        for v := range ch {  // 如果 ch 永不关闭，这里永远阻塞
            fmt.Println(v)
        }
    }()
}

// ✅ 修复：使用 context 超时
func fixed3(ctx context.Context) {
    ch := genChannel()
    go func() {
        for {
            select {
            case v, ok := <-ch:
                if !ok {
                    return  // channel 关闭
                }
                fmt.Println(v)
            case <-ctx.Done():
                return  // context 超期
            }
        }
    }()
}
```

#### 泄漏模式三：无缓冲 channel 发送阻塞

```go
// ❌ 泄漏：两个 goroutine 互相等待对方的 channel
//        造成死锁（goroutine 泄漏的特殊形式）
func leak3() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go func() {
        <-ch2  // 等待 ch2
        ch1 <- 1
    }()

    go func() {
        <-ch1  // 等待 ch1
        ch2 <- 2
    }()

    // 两个 goroutine 互相等待，永不返回
}
```

#### 泄漏模式四：HTTP 服务中的 context 泄漏

```go
// ❌ 泄漏：请求处理函数启动后台 goroutine，但请求结束后
//        goroutine 仍在运行（因为 channel 还在等待）
func leakHandler(w http.ResponseWriter, r *http.Request) {
    ch := make(chan int)  // 局部 channel，函数返回后无引用

    go func() {
        result := process()  // process() 可能很慢
        ch <- result         // 如果没人接收，这里永久阻塞
    }()

    // HTTP 请求很快返回，但 goroutine 泄漏了
    // 这个 ch 在函数返回后无引用（理论上 GC 应该回收）
    // 但 process() 本身可能在泄漏
}
```

### 2. map 泄漏

**Go map 不收缩**：即使删除元素，map 的 capacity 也不会降低。

```go
// ❌ 泄漏：缓存 map 持续增长，永不清理
type Cache struct {
    m map[string][]byte
}

func (c *Cache) Get(key string) ([]byte, bool) {
    v, ok := c.m[key]
    return v, ok
}

func (c *Cache) Set(key string, value []byte) {
    c.m[key] = value  // map 只增不减，内存持续增长
}
```

**解决方案：TTL + 定期清理**

```go
// ✅ 方案1：定期重建（最常用）
type TTLCache struct {
    mu    sync.RWMutex
    data  map[string][]byte
    maxAge time.Duration
}

func (c *TTLCache) Get(key string) ([]byte, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

func (c *TTLCache) Set(key string, value []byte) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}

// 定时重建 cache（每次重建开销 O(n)，但避免内存持续增长）
func (c *TTLCache) StartGC(interval time.Duration) {
    ticker := time.NewTicker(interval)
    for range ticker.C {
        c.mu.Lock()
        c.data = make(map[string][]byte)  // 旧 map 无引用，GC 回收
        c.mu.Unlock()
    }
}

// ✅ 方案2：按需清理（适合缓存量大但写入频率不高的场景）
func (c *TTLCache) cleanup(maxLen int) {
    if len(c.data) < maxLen {
        return
    }
    c.mu.Lock()
    defer c.mu.Unlock()

    // 删除最老的 20% 数据
    keys := make([]string, 0, len(c.data))
    for k := range c.data {
        keys = append(keys, k)
    }
    sort.Strings(keys)
    for i := 0; i < len(keys)/5; i++ {
        delete(c.data, keys[i])
    }
}
```

### 3. timer / ticker 泄漏

**timer 只触发一次，ticker 持续触发。误用 timer 做循环定时 = 泄漏。**

```go
// ❌ 泄漏：用 timer 做轮询（timer 只触发一次！）
func leakTimer() {
    for {
        result := fetchData()
        process(result)
        time.Sleep(time.Hour)  // 这个 timer 从不停止，不断创建新的 goroutine？
        // 实际上 time.Sleep 不是 goroutine，但定时任务有类似问题：
    }
}

// ❌ 泄漏：用 timer 代替 ticker（timer 只触发一次！）
func leakTimer2() {
    ch := make(chan int)
    go func() {
        timer := time.NewTimer(time.Second)  // ❌ 只触发一次！
        for {
            select {
            case <-timer.C:
                // 永远不会第二次执行
            }
        }
    }()
}

// ✅ 正确：用 ticker 做轮询
func correctTicker() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()  // 必须 Stop，否则 ticker 泄漏

    for {
        <-ticker.C
        // 每秒执行一次
    }
}
```

### 4. sync.Pool 泄漏

**sync.Pool 在 GC 时会被清空**（设计如此），清空后下一轮请求全部触发新分配。

```go
// ❌ 泄漏场景：Pool 容量不设限制，高并发下 pool 快速膨胀
var pool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)  // 每次 GC 后重新分配
    },
}

// ✅ 正确：限制 Pool 大小，或者接受 GC 清空
// 实际上 sync.Pool 的设计就是"不保证"复用
// 它的存在是为了减少 GC 后的瞬间分配压力
// 如果需要可靠的缓存，用 sync.Map 或自定义 LRUCache
```

### 5. 排查工具：pprof + trace 组合拳

**Step 1：pprof goroutine profile（快速定位 goroutine 数量异常）**

```bash
# 1. 访问 pprof 端点
curl http://localhost:6060/debug/pprof/goroutine > goroutine.prof

# 2. 分析
go tool pprof goroutine.prof

# 3. 在 pprof 中
(pprof) top    # 看 goroutine 数量排名
(pprof) traces # 看 goroutine 的调用栈
```

```bash
# 输出示例
Dropping nodes for which only hidden frames exist; use 'explicit' to show them
Active filters:
Showing nodes accounting for 512, 5.15% of 9946 total
      flat  flat%   sum%     cum   cum%
       120  1.21%  34.49%     120  1.21%  runtime.gopark
        99  0.99%  63.01%      99  0.99%  runtime.chansend
        80  0.80%  86.05%      80  0.80%  time.Sleep
        ...
# 120 个 goroutine 阻塞在 chansend → channel 发送阻塞（泄漏的典型特征）
```

**Step 2：pprof heap（定位内存分配来源）**

```bash
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof -inuse_space heap.prof  # 查看活跃内存
go tool pprof -alloc_space heap.prof  # 查看累计分配
```

**Step 3：trace（精确定位 goroutine 创建时间点）**

```go
// 在代码中启用 trace
import "runtime/trace"

func handler(w http.ResponseWriter, r *http.Request) {
    trace.StartRegion(r.Context(), "request")
    defer trace.End()

    // 业务逻辑
    // trace 会记录每个 goroutine 的创建和结束时间
}

// 生成 trace 文件
curl http://localhost:6060/debug/pprof/trace?seconds=30 > trace.out

// 查看 trace
go tool trace trace.out
```

**Step 4：生产环境自动告警**

```go
// 基于 pprof 的自动告警
func startLeakDetector(addr string, threshold int) {
    go func() {
        for {
            time.Sleep(5 * time.Minute)

            // 获取 goroutine 数量
            resp, _ := http.Get(fmt.Sprintf("http://%s/debug/pprof/goroutine?debug=1", addr))
            defer resp.Body.Close()

            buf, _ := io.ReadAll(resp.Body)
            count := strings.Count(string(buf), "goroutine")

            if count > threshold {
                alert("Goroutine 数量异常: %d > %d", count, threshold)
                // 保存 pprof 文件供事后分析
                savePprof(addr, "goroutine")
                savePprof(addr, "heap")
            }
        }
    }()
}
```

### 6. 真实案例：线上 OOM 排查

```
故障现象：服务内存从 500MB 增长到 8GB，最终 OOM

排查过程：
1. pprof goroutine → 发现 5 万个 goroutine 阻塞在 chansend
2. traces → 定位到 for-range channel 的消费 goroutine 全部阻塞
3. 根因：上游服务超时，导致 channel 无数据，消费者全在等
4. 修复：添加 context 超时取消，消费者在 30s 内全部退出
5. 预防：增加 goroutine 数量告警（> 5000 立即告警）
```

---

## 高频追问

**Q：goroutine 和 thread 有什么区别？为什么 goroutine 泄漏更隐蔽？**

> goroutine 是用户态调度（协程），创建成本极低（~2KB），而线程是内核态调度（~2MB）。goroutine 泄漏更隐蔽，因为每个泄漏的 goroutine 内存占用小，但数千个泄漏 goroutine 会消耗大量内存（栈空间累积）和调度开销（调度器负担加重）。线程泄漏会导致系统资源迅速耗尽，容易发现；goroutine 泄漏是渐进式的。

**Q：Go 的 GC 会扫描 goroutine 的栈吗？**

> 会。GC 的 mark 阶段会扫描所有 goroutine 的栈，找出栈上的指针。goroutine 泄漏意味着它的栈上有引用（channel、map、全局变量等），GC 无法回收这些对象。stop-the-world（STW）期间所有 goroutine 会被停止，GC 会扫描它们的栈。

**Q：如何设计一个不会泄漏的定时任务？**

```go
// 正确：所有资源显式释放
func startWorker(ctx context.Context) {
    ticker := time.NewTicker(time.Minute)
    defer ticker.Stop()  // 显式停止 ticker

    for {
        select {
        case <-ticker.C:
            doWork()
        case <-ctx.Done():
            return  // context 取消时退出（不是 break，是 return）
        }
    }
}

// 使用：启动时传入 context，shutdown 时取消
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
startWorker(ctx)
```

---

## 延伸阅读

- [Go pprof 官方文档](https://github.com/google/pprof/blob/main/doc/README.md)
- [runtime/trace 官方文档](https://pkg.go.dev/runtime/trace)
- [Uber Go 编码规范 - Goroutine](https://github.com/uber-go/guide/blob/master/src/content/style/style.md#goroutine-lifecycle)
