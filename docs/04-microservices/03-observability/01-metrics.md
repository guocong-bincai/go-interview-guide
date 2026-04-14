# 微服务可观测性 - 指标体系

> 考察频率：★★★★☆  难度：★★★☆☆

## RED 指标体系

微服务三大黄金指标：

| 指标 | 全称 | 说明 | 告警阈值示例 |
|------|------|------|------------|
| **Rate** | 请求率 | QPS，每秒请求数 | 低于基线 50% |
| **Error** | 错误率 | 错误请求占比 | > 1% |
| **Duration** | 延迟 | P50/P90/P99 响应时间 | P99 > 500ms |

> **USE 方法**（用于资源指标）：Utilization（利用率）、Saturation（饱和度）、Errors（错误）

---

## Prometheus Go 客户端

### 快速接入

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
)

var (
    // 业务指标
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5},
        },
        []string{"method", "path"},
    )

    activeConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, httpRequestDuration, activeConnections)
}

// HTTP 中间件
func PrometheusMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // 包装 ResponseWriter 捕获 status
        wrapped := &responseWriter{ResponseWriter: w, status: 200}
        next.ServeHTTP(wrapped, r)

        duration := time.Since(start).Seconds()
        path := r.URL.Path

        httpRequestsTotal.WithLabelValues(r.Method, path, 
            strconv.Itoa(wrapped.status)).Inc()
        httpRequestDuration.WithLabelValues(r.Method, path).Observe(duration)
    })
}
```

### Gin 框架集成

```go
import (
    "github.com/gin-gonic/gin"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

func SetupPrometheus(r *gin.Engine) {
    r.Use(gin.WrapH(promhttp.Handler()))

    r.GET("/metrics", func(c *gin.Context) {
        promhttp.Handler().ServeHTTP(c.Writer, c.Request)
    })
}
```

### 暴露 Go 运行时指标

```go
// Go 运行时指标（自动注册）
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/collectors"
)

func init() {
    // 注册 Go 运行时 Collector
    prometheus.MustRegister(collectors.NewGoCollector())
    prometheus.MustRegister(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}))
}
```

---

## Go 运行时关键指标

### GC 相关

| 指标 | 含义 | 优化目标 |
|------|------|---------|
| `go_gc_duration_seconds` | GC STW 时间 | P99 < 10ms |
| `go_gc_runs_total` | GC 运行次数 | 不频繁 |
| `go_memstats_live_heap_bytes` | 存活堆大小 | 稳定 |

### 内存相关

| 指标 | 含义 | 关注点 |
|------|------|-------|
| `go_memstats_alloc_bytes` | 当前堆内存 | 持续增长 = 泄漏 |
| `go_memstats_heap_alloc_bytes` | 堆分配字节 | 接近 limit = OOM 前兆 |
| `go_memstats_mcache_inuse_bytes` | mcache 大小 | 稳定 |

### Goroutine 相关

| 指标 | 含义 | 告警 |
|------|------|------|
| `go_goroutines` | 当前 goroutine 数 | 突增 = 泄漏 |
| `go_threads` | OS 线程数 | > 100 = 阻塞 |

---

## GOGC 和 GOMEMLIMIT 调优

### GOGC（GC 触发阈值）

默认 GOGC=100（堆增长 100% 时触发 GC）：

```go
// 方式一：环境变量
// GOGC=200 表示堆增长 200% 才触发 GC（更少 GC，更高内存）

// 方式二：代码设置
debug.SetGCPercent(200)

// 场景：
// - 内存敏感：降低 GOGC（如 50），更频繁 GC，内存占用低
// - CPU 敏感：提高 GOGC（如 300），GC 次数少，CPU 利用率高
```

### GOMEMLIMIT（内存上限）

Go 1.19+，限制内存使用：

```go
// 防止 OOM，接近 limit 时加速 GC
// GOMEMLIMIT=10GiB

debug.SetMemoryLimit(10 * 1024 * 1024 * 1024) // 10GB
```

**调优策略**：

```go
// 高吞吐服务（如消息队列消费者）
// GOGC=300，降低 GC 频率，提升吞吐

// 低延迟服务（如 RPC）
// GOGC=100，平衡内存和延迟

// 内存敏感（如 Sidecar）
// GOMEMLIMIT=100MiB，GOGC=50，严格控制内存
```

---

## SLO/SLA 定义与监控

### 常见 SLO

```yaml
# example-slo.yaml
service: order-service
latency:
  threshold: 200ms
  success_rate: 0.99  # P99 < 200ms
availability:
  target: 0.999       # 每月停机 < 4.4 分钟
errors:
  rate_threshold: 0.01 # 错误率 < 1%
```

### Grafana 面板配置

```go
// SLO 达标率计算
// 可用率 = 成功请求 / 总请求
// 延迟达标率 = P99 < 阈值 的请求 / 总请求

// Prometheus 告警规则
groups:
  - name: slo-alerts
    rules:
      - alert: LatencySLOBreach
        expr: |
          histogram_quantile(0.99, 
            rate(http_request_duration_seconds_bucket[5m])
          ) > 0.2
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "P99 latency exceeds 200ms"

      - alert: ErrorRateBreach
        expr: |
          rate(http_requests_total{status=~"5.."}[5m])
          / rate(http_requests_total[5m]) > 0.01
        for: 2m
        labels:
          severity: warning
