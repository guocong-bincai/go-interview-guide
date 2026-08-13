# 可观测性三支柱整合：OpenTelemetry 与零侵入采集

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：Metrics、Logs、Traces、OpenTelemetry、TraceID 贯穿、eBPF、零侵入、可观测性体系

---

## 面试官考察意图

"你们服务出问题了，怎么快速定位？"——高级工程师面试必问。单讲 Prometheus 或 ELK 已经不够，面试官想考察：

1. 是否理解**三支柱（Metrics/Logs/Traces）各自解决什么问题**，为什么缺一不可
2. 是否知道**三支柱如何联动**（TraceID 贯穿日志与指标，日志→链路→指标三跳定位）
3. 是否了解 **OpenTelemetry** 这个行业标准（2026 年已是大厂标配）
4. 是否有 **eBPF 零侵入采集**的认知（无需改代码就能观测）

这道题考察的是"可观测性体系"的完整认知，不是单点工具。

---

## 核心答案（30 秒版）

三支柱分工：**指标（Metrics）告诉你"系统出事了"（宏观），日志（Logs）告诉你"发生了什么"（细节），链路（Traces）告诉你"问题出在哪一跳"（路径）。**

整合的核心是 **TraceID 贯穿**：

```text
用户报障 "下单慢"
  │
  ▼ ① 看指标：下单接口 P99 从 200ms → 2s（确认异常 + 时间窗口）
  │
  ▼ ② 查日志：按 TraceID 搜到该次请求的日志（确认哪一步报错/超时）
  │
  ▼ ③ 看链路：Trace 里看到调用支付服务耗时 1.8s（定位问题节点）
  │
  ▼ ④ 下钻：支付服务该时段指标、依赖健康状态 → 根因
```

**一句话：指标发现问题，链路定位问题，日志解释问题；三者靠 TraceID 串联成一条排查路径。**

---

## 深度展开

### 1. 三支柱 vs 四信号

| 信号 | 回答的问题 | 典型工具 | 特点 |
|------|-----------|---------|------|
| **Metrics** | 系统健康吗？ | Prometheus + Grafana | 聚合、长期趋势、告警 |
| **Logs** | 具体发生了什么？ | ELK / Loki / ClickHouse | 细节、事件、排障 |
| **Traces** | 请求经历了什么？ | Jaeger / Tempo / Zipkin | 路径、耗时分布、依赖关系 |
| **Profiling**（第四信号） | 为什么慢？（CPU/内存） | pprof / Pyroscope / eBPF | 持续剖析，按需开启 |

**认知升级**：2026 年的标准答案是"**三支柱 + 持续剖析（Continuous Profiling）**"，全链路压测和线上故障时 Profiling 是定位 CPU/内存问题的利器。

### 2. OpenTelemetry：统一标准

**OpenTelemetry（OTel）** 是 CNCF 的行业标准，统一了三个信号的采集 API、SDK 和数据格式（OTLP）：

```text
应用（Go SDK）──OTLP──▶ Collector ──▶ 后端
                              ├──▶ Prometheus（Metrics）
                              ├──▶ Jaeger/Tempo（Traces）
                              └──▶ Loki/ES（Logs）
```

Go 集成 OTel：

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func initTracer() (func(context.Context) error, error) {
    ctx := context.Background()
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
        otlptracegrpc.WithInsecure())
    if err != nil {
        return nil, err
    }
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("order-service"),
            semconv.DeploymentEnvironment("prod"),
        )),
    )
    otel.SetTracerProvider(tp)
    return tp.Shutdown, nil
}

// 业务代码埋点（HTTP 中间件 / gRPC interceptor 统一处理，业务侧零侵入）
func handler(ctx context.Context) {
    ctx, span := otel.Tracer("order-service").Start(ctx, "CreateOrder")
    defer span.End()
    // span.SetAttributes(attribute.String("order_no", orderNo)) // 关键属性
    // 调用下游时 ctx 透传，自动串联成 Trace
}
```

**面试关键点**：
- **语义约定（Semantic Conventions）**：HTTP 状态码、RPC 方法名、DB 语句等有统一属性名，跨团队可互操作
- **上下文传播（Propagation）**：W3C TraceContext 标准，HTTP header `traceparent` 传递 TraceID，网关→服务→MQ 全链路
- **采样策略**：头部采样（Head-based，按请求 ID 决定整条链）vs 尾部采样（Tail-based，按结果决定，慢/错必采）。生产常用**混合采样**：全量错误 + 10% 正常流量 + 慢请求 100%

### 3. TraceID 贯穿：日志与链路联动

日志里打上 TraceID，排障时"日志 ↔ 链路"互跳：

```go
// 中间件：提取/生成 TraceID，注入日志上下文
func TraceMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        // 从传播头提取（或生成）TraceID
        traceID := getOrCreateTraceID(r) // 实际用 otel.GetTracerProvider 的 propagator
        // 注入 slog/zerolog 的 context，后续日志自动带上 trace_id
        logger := slog.With("trace_id", traceID, "path", r.URL.Path)
        ctx = withLogger(ctx, logger)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

