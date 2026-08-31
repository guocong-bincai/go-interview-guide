# Go 可观测性：日志/指标/链路追踪落地实践

> 考察频率：★★★★☆  优先级：P1
> 关键词：structured logging、slog、Prometheus metrics、OpenTelemetry、TraceID 透传、采样策略

## 面试官考察意图

可观测性是高级工程师**从写代码到运维的全栈能力**。面试官想听到候选人不仅会用 zap/logrus，还要能说清楚：
- 结构化日志 vs 文本日志的区别和取舍
- Prometheus 指标的四个类型及选型逻辑
- 分布式追踪如何跨服务传递上下文
- 采样率怎么设才能兼顾成本和效果

---

## 核心答案（30 秒版）

Go 的可观测性三支柱是：**结构化日志（slog/zap）+ Prometheus 指标 + OpenTelemetry 链路追踪**。关键是每个请求携带唯一的 TraceID，所有日志和指标都带上这个 ID。日志用 JSON 格式输出方便 ELK/Loki 解析；指标按业务维度打 label 区分；trace 对高频低价值请求做比例采样。**三者互补：日志看细节，指标看趋势，trace 看路径。**

---

## 1. 结构化日志最佳实践

### Go 1.21 slog 标准库方案

```go
package main

import (
    "context"
    "log/slog"
    "os"
)

type contextKey string

const traceIDKey contextKey = "traceID"

// 获取当前请求的 TraceID
func getTraceID(ctx context.Context) string {
    if v, ok := ctx.Value(traceIDKey).(string); ok {
        return v
    }
    return "unknown"
}

func NewLogger(ctx context.Context) *slog.Logger {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level:     slog.LevelDebug,
        AddSource: true,
    })

    // 为每个 Logger 实例附加 traceID
    return slog.New(handler).With("trace_id", getTraceID(ctx))
}

// HTTP Handler 示例
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    traceID := getTraceID(ctx)

    logger := slog.Default().With("trace_id", traceID, "method", r.Method, "path", r.URL.Path)
    
    logger.Info("处理请求",
        slog.String("user_id", getUserID(r)),
        slog.Float64("duration_ms", 123.5),
    )
}
```

### zap 替代方案（性能最优）

```go
import "go.uber.org/zap"

logger, _ := zap.NewProduction()
defer logger.Sync()

// 高性能结构化日志 — 零反射、零分配
logger.Info("处理请求",
    zap.String("trace_id", traceID),
    zap.String("user_id", userID),
    zap.Duration("duration", time.Since(start)),
)
```

**slog vs zap 选型：**
| 维度 | slog (stdlib) | zap |
|------|-------------|-----|
| 性能 | 中等 | 最优（业界标杆） |
| 零依赖 | ✅ | ❌ 需引入 |
| 生态 | 标准库，未来可期 | 社区成熟，生产验证多 |
| 建议 | Go 1.21+ 新项目用 slog | 高性能场景用 zap |

---

## 2. Prometheus 指标体系设计

### 四个指标类型的正确选择

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

// Counter — 只增不减（如请求总数、错误数）
var totalRequests = promauto.NewCounter(prometheus.CounterOpts{
    Namespace: "myapp",
    Subsystem: "http",
    Name:      "requests_total",
    Help:      "Total HTTP requests",
})

// Gauge — 可升可降（如在线用户数、队列长度）
var activeUsers = promauto.NewGauge(prometheus.GaugeOpts{
    Namespace: "myapp",
    Subsystem: "users",
    Name:      "active_count",
    Help:      "Currently active users",
})

// Histogram — 分布统计（如请求耗时，自动分桶）
var requestDuration = promauto.NewHistogramVec(
    prometheus.HistogramOpts{
        Namespace: "myapp",
        Subsystem: "http",
        Name:      "request_duration_seconds",
        Help:      "HTTP request duration",
        Buckets:   prometheus.DefBuckets, // [0.005, 0.01, ..., 10]
    },
    []string{"method", "status"}, // label 维度
)

// Summary — 客户端计算的分位数（慢，少用）
// 通常 Histogram + Grafana PromQL 就够了
```

### 指标命名规范（Prometheus 官方建议）

```
{namespace}_{subcategory}_{name}_{unit}

例子：
├── myapp_http_requests_total          # Counter
├── myapp_http_request_duration_seconds_bucket  # Histogram 桶
├── myapp_db_query_duration_seconds       # Histogram
├── myapp_redis_cache_hits_total          # Counter
└── myapp_goroutines_current             # Gauge
```

### 关键指标清单

```yaml
# 必监控的核心指标
业务层:
  - HTTP 请求量 (Counter by method+status)
  - HTTP 响应延迟 (Histogram p50/p95/p99)
  - 错误率 (Counter / Counter * 100%)

