# Prometheus + Grafana 监控实战

> 考察频率：★★★★☆  优先级：P1
> 关键词：PromQL、RED 方法、USE 方法、Histogram、P99、SLO

## 为什么需要监控

监控是服务可靠性的基石，目标是**先于用户发现问题**。

```
监控金字塔：
         ┌──────────────┐
         │   Business    │  ← 业务指标（DAU、订单量、转化率）
         ├──────────────┤
         │   Service    │  ← 应用指标（QPS、延迟、错误率）
         ├──────────────┤
         │     Infra     │  ← 基础设施（CPU、内存、磁盘）
         └──────────────┘

推荐优先级：Service > Business > Infra
原因：Service 指标最能反映用户实际体验
```

---

## 1. Prometheus 数据模型

### 时间序列

```
Prometheus 数据格式：
metric_name{label1="v1", label2="v2"} value timestamp

实际存储示例：
http_requests_total{method="GET", path="/api/orders", status="200"} 9527 1700000000
http_requests_total{method="POST", path="/api/orders", status="200"} 1234 1700000000
http_requests_total{method="GET", path="/api/orders", status="500"} 23 1700000000
```

**标签（Label）设计原则：**
```go
// ✅ 好的标签设计：低基数
http_requests_total{method="GET", path="/api/orders"}  // 路径是固定的

// ❌ 错误：高基数标签会导致 Prometheus 内存爆炸
http_requests_total{user_id="12345"}  // 100万用户 = 100万个时间序列！
http_requests_total{request_id="uuid-xxx"}  // 每请求一个 = 不可行
```

**高基数标签的危害：**
- 每增加一个唯一标签值 → 增加一个独立时间序列
- Prometheus 内存消耗 = 时间序列数 × 保留时间 × 采样率
- 10 亿时间序列 → Prometheus OOM

### 四种指标类型

```go
import "github.com/prometheus/client_golang/prometheus"

func setupMetrics() {
    // 1. Counter：只增不减的计数器（请求数、错误数、订单数）
    httpRequestsTotal := prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    // 2. Gauge：可增可减的仪表（当前在线人数、内存使用量、队列长度）
    currentUsers := prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "current_online_users",
            Help: "Current number of online users",
        },
    )

    // 3. Histogram：直方图（延迟分布、体积分布）
    httpDuration := prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request latency distribution",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
            // 桶边界决定 P99/P999 的精度
        },
        []string{"method", "path"},
    )

    // 4. Summary：客户端计算分位数（自定义分位数，由客户端计算）
    requestSummary := prometheus.NewSummaryVec(
        prometheus.SummaryOpts{
            Name:       "http_request_summary_seconds",
            Help:       "HTTP request latency summary",
            Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
            // 目标：P50=50.05，P90=0.01，P99=0.001
        },
        []string{"method", "path"},
    )
}
```

### Histogram vs Summary 对比

| 维度 | Histogram | Summary |
|------|-----------|---------|
| 分位数计算 | 服务端（PromQL 计算）| 客户端（预计算）|
| 查询灵活性 | ✅ 高（可算任意分位数）| ❌ 低（预定义分位数）|
| 存储空间 | 较大（多个桶的时间序列）| 较小（只有几个分位数序列）|
| 准确性 | 取决于桶边界设置 | 精确（客户端精确计算）|
| 适用场景 | 延迟分布（P50/P90/P99）| 体验更细腻的业务指标 |

---

## 2. PromQL 高频查询

### `rate()` vs `irate()`

```promql
# rate：计算平均每秒增长率（用于 P99/SUM 等聚合）
# 推荐用于：计算 QPS、累计错误数
rate(http_requests_total[5m])          # 5 分钟窗口的平均 QPS
rate(http_requests_total{status="500"}[5m])  # 5 分钟窗口的错误率

# irate：计算瞬时增长率（用于尖刺检测）
# 推荐用于：快速发现突变
irate(http_requests_total[5m])  # 最后一个样本点的瞬时 QPS

# 区别：
# rate(10分钟内收到100个请求) = 100/600 = 0.167/s（平均）
# irate = 最后两个样本点的斜率（可能比平均高很多或低很多）
```

