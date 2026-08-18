# Go tool trace：运行时诊断与性能分析进阶

> 考察频率：★★★★☆  难度：★★★★★
> 关键词：go tool trace、runtime/trace、调度延迟、GC 暂停、syscall 阻塞、网络 IO 等待

## 面试官考察意图

pprof 是**静态快照**（采样某个时刻的 CPU/内存状态），而 `go tool trace` 是**时间序列记录**（完整记录程序运行期间的 GMP 调度、GC、syscall 等事件）。高级工程师必须会同时使用这两套工具来定位不同维度问题。

这道题考的是：**你能不能用 trace 还原一个完整的请求生命周期，找出延迟到底卡在了哪个环节。**

---

## 核心答案（30 秒版）

`go run -trace=trace.out main.go` 或 `GODEBUG=schedtrace=1` 可以生成 runtime trace 文件。用 `go tool trace trace.out` 打开 Web UI 后，你可以看到每个 goroutine 的完整生命周期、每次 GC 的精确时间线、syscalls 的阻塞时长、网络 IO 和锁竞争的时间分布。trace 的优势在于**能看到 pprof 看不到的事件**——比如为什么某个请求在 P99 上特别慢（可能卡在了一次 syscall 或一次 long GC pause）。

---

## Trace vs Profile：什么时候用什么？

| 维度 | pprof profile | go tool trace |
|------|--------------|---------------|
| **本质** | 统计快照（sampling） | 完整事件日志（event-based） |
| **开销** | 低（~5%） | 较高（~2x~5x，影响生产慎用） |
| **覆盖范围** | CPU/Memory/Goroutine/Block/Mutex | 全部 + GC 细节 + Syscall 详情 + Netpoll + Scheduler |
| **粒度** | 毫秒级 | 纳秒级（记录每个事件的时间戳） |
| **适用场景** | 找热点函数、内存泄漏 | 找调度延迟、GC 停顿、IO 阻塞、锁争用 |

> **简单规则：先用 pprof 快速定位方向，再用 trace 深入看细节。**

---

## 采集 Trace

### 方法 1：启动时指定 `-trace`

```bash
# 开发环境：直接输出
go run -trace=trace.out main.go

# 线上服务（谨慎）：通过 HTTP API 触发
curl "http://localhost:6060/debug/pprof/trace?seconds=10" > trace.out
```

### 方法 2：代码中主动触发

```go
import (
    "os"
    "runtime/trace"
)

func main() {
    f, err := os.Create("trace.out")
    if err != nil {
        panic(err)
    }
    defer f.Close()
    
    if err := trace.Start(f); err != nil {
        panic(err)
    }
    defer trace.Stop()
    
    // ... 业务逻辑
    
    trace.Stop() // 确保停止并写盘
}
```

### 方法 3：通过环境变量

```bash
# Go 1.21+ 可以通过 GOEXPERIMENT 或 flag 控制更细粒度
# 但最实用的还是上面的 HTTP 方式
```

---

## 解读 Trace Web UI

### 概览面板

```bash
go tool trace trace.out
```

启动后访问 `http://localhost:xxxx`，你会看到：

1. **Overview**：总览信息（持续时间、goroutine 数、GC 次数等）
2. **Scheduler Timeline**：所有 P 的调度时间线（横轴时间，纵轴 P）
3. **GC Timeline**：每次 GC 的开始/结束时间
4. **Syscall Latency**：系统调用耗时分布
5. **Net Poll Latency**：网络轮询事件
6. **Events**：自定义标记的事件

### Scheduler Timeline（最重要）

```
P0 ████████████░░██████████░░████████████░░
P1 ░░██████████████░░██████████░░██████████
P2 ██████░░████████████░░██████████░░██████

灰色 = 空闲（P 没有 runnable G）
白色 = 执行 G 代码
黑色块 = GC STW（Stop The World）

关键指标：
- 灰色区域占比高 → scheduler 调度有问题或资源不足
- 黑色块出现频繁 → GC 太勤快
```

### GC Timeline

```
GC #1   [████] ← 0.2ms
         ↑       ↑
      start     end
      
GC #2   [██████████] ← 0.8ms  ← 这次太长！需要排查
         
GC #3   [██] ← 0.1ms

点击某个 GC 事件可以看到详细信息：
- mark phase 持续了多久
- sweep 持续了多久
- 扫描了多少 bytes
- STW 阶段占了多长
```

### Syscall 分析

```
每个 syscall 事件记录：
- syscall name（如 epoll_wait, read, write）
- duration
- blocked-on reason（如果因为 syscall 阻塞）

高频 syscall：
- epoll_wait → 网络连接太多或 poller 有问题
- mmap/munmap → 大内存分配
- clock_gettime → 高频 time.Now() 调用
```

### 自定义 Event 标记

