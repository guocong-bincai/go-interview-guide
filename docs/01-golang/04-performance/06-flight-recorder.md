# Go 1.25 Flight Recorder：生产环境链路追踪利器

## 面试官考察意图

考察候选人对 Go 最新诊断工具的掌握程度。
初级只知道 `pprof`，高级要能讲清楚 **Flight Recorder 的设计动机（解决生产环境「事后追查」问题）、与 pprof/execution trace 的区别、以及如何在长服务中主动捕获故障时间窗口**，体现对生产排障的系统性理解。

---

## 核心答案（30 秒版）

Go 1.25 引入了 **Flight Recorder（飞行记录器）**，将 `runtime/trace` 的执行轨迹以环形缓冲区方式常驻内存。程序检测到故障时（如超时、panic），直接读取缓冲区即可拿到**故障前几秒的完整 goroutine 调度、网络、GC 事件**，无需复现问题。

```go
// 开启飞行记录（收集最近 10 秒的 trace）
runtime/trace.StartFlightRecording(runtime/trace.FlightRecordingOptions{
    Mode:    runtime/trace.Embedding,
    Bufsize: 10 << 20, // 10MB
})

// 故障时导出
f, _ := os.Create("trace.rec")
runtime/trace.Read(f) // 读出的是最近 10 秒
```

---

## 深度展开

### 1. 为什么需要 Flight Recorder？

**传统 tracing 的困境：**

| 方式 | 问题 |
|------|------|
| `runtime/trace.Start/Stop` | 只能追踪**调用期间**的数据，短服务测试可以，生产长服务不适用 |
| 随机采样全量 trace | 需要大量基础设施，存储/处理成本高，且大多数数据没有有用信息 |
| 复现问题后再 trace | 偶发问题难以复现，等发现问题再去 Start 已经晚了 |

**Flight Recorder 的核心思想：** 程序总是比人更早知道出问题了。Flight Recorder 让程序在检测到故障的瞬间，主动保留「是谁在作妖」的关键证据。

```
时间线：
──────────────────────────────────────────────────────→
        [故障发生，程序检测到] ← 需要抓这个时刻
                        ↑
              Flight Recorder 环形缓冲区
                        ↑
        [持续写入最近 N 秒的 trace 数据]
```

### 2. 与 pprof 的区别

| | pprof | Execution Trace | Flight Recorder |
|---|---|---|---|
| **数据** | 采样统计数据（CPU/内存） | 精确事件序列 | 精确事件序列 |
| **触发方式** | 手动采样 | 固定时间窗口 | 任意时刻（事后读取） |
| **适用场景** | CPU/内存热点 | 单次请求延迟分析 | 偶发问题事后调查 |
| **生产使用** | 需控制采样率 | 不适合全量开启 | ✅ 适合常驻 |

**互补关系：** pprof 告诉你「哪个函数耗资源多」，Flight Recorder 告诉你「在故障那一刻，goroutine 们在做什么」。

### 3. 核心 API

```go
import "runtime/trace"

// 1. 开启飞行记录
// StartFlightRecording 替代 StartTrace，返回一个 io.Closer
rec, err := trace.StartFlightRecording(trace.FlightRecordingOptions{
    Mode:    trace.Embedding,         // 将 trace 数据嵌入程序
    Bufsize: 50 << 20,                // 缓冲区大小（默认 5MB）
    File:    "trace.rec",             // 可选：同时写入文件
})

// 2. 故障时读取缓冲区
func handlePanic() {
    f, err := os.Create("incident.trace")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()
    
    // Read 读取的是环形缓冲区的内容（最近 Bufsize 的数据）
    if err := trace.Read(f); err != nil {
        log.Fatal(err)
    }
    
    log.Printf("已导出故障 trace：%s", f.Name())
}

// 3. 停止记录
rec.Close()
```

### 4. 实际使用模式

#### 模式一：与 health check 集成

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    // 检查关键指标
    if isDegraded() {
        // 导出当前 trace
        exportTrace("health-check-degraded")
        http.Error(w, "degraded", 503)
        return
    }
    w.WriteHeader(200)
})