### `histogram_quantile()` 计算 P99

```promql
# P99 延迟（99% 请求在多少秒内完成）
histogram_quantile(0.99,
    rate(http_request_duration_seconds_bucket[5m])
)
# 原理：
# 1. rate() 计算每个桶的每秒请求数
# 2. histogram_quantile(0.99, ...) 按桶累积
# 3. 找到第 99% 请求所在的桶边界

# P99 按 path 分类
histogram_quantile(0.99,
    rate(http_request_duration_seconds_bucket{path="/api/orders"}[5m])
)
```

### 按维度聚合

```promql
# sum by：按标签聚合（保留指定标签）
sum by (method, status)(rate(http_requests_total[5m]))

# sum without：按标签聚合（排除指定标签）
sum without (instance)(rate(http_requests_total[5m]))

# count_values：按值分组计数
count_values("status_code", http_response_status)
```

### Recording Rules（预计算）

```yaml
# prometheus.yml 或单独 rule 文件
groups:
  - name: service_slo
    interval: 30s
    rules:
      # 预计算慢查询（避免 dashboard 每次实时计算）
      - record: job:http_request_p99:rate5m
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
          * 1000  # 转换为毫秒
```

---

## 3. Go 服务接入全流程

### 安装依赖

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promhttp
go get github.com/prometheus/client_golang/prometheus/promauto
```

### 注册指标并暴露 `/metrics`

```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // HTTP 请求总数
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    // HTTP 请求延迟
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request latency in seconds",
            Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1},
        },
        []string{"method", "path"},
    )

    // 当前在线用户数
    onlineUsers = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "online_users_total",
            Help: "Current number of online users",
        },
    )

    // 业务指标：订单数
    ordersTotal = promauto.NewCounter(
        prometheus.CounterOpts{
            Name: "orders_created_total",
            Help: "Total number of orders created",
        },
    )
)

func main() {
    // /metrics 端点（Prometheus 抓取用）
    http.Handle("/metrics", promhttp.Handler())

    // 业务接口
    http.HandleFunc("/api/orders", handleOrders)
    http.ListenAndServe(":8080", nil)
}
```

### HTTP 中间件：自动记录请求指标

```go
// PrometheusMiddleware 记录每个请求的 QPS 和延迟
func PrometheusMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 跳过 /metrics 自身
        if r.URL.Path == "/metrics" {
            next.ServeHTTP(w, r)
            return
        }

        start := time.Now()
        path := r.URL.Path
        method := r.Method

        // 包装 ResponseWriter，捕获 status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(wrapped, r)

        duration := time.Since(start).Seconds()
        status := fmt.Sprintf("%d", wrapped.statusCode)

        // 记录指标
        httpRequestsTotal.WithLabelValues(method, path, status).Inc()
        httpRequestDuration.WithLabelValues(method, path).Observe(duration)
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

### Gin 框架集成

```go
import (
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()

    // /metrics 端点
    r.GET("/metrics", gin.WrapH(promhttp.Handler()))

    // 业务路由
    r.GET("/api/orders", listOrders)

    r.Run(":8080")
}

// Gin 中间件（需要 github.com/gin-contrib/obsresp）
```
---

## 4. Grafana 看板设计

### RED 方法（Requests / Errors / Duration）

```
RED 方法 = Rate（QPS）+ Errors（错误率）+ Duration（延迟分布）

看板布局建议（3 行 3 列）：
┌─────────────────┬─────────────────┬─────────────────┐
│   Rate (QPS)    │  Error Rate (%) │  P50 Latency    │
│   sum(rate())   │  errors/total   │  histogram_q(0.5)│
├─────────────────┼─────────────────┼─────────────────┤
│   P90 Latency   │   P99 Latency   │  P999 Latency   │
├─────────────────┼─────────────────┼─────────────────┤
│  Error 500 Count │ Timeout Count   │  CPU Usage      │
└─────────────────┴─────────────────┴─────────────────┘
```

