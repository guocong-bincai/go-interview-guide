[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚡ 性能调优](../README.md)

---

# runtime/metrics：生产级指标采集与 ReadMemStats 的 STW 陷阱

> 考察频率：★★★★☆  优先级：P0（线上服务必备）
> 关键词：runtime/metrics、ReadMemStats STW、Sample、直方图、Prometheus 暴露

---

## 面试官考察意图

考察候选人**是否真的在生产里采过 Go 运行时指标**。
初级只会 `runtime.ReadMemStats` 打日志；高级要能说清楚：**`ReadMemStats` 会 STW、为什么它不能高频调用、`runtime/metrics` 的稳定命名与直方图能力、以及如何把指标暴露给 Prometheus 并且控制基数**。

---

## 核心答案（30 秒版）

| 维度 | `runtime.ReadMemStats` | `runtime/metrics`（Go 1.16+）|
|------|------------------------|------------------------------|
| STW | **会短暂 STW（暂停所有 goroutine）** | 无 STW |
| 指标范围 | 只有内存相关 | GC / 调度 / 内存 / 锁 / 协程 / cgo 全覆盖 |
| 命名稳定性 | `MemStats` 字段，长期稳定但粗粒度 | `/gc/heap/allocs:bytes` 形式，**有版本承诺** |
| 直方图 | 无 | 支持 `Float64Histogram`（延迟分布）|
| 推荐频率 | 分钟级、低频 | 秒级、高频无压力 |

**结论：新代码一律用 `runtime/metrics`；`ReadMemStats` 只作为兜底或调试用。**

---

## 深度展开

### 1. 为什么 `ReadMemStats` 不能高频调用

`ReadMemStats` 为了保证数据一致，需要**停止世界（STW）**读取各 P 的 mcache、mcentral 统计：

```go
// 伪代码：ReadMemStats 内部的代价
// 1. stopTheWorld("read mem stats")
// 2. 遍历所有 P 的 mcache、所有 mspan 统计
// 3. startTheWorld()
var m runtime.MemStats
runtime.ReadMemStats(&m) // ← 每次调用都可能带来微秒~毫秒级停顿
```

在 QPS 几万的网关里，把 `ReadMemStats` 放进每个请求或每 100ms 的 goroutine 里，
会直接表现为**周期性 P99 毛刺**。反面教材：

```go
// ❌ 反模式：高频 ReadMemStats
go func() {
    for range time.Tick(50 * time.Millisecond) {
        var m runtime.MemStats
        runtime.ReadMemStats(&m) // 每 50ms 触发一次 STW
        log.Printf("heap=%dMB", m.HeapAlloc/1024/1024)
    }
}()
```

### 2. 正确姿势：`runtime/metrics`

```go
import (
    "runtime/metrics"
    "time"
)

// 1) 启动时把关心的指标"订阅"下来（避免每次 All()）
var samples []metrics.Sample
var descs = []string{
    "/gc/heap/allocs:bytes",          // 累计分配字节
    "/gc/heap/objects:objects",       // 堆对象数
    "/gc/cycles/automatic:gc-cycles", // 自动 GC 次数
    "/gc/pauses:seconds",             // GC 停顿直方图
    "/sched/goroutines:goroutines",   // 当前协程数
    "/sched/latencies:seconds",       // 调度延迟直方图
    "/sync/mutex/wait/total:seconds", // 锁等待总时长
    "/memory/classes/total:bytes",
}

func init() {
    samples = make([]metrics.Sample, len(descs))
    for i, d := range descs {
        samples[i].Name = d
    }
}

func collect() {
    metrics.Read(samples) // 无 STW，可按秒级调用
    for _, s := range samples {
        switch s.Value.Kind() {
        case metrics.KindUint64:
            report(s.Name, float64(s.Value.Uint64()))
        case metrics.KindFloat64:
            report(s.Name, s.Value.Float64())
        case metrics.KindFloat64Histogram:
            h := s.Value.Float64Histogram()
            report(s.Name+"_p99", percentile(h, 0.99))
        case metrics.KindBad:
            // 指标在当前 Go 版本不存在（名字写错或版本差异）
        }
    }
}
```

要点：

