# 性能指标体系

> 考察频率：★★★★☆  难度：★★★☆☆

## 核心性能指标

### 延迟指标

| 指标 | 定义 | 含义 |
|------|------|------|
| **P50 (Median)** | 50% 请求的延迟 | 中位数，普通用户体验 |
| **P90** | 90% 请求的延迟 | 头部用户体验 |
| **P95** | 95% 请求的延迟 | 大多数 SLA 基准 |
| **P99** | 99% 请求的延迟 | 极端用户体验，通常是优化重点 |
| **P999** | 99.9% 请求的延迟 | 极端案例，高可用要求 |

```go
// Go 计算延迟分位数
func percentile(latencies []time.Duration, p float64) time.Duration {
    sort.Slice(latencies, func(i, j int) bool {
        return latencies[i] < latencies[j]
    })
    idx := int(float64(len(latencies)) * p)
    if idx >= len(latencies) {
        idx = len(latencies) - 1
    }
    return latencies[idx]
}

// 使用示例
latencies := []time.Duration{1, 2, 3, 4, 5, 6, 7, 8, 9, 10} // ms
fmt.Printf("P50: %v\n", percentile(latencies, 0.50)) // 5ms
fmt.Printf("P99: %v\n", percentile(latencies, 0.99)) // 10ms
```

### P50 / P95 / P99 的区别

```
假设 10000 个请求，响应时间分布如下：

P50 = 100ms    → 5000 个请求在 100ms 内完成
P95 = 500ms    → 9500 个请求在 500ms 内完成
P99 = 2000ms   → 9900 个请求在 2000ms 内完成

重点：P99 = 2000ms 意味着有 100 个用户等了 2 秒！
```

### 为什么 P99 比 P95 更重要？

```
P95：5% 用户受影响
P99：1% 用户受影响，但绝对数量更大

对于 1 万 QPS 的系统：
- P95 超标：500 QPS 用户受影响
- P99 超标：100 QPS 用户受影响

但 P99 超标通常意味着：
1. 系统有潜在的极端 case
2. 可能接近系统瓶颈
3. 容易引发雪崩
```

---

## 可用性指标

### SLA（Service Level Agreement）

| SLA | 日可用时间 | 年故障时间 |
|-----|----------|-----------|
| 99% | 99% | 3.65 天 |
| 99.9% | 99.9% | 8.76 小时 |
| 99.99% | 99.99% | 52.6 分钟 |
| 99.999% | 99.999% | 5.26 分钟 |

### 可用性计算

```go
// 单一服务可用性
availability := uptime / (uptime + downtime) * 100

// 多服务串联（所有服务都可用才算可用）
// A系统 99.9% × B系统 99.9% × C系统 99.9% = 99.7%
// 99.9%^3 = 99.7%

// 多服务并联（任一服务可用即可用）
// A系统 99.9% + B系统 99.9% - 重叠部分
```

### 高可用设计原则

```go
// 1. 冗余：每层至少 2 台机器
// 2. 隔离：不同服务分开，避免单点故障
// 3. 超时：避免请求无限等待
// 4. 熔断：快速失败，防止雪崩
// 5. 重试：幂等重试，但要有上限

// 超时配置示例
client := &http.Client{
    Timeout: 3 * time.Second,  // 3 秒超时
}

// 熔断配置
cb := &CircuitBreaker{
    failureThreshold: 5,      // 5 次失败触发
    timeout:          30 * time.Second,  // 30 秒后尝试恢复
}
```

---

## 吞吐指标

| 指标 | 定义 | 监控意义 |
|------|------|---------|
| **QPS** | 每秒请求数 | 系统负载 |
| **TPS** | 每秒事务数 | 业务处理能力 |
| **并发数** | 同时处理的请求 | 资源占用 |
| **响应时间** | 请求到响应的耗时 | 用户体验 |

### QPS 和并发数的关系