### USE 方法（Utilization / Saturation / Errors）

```
USE 方法 = 利用率 + 饱和度 + 错误数（适合资源层监控）

| 资源类型 | 利用率（Usage）| 饱和度（Saturation）| 错误（Errors）|
|---------|--------------|-------------------|-------------|
| CPU     | cpu_usage%   | load_average      | -           |
| 内存    | memory_used% | oom_count         | -           |
| 磁盘    | disk_io%     | disk_queue_length | -           |
| 网络    | net_tx/rx    | net_drop_rate     | net_err     |
```

### SLO 看板示例

```promql
# 服务可用性（月度）
# 可用性 = 成功请求 / 总请求（status < 500）
sum(rate(http_requests_total{status=~"2.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# SLO 达标率（本月 99.9% 可用性目标）
# 如果 < 99.9%，触发告警
(
  sum(rate(http_requests_total{status=~"2.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
) > 0.999

# P99 延迟 SLO（< 200ms）
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) < 0.2
```

### Grafana Variables（动态切换）

```yaml
# Grafana 变量配置（支持动态切换环境/实例）
variables:
  - name: env
    type: query
    query: label_values(http_requests_total, env)  # 自动从 metrics 获取
    # 支持：prod / staging / dev

  - name: instance
    type: query
    query: label_values(http_requests_total{env="$env"}, instance)
    # 根据 env 动态过滤 instance

# Panel 中使用：
# ${env} / ${instance} → 实际值
sum by (path) (rate(http_requests_total{env="$env", instance="$instance"}[5m]))
```

---

## 5. AlertManager 告警配置

### 告警规则

```yaml
# alertrules.yml（PrometheusRule CRD）
groups:
  - name: service_alerts
    rules:
      # P1：P99 延迟超过 2s
      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 2m    # 持续 2 分钟才触发（避免抖动）
        labels:
          severity: critical
        annotations:
          summary: "服务 P99 延迟过高"
          description: "{{ $labels.job }} P99 延迟 {{ $value }}s，超过 2s 阈值"

      # P2：错误率超过 1%
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m])) > 0.01
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "错误率 {{ $value | humanizePercentage }} 超过 1%"

      # P3：服务不可用
      - alert: ServiceDown
        expr: up{job="order-service"} == 0
        for: 30s
        labels:
          severity: critical
```

### AlertManager 配置

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.exmail.qq.com:587'
  smtp_from: 'alert@company.com'

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s        # 告警聚合等待时间
  group_interval: 5m     # 同组告警发送间隔
  repeat_interval: 4h    # 重复告警间隔
  receiver: 'default'
  routes:
    # critical 告警 → 立即通知
    - match:
        severity: critical
      receiver: 'critical-receiver'
      group_wait: 0s
    # warning 告警 → 工作时间通知
    - match:
        severity: warning
      receiver: 'warning-receiver'

receivers:
  - name: 'critical-receiver'
    # 立即通知：电话 + 短信 + 钉钉
    webhook_configs:
      - url: 'http://dingtalk-webhook/alert'

  - name: 'warning-receiver'
    # 告警 → 钉钉工作群（延迟可接受）
    webhook_configs:
      - url: 'http://dingtalk-webhook/warning'

# 告警抑制：P0 发生时，抑制 P1/P2 告警
inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'instance']  # 同 alertname 和 instance 才抑制
```

---

## 6. 生产踩坑

### 坑 1：Prometheus 内存暴涨（高基数标签）

```go
// ❌ 错误：高基数标签
httpRequestsTotal.WithLabelValues(userID, requestID, traceID)  // 灾难

// ✅ 正确：低基数标签
httpRequestsTotal.WithLabelValues(method, path, status)

