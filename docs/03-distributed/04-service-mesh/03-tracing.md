# 链路追踪

> 考察频率：★★★☆☆  难度：★★★★☆
> 关键词：TraceID、Span、OpenTelemetry、Jaeger、采样策略

## 链路追踪核心概念

```
完整调用链：
═══════════════════════════════════════════════════════

Trace: 全链路唯一次标识（一次请求从头到尾）
  │
  ├── Span: 操作单元（一个函数/一次 RPC）
  │     span-1: handleOrder (0ms ──────────────── 100ms)
  │              │
  │              ├── span-2: queryDB (10ms ─── 50ms)
  │              │
  │              └── span-3: callPay (55ms ─── 90ms)
  │                        │
  │                        └── span-4: httpCall (60ms ─ 85ms)

一个 Trace = 多个 Span 组成的有向无环图（DAG）
每个 Span 包含：name、start-time、end-time、parent-span-id、attributes
```

---

## OpenTelemetry（推荐标准）

### 核心组件

| 组件 | 说明 |
|------|------|
| **OTLP** | 传输协议（OpenTelemetry Protocol） |
| **SDK** | 各语言的实现 |
| **Collector** | 接收、处理、导出数据 |
| **Jaeger/Zipkin** | 后端存储和 UI |

### Go + OpenTelemetry 示例

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.4.0"
	"go.opentelemetry.io/otel/trace"
)

var tracer trace.Tracer

func initTracer() (func(), error) {
	// 1. 配置 OTLP exporter（发送到 Collector）
	ctx := context.Background()
	exporter, err := otlptracegrpc.New(ctx,
		otlptracegrpc.WithEndpoint("localhost:4317"),
		otlptracegrpc.WithInsecure(), // 无 TLS（测试环境）
	)
	if err != nil {
		return nil, fmt.Errorf("create exporter: %w", err)
	}

	// 2. 创建 TracerProvider
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("order-service"),
			semconv.ServiceVersionKey.String("1.0.0"),
			attribute.String("env", "production"),
		)),
		// 采样策略
		sdktrace.WithSampler(sdktrace.TraceIDRatioBasedSampler(0.1)), // 10% 采样
	)

	// 3. 注册全局 TracerProvider
	otel.SetTracerProvider(tp)
	tracer = tp.Tracer("order-service")

	return func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		tp.Shutdown(ctx)
	}, nil
}

// 业务处理函数
func handleOrder(ctx context.Context, orderID string) error {
	// 从 HTTP Header 提取 TraceID（如果存在）
	ctx, span := tracer.Start(ctx, "handleOrder",
		trace.WithAttributes(
			attribute.String("order.id", orderID),
		),
	)
	defer span.End()

	// 查询数据库
	ctx, dbSpan := tracer.Start(ctx, "queryDB")
	err := queryUser(ctx, orderID)
	dbSpan.End()
	if err != nil {
		span.RecordError(err)
		return err
	}

	// 调用支付服务
	ctx, paySpan := tracer.Start(ctx, "callPayment")
	err = callPayment(ctx, orderID)
	paySpan.End()
	if err != nil {
		span.RecordError(err)
		return err
	}

	// 发消息
	ctx, mqSpan := tracer.Start(ctx, "sendMQ")
	err = sendOrderEvent(ctx, orderID)
	mqSpan.End()

	return nil
}

func queryUser(ctx context.Context, orderID string) error {
	_, span := trace.Start(ctx, "SELECT * FROM orders WHERE id=?")
	defer span.End()
	
	span.SetAttributes(
		attribute.String("db.system", "mysql"),
		attribute.String("db.statement", "SELECT * FROM orders WHERE id=?"),
	)
	
	// 模拟 DB 操作
	time.Sleep(10 * time.Millisecond)
	return nil
}

func callPayment(ctx context.Context, orderID string) error {
	_, span := trace.Start(ctx, "HTTP POST /pay")
	defer span.End()
	
	span.SetAttributes(
		attribute.String("http.method", "POST"),
		attribute.String("http.url", "http://payment-service/pay"),
		attribute.String("http.status_code", "200"),
	)
	
	// 模拟 RPC
	time.Sleep(20 * time.Millisecond)
	return nil
}

func sendOrderEvent(ctx context.Context, orderID string) error {
	_, span := trace.Start(ctx, "kafka send")
	defer span.End()
	
	span.SetAttributes(
		attribute.String("messaging.system", "kafka"),
		attribute.String("messaging.destination", "order-events"),
	)
	
	return nil
}

func main() {
	shutdown, err := initTracer()
	if err != nil {
		log.Fatal(err)
	}
	defer shutdown()

	ctx := context.Background()
	handleOrder(ctx, "order-001")
}
```

---

## TraceID 传播

### HTTP 传播（W3C TraceContext）

```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net/http"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel"
)

// HTTP 传播：自动注入和提取 TraceID
func httpPropagation() {
	// 设置 Propagator（W3C 标准）
	otel.SetTextMapPropagator(propagation.NewCompositeKeyPropagator(
		propagation.TraceContext{}, // W3C TraceContext
		propagation.Baggage{},
	))

	// 创建带 trace 的 HTTP 客户端
	client := &http.Client{
		Transport: otelhttp.NewTransport(http.DefaultTransport),
	}

	// 自动注入 TraceID 到 Header
	req, _ := http.NewRequest("GET", "http://localhost:8080/api", nil)
	resp, err := client.Do(req)
	if err != nil {
		log.Fatal(err)
	}
	defer resp.Body.Close()

	fmt.Println("Response:", resp.Status)
}