// isDegraded 检测到问题时，Flight Recorder 已经准备好「前几秒的数据」
```

#### 模式二：与 panic recovery 集成

```go
func serve() {
    defer func() {
        if r := recover(); r != nil {
            exportTrace("panic-recovery")
            log.Printf("panic: %v", r)
        }
    }()
    // 业务逻辑
}
```

#### 模式三：定时健康报告

```go
// goroutine：定期检查是否需要导出 trace
go func() {
    ticker := time.NewTicker(5 * time.Minute)
    for range ticker.C {
        if healthScore() < threshold {
            exportTrace("low-health-score")
        }
    }
}()
```

### 5. 查看 Flight Recorder trace

```bash
# 1. 导出 trace（程序内写入文件）
# 2. 用 go tool trace 查看
go tool trace trace.rec

# 在浏览器中你会看到：
# - Goroutine 分析（哪个 goroutine 在哪个时间在做什么）
# - 调度器事件（抢占、阻塞、唤醒）
# - GC 事件（标记开始/结束、GC 耗时占比）
# - 网络阻塞事件
# - 系统调用事件
```

**关键视图：**

- **User Annotations**：`trace.WithRegion`/`trace.Log` 标记的业务事件，帮助定位具体是哪行业务代码慢
- **Goroutine Analysis**：按状态（运行/等待/阻塞）分组，直观看 goroutine 泄漏
- **Network**：网络 IO 和调度关系，看是否因为网络 IO 导致协程阻塞
- **Synchronization**：锁竞争与同步原语事件

### 6. 生产环境使用注意事项

| 注意事项 | 说明 |
|----------|------|
| **内存开销** | 1MB 缓冲区 ≈ 可记录 1 秒的高负载 trace。默认 5MB，通常增加 ~10MB 内存 |
| **CPU 开销** | 开启时约增加 2~5% CPU 开销（Go 1.25 优化后更低） |
| **文件存储** | 建议按日期/事件类型命名，如 `trace-20260415-p99-timeout.rec` |
| **安全** | trace 可能包含敏感数据（请求参数），确保导出的文件权限正确 |
| **不要全量开启** | Flight Recorder 是常驻的，pprof 按需采样，两者结合使用 |

### 7. Go 1.26 中的 Flight Recorder

Go 1.26 对 Flight Recorder 做了进一步优化：
- 改善了大容量缓冲区（>50MB）的写入性能
- 新增 `trace.Recording` 接口，可查询当前缓冲区状态
- 支持与 `go tool trace` 的流式对接（无需先导出文件）

---

## 高频追问

**Q：Flight Recorder 和 dlv debugger 有什么区别？**
> dlv 是逐行调试器，用于开发/测试环境。Flight Recorder 是生产环境工具，不需要暂停程序，用环形缓冲区捕获历史数据。两者互补。

**Q：Flight Recorder 能替代 pprof 吗？**
> 不能。pprof 是采样统计数据（告诉我「哪些函数占用最多 CPU/内存」），Flight Recorder 是事件序列（告诉我「在故障那一刻 goroutine 们在做什么」）。生产排障通常先用 pprof 定位热点函数，再用 Flight Recorder 还原故障现场。

**Q：环形缓冲区满了会怎样？**
> 环形缓冲区自动覆盖最旧的数据。这意味着 Flight Recorder 总是保留「最近 N 秒」的数据，适合捕获偶发问题。如果想保留特定事件，可以结合 `trace.WithRegion` 标记重要区间。

---

## 延伸阅读

- [Go 1.25 Flight Recorder 官方博客](https://go.dev/blog/flight-recorder)
- [runtime/trace 官方文档](https://pkg.go.dev/runtime/trace)
- [Go Execution Traces 2024](https://go.dev/blog/execution-traces-2024)
- [go tool trace 使用指南](https://go.dev/blog/execution-traces-2024)