// 如果确实需要记录 user_id，用 Cardinality 限制
// 或者用 tracing 系统（Jaeger）而非 Prometheus 指标
```

### 坑 2：Histogram 桶边界设置不合理

```go
// ❌ 桶边界不均匀：无法精确区分 P99
Buckets: []float64{0.001, 0.002, 0.005, 0.01, 0.02, 0.05, 0.1, 0.2, 0.5, 1, 2, 5, 10}

// ✅ 桶边界均匀（针对 P99 优化）：
// 桶数：16-20 个，覆盖 1ms ~ 10s
Buckets: []float64{0.001, 0.002, 0.004, 0.008, 0.016, 0.032, 0.064,
                    0.128, 0.256, 0.512, 1, 2, 4, 8, 16}

// ✅ 如果目标是 P99 = 200ms：
Buckets: prometheus.ExponentialBuckets(0.01, 1.5, 15)
// 起始 10ms，增长率 1.5 倍，15 个桶 → 覆盖 10ms ~ 3s
```

### 坑 3：长期存储（Thanos / VictoriaMetrics）

```yaml
# Thanos Sidecar 架构
# Prometheus + Sidecar → 对象存储（S3）
# Querier → 统一查询（实时 + 历史）
components:
  - sidecar（每个 Prometheus 伴随）
  - store（历史数据查询）
  - querier（统一查询入口）
  - receiver（Remote Write 接收端）
```

---

## 高频追问

### Q1：Histogram 的 bucket 应该怎么设置？

**原则：围绕 SLA 目标设置桶中心：**

```go
// 场景：API 服务，P50=50ms，P90=200ms，P99=500ms
// 目标 P99 = 500ms，桶边界应该围绕 500ms 密集分布

Buckets: []float64{
    0.010,   // 10ms   → 覆盖 P50
    0.050,   // 50ms
    0.100,   // 100ms  → P70
    0.200,   // 200ms  → P90
    0.300,   // 300ms
    0.400,   // 400ms
    0.500,   // 500ms  → P99 目标
    0.750,   // 750ms
    1.000,   // 1s
    2.500,   // 2.5s
    5.000,   // 5s
    10.00,   // 10s
}
```

### Q2：P99 和 P999 的区别，分别适合什么场景？

| 分位数 | 含义 | 适用场景 |
|--------|------|---------|
| **P50** | 中位数，50% 请求在此时完成 | 衡量整体性能，检测异常低值 |
| **P90** | 90% 请求在此时完成 | SLA 承诺（如 P90 < 200ms）|
| **P99** | 99% 请求在此时完成 | 高标准 SLA，检测长尾问题 |
| **P999** | 99.9% 请求在此时完成 | 极其关键服务，发现极端长尾 |

**P99 vs P999：**
- P99 = 200ms：说明 1% 的用户感受到 > 200ms 的延迟
- P999 = 2s：说明 0.1% 的用户感受到 > 2s 的延迟
- 电商 P99；金融/支付 P999

### Q3：为什么高基数标签会导致 Prometheus OOM？

**根本原因：Prometheus 存储模型是"每个时间序列独立存储"的**

```
时间序列数计算：
时间序列 = metric_name × label_values 的笛卡尔积

例子：
- 1 个 metric = http_requests_total
- 2 种 method（GET, POST）
- 100 种 path（/api/users, /api/orders, ...）
- 10 台 instance

= 1 × 2 × 100 × 10 = 2000 个时间序列（合理）

如果 path = user_id（假设 100 万用户）：
= 1 × 2 × 1,000,000 × 10 = 20,000,000 个时间序列（OOM 预警）
```

**解决高基数问题：**
1. 用 tracing 系统（Jaeger）跟踪单个请求，不依赖 metrics
2. 用 `apiserver_request_duration_seconds_bucket` 等预聚合 metrics
3. 使用 VictoriaMetrics（支持更高基数）