```go
// 并发数 = QPS × 平均响应时间
// QPS = 1000, 平均响应时间 = 100ms
// 并发数 = 1000 × 0.1 = 100

// 反推：
// 最大并发 = 1000，需要多少 QPS？
// QPS = 并发数 / 响应时间 = 1000 / 0.1 = 10000 QPS

// 拐点识别：
// 并发数 ↑ 但 QPS 不变 → 系统达到瓶颈
// 响应时间随并发数线性增长 → 资源不足
```

---

## 资源指标

### 常见监控指标

| 指标 | 告警阈值 | 说明 |
|------|---------|------|
| CPU 使用率 | > 80% | 持续高负载 |
| 内存使用率 | > 85% | OOM 风险 |
| 磁盘使用率 | > 80% | 存储不足 |
| 网络带宽 | > 70% | 带宽瓶颈 |
| GC 频率 | > 5 次/秒 | 内存压力 |
| Goroutine 数 | 持续增长 | 泄漏 |

### Go 运行时监控

```go
import (
    "runtime"
    "runtime/metrics"
)

func monitorMetrics() {
    // 基础指标
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Alloc: %d MB\n", m.Alloc/1024/1024)
    fmt.Printf("NumGoroutine: %d\n", runtime.NumGoroutine())
    fmt.Printf("NumGC: %d\n", m.NumGC)

    // 更精确的指标（Go 1.17+）
    const summary = "/gc/heap/alloc:bytes"
    sm := metrics.MakeSnapshot()
    if h, ok := sm.Get(summary); ok {
        fmt.Printf("Heap Alloc: %d bytes\n", h.FloatDetails())
    }
}
```

### Prometheus 指标暴露

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "path"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, httpDuration)
}

func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &responseWriter{ResponseWriter: w, status: 200}
        next.ServeHTTP(rw, r)
        duration := time.Since(start)

        httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, 
            strconv.Itoa(rw.status)).Inc()
        httpDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration.Seconds())
    })
}
```

---

## SLO / SLA 设定

### 合理的 SLO 示例

```yaml
# 面向用户的服务 SLO
serviceLevelObjectives:
  - name: API 可用性
    target: 99.95%  # 允许每年 4.38 小时宕机
    window: 30d

  - name: API P99 延迟
    target: < 500ms
    window: 30d

  - name: 下单成功率
    target: > 99.99%
    window: 30d

  # 内部依赖 SLO
  - name: 数据库可用性
    target: 99.99%

  - name: Redis 可用性
    target: 99.99%
```

### 错误预算（Error Budget）

```
SLO = 99.95% → 错误预算 = 0.05%/天 = 21.6 分钟/月

如果错误率超过 SLO：
1. 停止非紧急变更
2. 全力修复问题
3. 错误预算耗尽 = 违约

SLO 留余量：设置更严格的内部 SLO（如 99.99%）来保护对外 SLA
```

---

## 面试话术

**Q：P99 延迟高怎么排查？**

> P99 高说明有 1% 的请求特别慢，通常原因：1）GC 停顿导致少量请求等待；2）数据库慢查询（缺索引、锁冲突）；3）外部依赖抖动；4）代码里有 O(N²) 或大锁。用 pprof 采样的 P99 延迟请求，看调用栈里哪一步耗时最多。

**Q：如何设定合理的 SLO？**

> 1）看历史数据，现有系统能稳定达到什么水平；2）看业务需求，对可用性有多敏感；3）比竞品略高或持平；4）设定后持续监控，偏差超过 10% 就要告警。SLO 不是越高越好，越高意味着成本指数增长。

**Q：系统突然变慢，怎么快速定位？**

> 1）先看监控，是所有接口都慢还是某一个慢；2）如果所有接口慢，看 CPU/内存/GC，有可能是资源耗尽；3）如果单个接口慢，加链路追踪，看是哪个下游超时；4）再看数据库慢查询日志，是否有全表扫描。Go 服务的话先用 pprof 采当前请求。