- **`metrics.All()` 只在启动时调一次**：它返回所有指标描述，本身开销不小；
- **`Sample.Name` 写错不会报错**，`Read` 后 `Kind() == KindBad`，必须显式判断；
- **累计型指标（如 `allocs:bytes`）要用 `rate()` 思路**：Prometheus 端用 `rate()`/`increase()`，不要直接看绝对值；
- **直方图指标（pauses、latencies）** 是定位长尾延迟的关键，`_p99` 比平均值有用得多。

### 3. 关键指标速查表

| 指标 | 含义 | 典型用法 |
|------|------|---------|
| `/sched/goroutines:goroutines` | 当前 goroutine 数 | 泄漏排查（单调上涨 = 泄漏）|
| `/gc/heap/allocs:bytes` | 累计分配量 | 分配速率，定位热点分配 |
| `/gc/heap/live:bytes` | 活跃堆大小 | 真实内存占用 |
| `/gc/cycles/automatic:gc-cycles` | 自动 GC 次数 | GC 频率是否异常 |
| `/gc/pauses:seconds` | GC 停顿分布 | P99 毛刺归因 |
| `/sync/mutex/wait/total:seconds` | 锁等待总时长 | 锁竞争定位 |
| `/memory/classes/heap/objects:bytes` | 堆对象占用 | 内存泄漏趋势 |

### 4. 与 Prometheus / expvar / pprof 的关系

```
┌─────────────── 采集层 ───────────────┐
runtime/metrics ──┐
                  ├─→ prometheus/client_golang（新版内部已用 metrics）
expvar ───────────┘
pprof（按需拉取，采样分析）
```

- **pprof 是"按需诊断"，metrics 是"持续监控"**，两者互补，不要用 pprof 做监控；
- `expvar` 更多用于业务自定义变量，标准库运行时指标已不推荐；
- `prometheus/client_golang` 从 v1.13+ 起内置 collector 走 `runtime/metrics`，所以业务侧只需要加业务指标。

### 5. 生产坑点

```go
// ❌ 坑：把 metrics 当业务指标用，Name 拼错后静默失效
samples[i].Name = "/gc/heap/allocs"    // 少了 ":bytes" → KindBad，永远采不到

// ✅ 正确：启动时校验，错的名字直接 fail fast
for i := range samples {
    if samples[i].Value.Kind() == metrics.KindBad {
        log.Fatalf("unknown metric: %s", samples[i].Name)
    }
}
```

- **指标名必须带单位后缀**（`:bytes` / `:seconds` / `:goroutines`），这是稳定性承诺的一部分；
- **不要每请求 `metrics.All()`**，也不要在热路径构造 `Sample` 切片；
- **高基数陷阱在业务侧**：不要用 user_id、URL 全路径做 label，运行时指标本身基数固定。

---

## 高频追问

**Q：Go 1.24+ 的 `runtime/metrics` 有什么变化？**

Green Tea GC（Go 1.26）增加了新的 GC 相关指标，部分 `godebug` 行为也可通过
`/godebug/non-default-behavior/<name>:events` 观测——这对排查"某个 GODEBUG 开关被谁改了"很有用。

**Q：如何在 K8s 里用它排查 OOMKilled？**

组合看：`/memory/classes/heap/objects:bytes` 是否上涨（Go 堆）；
`/memory/classes/total:bytes` 与容器 limit 对比；
配合 `GOMEMLIMIT` 生效情况。区分"Go 堆大"还是"Go 之外的 mmap/线程栈/CGO 占用"。

**Q：`ReadMemStats` 在 Go 1.2x 有没有优化？**

它仍然是 STW 语义，官方推荐迁移到 `runtime/metrics`；旧字段（如 `PauseNs`）不会新增能力。

---

## 面试话术

> "线上采运行时指标应该用 `runtime/metrics`，`Read` 没有 STW，秒级采集无压力；
> `runtime.ReadMemStats` 会 STW，高频调用就是给自己制造 P99 毛刺。
> 采样时要先订阅名字、启动时校验 KindBad，累计量交给 Prometheus 做 rate，
> 重点关注 goroutines、GC pauses p99、mutex wait 这几个。"

---

## 延伸阅读

- [runtime/metrics 文档](https://pkg.go.dev/runtime/metrics)
- [Go 官方博客：Metrics 设计](https://go.dev/blog/runtime-metrics)
- [Prometheus client_golang metrics collector](https://pkg.go.dev/github.com/prometheus/client_golang)

---

**[← 上一篇：Blocking Profile](./80-blocking-profile-analysis.md)** · **[返回模块首页 →](../README.md)**
