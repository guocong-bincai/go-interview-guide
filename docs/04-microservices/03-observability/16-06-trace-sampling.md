# 分布式链路追踪原理与采样策略

> 考察频率：★★★★★  难度：★★★★☆  关键词：TraceID/SpanID、上下文传播、W3C traceparent、头部采样 vs 尾部采样

## 一、数据模型：Trace / Span / Context

链路追踪源自 Google 的 **Dapper** 论文，核心只有三个概念：

| 概念 | 含义 |
|------|------|
| **Trace** | 一次完整请求的调用链，由 `TraceID`（全局唯一，128bit）标识 |
| **Span** | 链路中的一个操作单元（一次 RPC、一次 SQL、一次函数调用），有 `SpanID` + `ParentSpanID`，构成树形结构 |
| **Context** | 需要在进程/服务间传递的追踪信息（TraceID、SpanID、采样标志、Baggage） |

一次请求的 span 树：

```
TraceID: 4bf92f3577b34da6a3ce929d0e0e4736
└── Span: GET /order (gateway)              [0ms → 120ms]
    ├── Span: OrderService.Create           [5ms → 90ms]
    │   ├── Span: MySQL INSERT orders       [10ms → 25ms]
    │   └── Span: Redis SET lock            [30ms → 33ms]
    └── Span: InventoryService.Deduct       [95ms → 118ms]
```

**关键点**：TraceID 要在**所有异步边界**继续传递——goroutine、消息队列（写进消息 Header）、定时任务，否则链路会断。Go 里靠 `context.Context` 携带，`go func(ctx)` 启动 goroutine 时必须把 ctx 传进去（不要用 `context.Background()`）。

---

## 二、上下文传播：W3C Trace Context（高频考点）

跨进程传播靠 HTTP/gRPC Header。行业标准是 **W3C traceparent**：

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                │
             │  TraceID(32 hex)                  SpanID(16 hex)   flags
             version                                              (01=sampled)
```

外加 `tracestate` 传递供应商私有信息。**面试易错点**：

- 老规范是 B3（Zipkin）和 Jaeger 的 `uber-trace-id`，现在 **W3C + B3 双写**是过渡期常见做法。
- 传播要么用标准 Header，要么用 `grpc-metadata`；**自定义 Header 容易被网关/代理丢掉**。
- HTTP Header 大小有限（通常 8KB），不要把整个链路塞进 Header（Baggage 要克制）。

---

## 三、Go + OpenTelemetry 极简实现

```go
// 1) 初始化 TracerProvider（一次性）
func InitTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
	exp, err := otlptracehttp.New(ctx,
		otlptracehttp.WithEndpoint("otel-collector:4318"), // OTLP over HTTP
		otlptracehttp.WithInsecure(),
	)
	if err != nil {
		return nil, err
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exp), // 批量上报，别每条 span 发一次网络请求
		sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.01))), // 1% 采样
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceName("order-service"),
		)),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{}, // W3C
		propagation.Baggage{},
	))
	return tp, nil
}

// 2) 业务里创建 span
func (s *OrderService) Create(ctx context.Context, req *Req) error {
	ctx, span := otel.Tracer("order").Start(ctx, "OrderService.Create")
	defer span.End()

	if err := s.validator.Check(ctx, req); err != nil {
		span.RecordError(err)                      // 只记 error 属性，不改 Status 语义
		span.SetStatus(codes.Error, "validate failed")
		return err
	}
	return s.repo.Insert(ctx, req) // 下游用同一个 ctx → 自动成为子 span
}
```

---

## 四、采样策略：头部采样 vs 尾部采样（必考）

| 策略 | 决策时机 | 优点 | 缺点 |
|------|---------|------|------|
| **头部采样（Head Sampling）** | 请求入口，生成 TraceID 时 | 实现简单、开销极小、采样决策随上下文传播 | **看不到结果**，无法"只留错误/慢请求"，且小流量 + 低采样率会丢关键链路 |
| **尾部采样（Tail Sampling）** | 链路结束后，在 Collector 聚合完整 trace 再决定 | 可基于错误、延迟、Trace 特征做智能保留 | 需 Collector 缓存完整链路，内存/成本高，决策延迟 |

**组合策略（生产推荐）**：

```
入口：TraceIDRatioBased(1%) 头部采样——保证有基础样本
       + ParentBased——保证被采样链路的子服务强制采样（否则链路断）
Collector：尾部采样——额外保留 100% 的 error / P99 慢请求
```

**为什么必须用 `ParentBased`？** 如果 A 服务采样了、B 服务独立按 1% 随机决定不采样，链路就会断在半路——一个 trace 的采样决策必须**全局一致**，靠 traceparent 的 sampled flag 传递。

---

## 五、采样率怎么定（工程实践）

| 场景 | 建议 |
|------|------|
| 日均千万请求 | 头部 1%~5% + 尾部保留全量错误 |
| 低 QPS 核心服务 | 100% 采样（样本太少就失去意义） |
| 高 QPS 内部服务 | 0.1%~1% |
| 排查线上问题 | 临时对该服务调高采样率 / 开定向采样（按 UID、按 Header） |

**成本估算思路**：每天 1 亿请求，1% 采样 = 100 万 trace；每个 trace 平均 20 span → 2000 万 span。存储要按这个量级评估，而不是拍脑袋。

---

## 六、常见坑

| 坑 | 后果 | 处理 |
|----|------|------|
| goroutine 里丢了 ctx | 子 span 变成独立新 trace，链路断 | `go func(ctx context.Context)` 显式传 ctx |
| 异步 MQ 没传 Header | 生产/消费链路断开 | 把 traceparent 写入消息 Header |
| 采样子服务各采各的 | 链路残缺 | 统一 `ParentBased` |
| span 没有 `End()` | 内存泄漏 + 上报缺失 | `defer span.End()` |
| 把用户隐私写进 attributes | 数据合规风险 | 只记 ID/枚举，不记明文手机号 |

---

## 面试话术

> **"链路追踪的核心模型就三个：Trace、Span、Context，TraceID 全局唯一、SpanID 组成父子树。跨进程靠 W3C traceparent Header 传播，跨 goroutine 靠 context.Context 显式传递——这里最容易断链路。Go 里用 OpenTelemetry SDK，TracerProvider 全局初始化一次，Batch 上报，采样用 ParentBased + 概率采样：ParentBased 保证同一条链路的采样决策全局一致，否则子服务各采各的会断链。生产上我一般头部采 1% 打底、Collector 侧尾部采样额外保留全部 error 和慢请求。成本要按 '请求数 × 采样率 × 平均 span 数' 估算，不能拍脑袋。"**