// 服务端提取 TraceID
func httpServer() {
	// 设置全局 Propagator
	otel.SetTextMapPropagator(propagation.NewCompositeKeyPropagator(
		propagation.TraceContext{},
	))

	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// otelhttp 中间件自动提取 TraceID 并创建 Root Span
		ctx := r.Context()
		
		// 手动提取也可以
		propagator := otel.GetTextMapPropagator()
		ctx = propagator.Extract(ctx, propagation.HeaderCarrier(r.Header))
		
		// 从 context 获取当前 span
		span := trace.SpanFromContext(ctx)
		fmt.Printf("Received request, TraceID: %s\n", span.SpanContext().TraceID())
		
		w.Write([]byte("OK"))
	})

	// 包装中间件
	http.Handle("/api", otelhttp.NewHandler(handler, "api"))
	http.ListenAndServe(":8080", nil)
}

// gRPC 传播
func grpcPropagation() {
	// gRPC 有专门的 otelgrpc 包
	// 会自动在 metadata 中传播 TraceID
}
```

---

## 采样策略

### 常见采样策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **AlwaysOn** | 全部采集 | 测试环境 |
| **AlwaysOff** | 全不采集 | 关闭追踪 |
| **TraceIDRatio** | 按比例采样 | 高流量生产环境 |
| **ParentBased** | 父 Span 采样 | 根请求采样，子请求跟随 |
| **TailSampling** | 尾部采样（先缓存再判断） | 需要保留异常请求 |

```go
package main

import (
	"go.opentelemetry.io/otel/sdk/trace"
	"go.opentelemetry.io/otel/sdk/trace/tracetest"
)

// 自定义采样器：保留慢请求和错误请求
type AdaptiveSampler struct {
	thresholdMS int64
}

func (s *AdaptiveSampler) ShouldSample(p trace.SamplingParameters) trace.SamplingResult {
	// 慢请求 (>500ms) 必采
	if p.ParentEndTime.UnixMilli()-p.StartTime.UnixMilli() > s.thresholdMS {
		return trace.SamplingResult{Decision: trace.RecordAndSample }
	}
	// 有错误的请求必采
	for _, attr := range p.Attributes {
		if attr.Key == "error" && attr.Value.AsBool() {
			return trace.SamplingResult{Decision: trace.RecordAndSample }
		}
	}
	// 其他按比例
	if p.TraceID[0]&0x07 == 0 { // 12.5% 采样
		return trace.SamplingResult{Decision: trace.RecordAndSample }
	}
	return trace.SamplingResult{Decision: trace.Drop}
}

func (s *AdaptiveSampler) Description() string {
	return "AdaptiveSampler"
}
```

---

## Jaeger 部署

```yaml
# docker-compose.yml
version: '3'
services:
  jaeger:
    image: jaegertracing/all-in-one:1.50
    ports:
      - "16686:16686"   # UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
    environment:
      COLLECTOR_OTLP_ENABLED: true
```

```bash
# 访问 Jaeger UI
open http://localhost:16686
```

---

## 常见问题排查

```bash
# 1. 查看某 Trace 的耗时分布
# Jaeger UI → Search → 点击 Trace → Waterfall 图

# 2. 常见慢调用定位
# - db.query 耗时高 → 检查索引 / 连接池
# - http.call 耗时高 → 检查下游服务 / 网络
# - redis.call 耗时高 → 检查 bigkey / 网络

# 3. 错误率关联
# span 中有 error=true 的属性 → 在 Jaeger 中 filter error:true
```

---

## 面试话术

**Q：TraceID 是怎么在线程/goroutine 间传递的？**

> 通过 Context（Go）或 ThreadLocal（Java）传递。根请求进来时中间件从 HTTP Header 或 gRPC Metadata 里提取 TraceID，创建一个 Root Span，然后把它塞进 context 往下游传。下游服务从 context 里取 Span，继续创建子 Span。整个链路所有 Span 共享同一个 TraceID，通过 ParentSpanID 串联成树形结构。

**Q：全量采集 Trace 数据量太大怎么办？**

> 采样。常见策略：1）固定比例采样（如 1% 或 10%）；2）优先采集慢请求和错误请求；3）尾部采样（Tail-based）：先缓存所有 Span，等 Trace 完成后根据结果决定是否保存，好处是不会漏掉异常 Trace，但需要额外的缓存资源。生产环境推荐 ParentBased + 固定比例，异常请求比例设高一些。

**Q：OpenTracing 和 OpenTelemetry 是什么关系？**

> OpenTracing 是较早的社区标准（CNCF），定义了一套与厂商无关的 API。OpenTelemetry（OTel）是 OpenTracing + OpenCensus 合并后的新标准（也是 CNCF），功能更全，涵盖了 Traces、Metrics、Logs。**新项目直接用 OpenTelemetry**，老项目可以迁移。Jaeger、Zipkin 都支持 OTLP 协议。