```text
# 日志输出示例（统一字段，可被 Loki/ES 按 trace_id 检索）
{"level":"ERROR","trace_id":"4bf92f3577b34da6a3ce929d0e0e4736",
 "service":"order-service","msg":"call pay timeout","cost_ms":1800}
```

**落地规范**：
- 所有服务的日志必须包含 `trace_id`（和 `span_id`）
- 网关入口统一生成 TraceID，全链路透传（HTTP header / gRPC metadata / MQ 消息头）
- 日志、链路、指标都带 `service`、`env`、`instance` 标签，保证可关联

### 4. eBPF 零侵入采集（2026 热点）

**eBPF** 可以在不修改业务代码、不重启进程的情况下，在内核态采集应用行为：

| 能力 | 说明 |
|------|------|
| 零侵入 | 无需埋点 SDK，对业务代码完全透明 |
| 内核级观测 | 系统调用、网络包、文件 IO、进程调度 |
| 安全 | 沙箱执行，不阻塞业务路径（可观测性附加开销极小） |
| 代表项目 | Cilium、Pixie、DeepFlow、eBPF 版 APM |

```text
应用进程（不改代码）
    │ eBPF 挂载（tracepoint/kprobe/uprobe）
    ▼
内核态采集 syscall/网络/函数调用
    ▼
用户态聚合 → 生成 Metrics/Traces
```

**应用场景**：
- 无法改代码的**存量老系统**快速接入可观测性
- 网络链路观测（延迟在哪一跳：client→LB→Pod→DB）
- 数据库慢查询根因（应用侧无法观测的连接池问题）

> 面试话术：**"我们有 OpenTelemetry 埋点的标准方案；对于无法改代码的存量服务，用 eBPF 零侵入补齐观测，两类方案互补。"**（体现工程广度）

### 5. 三支柱联动排障 SOP

```text
线上告警（Metrics）：下单成功率 < 99.5%
  │
  ▼
1. 看大盘：哪个服务？哪个接口？哪个机房？（Grafana 下钻）
  │
  ▼
2. 拉错误日志：按 service + error 过滤，取 TraceID 样本（Loki/ES）
  │
  ▼
3. 查链路：TraceID → 看哪一跳耗时/报错（Jaeger/Tempo）
  │
  ▼
4. 定位：支付服务超时 → 看支付服务指标（连接池/GC/依赖健康）
  │
  ▼
5. 处置：熔断/降级/扩容 → 验证指标恢复
```

---

## 高频追问

### Q1：Metrics 能替代 Tracing 吗？为什么还要链路追踪？
> 不能。指标是**聚合视角**（平均/P99），回答"整体是否健康"；链路是**单请求视角**（路径+耗时），回答"这个请求经历了什么"。指标发现问题但无法定位到具体请求和节点，链路才能精确到"哪一次调用、哪一跳慢"。两者互补，不是替代。

### Q2：TraceID 是怎么跨服务传递的？
> W3C TraceContext 标准：网关入口生成 TraceID（如 `4bf92f35...`），通过 HTTP header `traceparent`（gRPC 用 metadata）传给下游；MQ 场景放进消息头。每个服务从上游提取 TraceID，创建自己的 SpanID，parent 关系串联成树。**关键：上下文必须显式透传，业务代码漏传就会断链。**

### Q3：全量采集还是采样？采样率怎么定？
> 全量采集成本高（存储+性能），生产用采样：**错误/慢请求 100% 必采，正常流量按 10% 采样**。高流量系统可降到 1%+错误全采。注意：采样决策点不同（头部/尾部）会影响链路完整性。

### Q4：OpenTelemetry 和 Prometheus/Jaeger 是什么关系？
> OTel 是**采集与标准层**（API/SDK/Collector），Prometheus/Jaeger 是**存储与展示层**。OTel 采集的数据可以导出到 Prometheus（指标）、Jaeger/Tempo（链路）、Loki/ES（日志）。关系：**OTel 统一入口，后端可替换**。

### Q5：日志里没有 TraceID 的老系统怎么排查？
> 降级方案：按业务键（订单号、用户ID、request_id）关联；或通过 eBPF 零侵入补齐。长期方案：推动网关统一注入 TraceID，逐步迁移到 OTel。

---

## 延伸阅读

- 本模块：[Prometheus + Grafana、RED 指标、SLO](09-01-metrics.md)
- 本模块：[结构化日志、ELK 方案、日志采样](14-02-logging.md)（日志与 TraceID 结合章节）
- 本模块：[告警规则设计与告警疲劳治理](10-03-alerting.md)
- 分布式模块：[链路追踪原理（Trace/Span/采样）](../../03-distributed/04-service-mesh/16-03-tracing.md)