Go 运行时:
  - goroutine 数量 (Gauge)
  - GC 暂停时间 (Histogram)
  - 堆内存使用量 (Gauge)
  - alloc_bytes/sec (Counter)

中间件:
  - DB 连接池活跃数 (Gauge)
  - Redis 查询耗时 (Histogram)
  - MQ 消息堆积数 (Gauge)
```

---

## 3. 分布式追踪与 TraceID 透传

### 自定义 middleware 实现 TraceID 透传

```go
func TraceMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 从请求头或生成新 TraceID
        traceID := r.Header.Get("X-Trace-Id")
        if traceID == "" {
            traceID = generateUUID()
        }
        
        parentSpanID := r.Header.Get("X-Span-Id")
        spanID := generateUUID()

        ctx := context.WithValue(r.Context(), "trace_id", traceID)
        ctx = context.WithValue(ctx, "span_id", spanID)
        ctx = context.WithValue(ctx, "parent_span_id", parentSpanID)

        // 注入回响应头，供下游使用
        w.Header().Set("X-Trace-Id", traceID)
        w.Header().Set("X-Span-Id", spanID)

        start := time.Now()
        next.ServeHTTP(w, r.WithContext(ctx))
        duration := time.Since(start)

        // 记录结构化日志
        slog.Info("HTTP request completed",
            slog.String("trace_id", traceID),
            slog.String("method", r.Method),
            slog.String("path", r.URL.Path),
            slog.Int("status", getStatusFromResponse(w)),
            slog.Duration("duration", duration),
        )
    })
}

// 调用下游服务时自动传递 TraceID
func (svc *UserService) CallPaymentService(ctx context.Context, orderID string) error {
    traceID := ctx.Value("trace_id").(string)
    spanID := ctx.Value("span_id").(string)

    req, _ := http.NewRequestWithContext(ctx, "POST", paymentURL, body)
    req.Header.Set("X-Trace-Id", traceID)
    req.Header.Set("X-Span-Id", spanID)
    
    resp, err := httpClient.Do(req)
    // ...
}
```

### OpenTelemetry 集成（生产推荐）

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

// 初始化 tracer provider（采集到 Jaeger/Zipkin）
func initTracer() (*sdktrace.TracerProvider, error) {
    exporter, err := jaeger.New(jaeger.WithCollectorEndpoint(
        jaeger.WithEndpoint("http://jaeger-collector:14268/api/traces"),
    ))
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithSampler(sdktrace.ParentBased(
            sdktrace.TraceIDRatioBased(0.1), // 10% 全量采样
        )),
        sdktrace.WithBatcher(exporter),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}

// Span 使用示例
func OrderPayment(ctx context.Context, orderID string) error {
    tracer := otel.Tracer("myapp/payment")
    ctx, span := tracer.Start(ctx, "process_payment")
    defer span.End()

    span.SetAttributes(
        attribute.String("order.id", orderID),
    )

    result, err := processOrder(ctx, orderID)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
    }
    return err
}
```

---

## 4. 采样策略

| 采样类型 | 比例 | 适用场景 |
|---------|------|---------|
| 全采样 (100%) | 1.0 | 低流量服务、核心链路、故障排查期间 |
| 按比例采样 (1%) | 0.01 | 高流量服务日常运行 |
| 基于错误的采样 | 动态 | 仅对失败请求全采样，正常请求抽样 |
| 固定根因采样 | 100% | 特定 TraceID（bug 复现时用） |

```go
// 基于错误率的动态采样
func dynamicSampler(rate float64) sdktrace.Sampler {
    return sdktrace.ParentBased(sdktrace.TraceIDRatioBased(rate))
}

// 采样率调整公式：
// 目标 QPS × 平均 span 大小 ≈ 带宽上限
// 假设 Jaeger 每 span 1KB，带宽 100MB/s
// → 最大支持 100,000 span/s → 对应 10,000 QPS（10%采样）
```

---

## 面试话术

> "我团队的可观测性体系由三部分构成：用 slog 输出结构化 JSON 日志给 Loki 做聚合检索；用 Prometheus client_golang 暴露四种指标类型，覆盖业务层、运行时和中间件三个维度；用 OpenTelemetry SDK 实现跨服务的链路追踪，通过自定义 middleware 把 TraceID 透传到所有上下游。日常运营用 1% 的比例采样，出问题时手动切换到全采样做根因分析。"

---

[🏠 首页](../../../README.md) · [📦 工程素养](../README.md) · [🛡️ 技术领导力](./README.md)