```

---

## 实战：Go 服务指标体系落地

### 1. 接入层指标

```go
type MiddlewareMetrics struct {
    requestsTotal   *prometheus.CounterVec
    requestDuration *prometheus.HistogramVec
}

func NewMiddlewareMetrics(namespace string) *MiddlewareMetrics {
    m := &MiddlewareMetrics{
        requestsTotal: prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Namespace: namespace,
                Name:      "requests_total",
                Help:      "Total requests",
            },
            []string{"method", "path", "status"},
        ),
        requestDuration: prometheus.NewHistogramVec(
            prometheus.HistogramOpts{
                Namespace: namespace,
                Name:      "request_duration_seconds",
                Help:      "Request duration",
                Buckets:   []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5},
            },
            []string{"method", "path"},
        ),
    }
    prometheus.MustRegister(m.requestsTotal, m.requestDuration)
    return m
}
```

### 2. 业务层指标

```go
// 订单服务特有指标
var (
    ordersCreatedTotal = prometheus.NewCounter(prometheus.CounterOpts{
        Namespace: "order",
        Name:      "created_total",
        Help:      "Total orders created",
    })

    orderPaymentDuration = prometheus.NewHistogram(prometheus.HistogramOpts{
        Namespace: "order",
        Name:      "payment_duration_seconds",
        Help:      "Payment processing duration",
        Buckets:   []float64{.1, .25, .5, 1, 2.5, 5, 10},
    })

    inventoryReservedGauge = prometheus.NewGauge(prometheus.GaugeOpts{
        Namespace: "inventory",
        Name:      "reserved_quantity",
        Help:      "Reserved inventory quantity",
    })
)
```

### 3. 基础设施层指标

```go
// Redis/DB 连接池
var (
    dbPoolConnections = prometheus.NewGauge(prometheus.GaugeOpts{
        Namespace: "db",
        Subsystem: "pool",
        Name:      "connections",
        Help:      "DB pool connections",
    })

    dbPoolWaitDuration = prometheus.NewHistogram(prometheus.HistogramOpts{
        Namespace: "db",
        Subsystem: "pool",
        Name:      "wait_duration_seconds",
        Help:      "Time waiting for connection",
        Buckets:   []float64{.001, .005, .01, .025, .05, .1},
    })
)
```

---

## 常见告警规则

```yaml
# alertmanager 告警规则
groups:
  - name: go-service
    rules:
      # Goroutine 泄漏
      - alert: GoroutineLeak
        expr: go_goroutines > 10000
        for: 10m
        annotations:
          summary: "Goroutine count abnormally high"

      # 内存泄漏
      - alert: MemoryLeak
        expr: |
          rate(go_memstats_alloc_bytes[30m]) > 0
          and go_memstats_alloc_bytes > 1e9
        for: 30m
        annotations:
          summary: "Possible memory leak detected"

      # GC 频繁
      - alert: GCPressure
        expr: rate(go_gc_runs_total[5m]) > 50
        for: 5m
        annotations:
          summary: "GC running too frequently"

      # 连接池耗尽
      - alert: DBPoolExhausted
        expr: db_pool_wait_duration_seconds{quantile="0.99"} > 0.1
        for: 5m
        annotations:
          summary: "DB connection pool wait time high"
```

---

## 总结：Go 服务指标四层模型

| 层次 | 指标 | 工具 |
|------|------|------|
| **应用层** | QPS、延迟、错误率 | Prometheus + Histogram |
| **运行时层** | Goroutine、GC、线程 | `go_goroutines`、`go_gc_duration_seconds` |
| **内存层** | 堆内存、mcache | `go_memstats_alloc_bytes` |
| **连接层** | DB/Redis 连接池 | 自定义 Gauge |

### 关键原则
1. **RED 优先**：Rate、Error、Duration 是核心
2. **服务级别指标**：每个服务独立注册
3. **卡片inality 控制**：高 cardinality 标签（user_id）不要入指标
4. **SLO 驱动**：告警围绕 SLO，而非具体数值

---

## 面试话术

**Q：Go 服务的内存指标怎么看？**

> 主要看 `go_memstats_alloc_bytes`（当前堆内存）和 `go_memstats_heap_alloc_bytes`（已分配堆）。如果堆内存持续增长，可能是内存泄漏；如果接近 GOMEMLIMIT，需要尽快扩容或优化。另外 `go_gc_duration_seconds` 反映 GC STW 时间，如果 P99 > 10ms，说明 GC 太重，需要调 GOGC 或优化内存分配。

**Q：如何通过指标发现 goroutine 泄漏？**

> 监控 `go_goroutines` 指标。正常服务 goroutine 数应该在几百到几千稳定。如果发现 goroutine 数持续增长，不回落，配合 `go_goroutine` 的 pprof profile（查找 `goroutine` view），找到创建 goroutine 但不退出的地方——通常是 channel 阻塞或 context 未取消。

**Q：GOGC 怎么调？**

> GOGC=100 是默认值，表示堆内存翻倍时触发 GC。如果服务延迟敏感（RPC），保持 GOGC=100；如果吞吐敏感（离线任务），可以提高到 200-300 减少 GC；如果内存敏感（Sidecar），降到 50。Go 1.19 后用 GOMEMLIMIT 硬性限制内存。