```go
import "runtime/trace"

func processOrder(ctx context.Context, orderID string) error {
    ctx, task := trace.NewTask(ctx, "processOrder")
    defer task.End()
    
    // 添加详细标签
    trace.Log(ctx, "order.id", orderID)
    trace.Label(ctx, "user.tier", "premium")
    
    // 标记一段耗时操作
    trace.WithRegion(ctx, "query_inventory", func() {
        inventory, err := db.QueryInventory(orderID)
        if err != nil {
            trace.Fatalf(ctx, "inventory query failed: %v", err)
        }
        _ = inventory
    })
    
    return nil
}
```

在 trace UI 中可以用 filter 搜索特定 tag 的事件。

---

## 实战场景

### 场景 1：偶发高延迟根因

```
症状：某接口平均 50ms，TP99 偶尔跳到 500ms+

分析步骤：

1. pprof CPU profile → 发现 handleRequest 耗时最多
2. 但 CPU 没打满，说明不是计算密集
   
3. go tool trace → 查看 Scheduler Timeline
   发现在几次 TP99 spike 时，有黑色大块（GC STW）
   
4. 放大到 GC Timeline → 确认有 200ms+ 的 GC pause
   
5. 结论：GC 压力过大导致偶发的长 STW → 
   优化方案：减少对象分配（sync.Pool）、调 GOGC
```

### 场景 2：Goroutine 看起来"卡住了"

```
症状：某个 worker goroutine 似乎不处理新任务了

分析步骤：

1. pprof goroutine profile → 看到 goroutine 状态
2. go tool trace → 查看 Scheduler Timeline
   
   在 timeline 中追踪该 G 的活动：
   - 最后执行了什么代码
   - 最后一次被调度是什么时候
   - 是否卡在 syscall（如 netconn.Read）
   - 是否被 mutex 挡住（mutex profile）
   
3. 通常结果是：卡在某个 network read 或 channel receive
```

### 场景 3：CPU 利用率不均

```
症状：多核服务器上只有部分核满载，其他空闲

分析步骤：

1. go tool trace → Scheduler Timeline
   观察多个 P 的活动模式
   
2. 常见原因：
   - 大量长时间 syscall → P 被 blocking syscall 占用
   - 全局锁竞争 → 某些 P 一直在等锁
   - goroutine 数量 < P 数量 → P 闲置
   
3. 修复：减少 syscall 次数、优化锁粒度、调整 GOMAXPROCS
```

---

## Trace 的高级用法

### 过滤与搜索

```
在 trace UI 中：
- 顶部输入框：按 goroutine ID / function name / region name 搜索
- 时间轴缩放：鼠标滚轮 / 拖拽选择时间段
- 导出 PNG/SVG：用于报告分享
```

### 对比两次 Trace

```bash
# 比较不同版本之间的行为变化
# 方法 1：分别跑 trace 然后手动对比
go tool trace trace_v1.out
go tool trace trace_v2.out

# 方法 2：在 trace 页面中使用 Filter
# 按 Goroutine ID 筛选同一条请求的前后对比
```

### Trace + Metrics 联动

```go
// 在 Prometheus metrics 中标记 trace 相关的指标
var gcPauseDuration = promauto.NewHistogramVec(
    prometheus.HistogramOpts{
        Name: "golang_gc_pause_seconds",
        Help: "GC pause durations",
    },
    []string{"phase"}, // mark, mark termination, sweep
)
```

---

## 高频追问

**Q：trace 会影响生产性能吗？**
A：会。全量 trace 大约 2x~5x 的性能开销。生产环境中不建议长期开启。推荐：仅在 debug 模式下采集短时间（几秒到几十秒），或通过 HTTP 按需触发。

**Q：trace 能替代 pprof 吗？**
A：不能互补。pprof 适合快速定位"哪里慢"，trace 适合深度分析"为什么慢"。典型的流程是：pprof 发现瓶颈点 → trace 验证猜测 → 找到真正原因。

**Q：Trace 中看到的 GC pause 和 pprof 里有什么不同？**
A：pprof 的 CPU profile 只能间接反映 GC（GC 期间也有样本被记录），但看不到精确的开始/结束时间和各阶段的耗时。Trace 直接记录每次 GC 的完整时间线，包括 STW、mark、sweep 的详细分段。

**Q：Go 1.21+ 有哪些 trace 改进？**
A：Go 1.21 增加了更多可配置的 trace event 和更好的 Web UI。Go 1.22 改善了 trace 文件的压缩和加载速度。Go 1.23 新增了更细粒度的 scheduler 事件。建议关注官方 release notes。

---

## 延伸阅读

- [Go Blog: Using the Go Tooltrace](https://go.dev/doc/tutorial/intro-to-go-trace)
- [Go Runtime Trace Documentation](https://pkg.go.dev/runtime/trace)
- [Uber: Tracing in Production Go](https://eng.uber.com/tracing-in-production/)
