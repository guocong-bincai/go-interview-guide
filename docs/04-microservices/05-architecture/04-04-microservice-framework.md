# Go 微服务框架选型：go-zero vs Kratos vs Kitex

> 考察频率：★★★★☆  难度：★★★☆☆  关键词：框架选型、代码生成、中间件、性能与生态

## 一、为什么面试官爱问框架选型

选型题考的不是"你会用哪个"，而是**你能不能把技术选型讲成一次有约束的决策**：团队规模、技术栈、性能要求、生态成熟度、迁移成本各占什么权重。答题套路：**先讲约束，再讲候选，最后给结论 + 代价**。

---

## 二、三大主力框架对比

| 维度 | **go-zero** | **Kratos** | **Kitex** |
|------|-------------|------------|-----------|
| 出品方 | 好未来 | B站 | 字节跳动 |
| 核心理念 | 约定优于配置、极简全家桶 | 大而全的微服务套件、DDD 友好 | 高性能 RPC 框架（专注通信层） |
| 代码生成 | `goctl` 一把梭（api/rpc/model） | `kratos` 脚手架 + protoc | `kitex` 代码生成 |
| 传输协议 | HTTP + gRPC | HTTP + gRPC | 自研 TTHeader/HTTP2（gRPC 兼容） |
| 性能 | 中高 | 中高 | **最高**（字节场景打磨） |
| 治理能力 | 内置限流/熔断/自适应降载 | 中间件 + 生态 contrib（otel 等） | 靠外部（Polaris 等） |
| 学习曲线 | 平缓，文档中文友好 | 中等，概念多（wire/DDD 分层） | 偏底层，需自己拼装 |
| 适合 | 中小团队快速起服务 | 中大型团队、重视规范与可扩展 | 高并发、极致性能、自研治理体系 |

**一句话记忆**：

- **go-zero**：小团队"开箱即用"，api/rpc/model 一套生成，集成度和开发效率最高。
- **Kratos**：B站生产级全家桶，分层清晰、中间件生态好、和 OpenTelemetry/Prometheus 集成顺畅。
- **Kitex**：性能最好、最底层，适合自己掌控治理栈的大厂场景，通常搭配 K8s + 自研控制面。

---

## 三、选型的四个判断维度

1. **团队规模与规范**：5 人以下要效率 → go-zero；有平台组、要统一规范 → Kratos。
2. **性能天花板**：QPS 万级以下，框架差异远小于 DB/缓存瓶颈；十万级、极致 P99 → Kitex。
3. **生态与可观测性**：是否原生对接 OpenTelemetry、Prometheus、链路追踪（Kratos 这块最顺）。
4. **迁移成本**：框架是**长期负债**，中途换框架几乎等于重写，所以选型要看 3 年后的团队规模。

---

## 四、手写最小中间件：理解框架的"治理内核"

无论哪个框架，治理能力都以**中间件/拦截器链**的形式实现。以 gRPC 为例：

```go
// 服务端一元拦截器链：日志 → 链路追踪 → 限流 → 业务
func ChainInterceptors(interceptors ...grpc.UnaryServerInterceptor) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
		// 逆序包装，形成洋葱模型：谁先注册谁最外层
		build := handler
		for i := len(interceptors) - 1; i >= 0; i-- {
			ic := interceptors[i]
			next := build
			build = func(ctx context.Context, req any) (any, error) {
				return ic(ctx, req, info, next)
			}
		}
		return build(ctx, req)
	}
}

// 敏感接口限流中间件（令牌桶）
func RateLimitMiddleware(limiter *rate.Limiter) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
		if !limiter.Allow() {
			return nil, status.Error(codes.ResourceExhausted, "rate limited")
		}
		return handler(ctx, req)
	}
}
```

**要点**：中间件顺序很关键——**限流要放在链路追踪之后、业务之前**（这样被限流的请求也能被追踪到）；**Recover 要在最外层**（否则 panic 会穿透）；**日志/指标要在最外层**（统计全量耗时）。

---

## 五、面试延伸

**Q：为什么 Go 微服务框架不像 Java Spring Cloud 那样大而全？**
A：Go 的哲学是组合优于继承、库优于框架；而且云原生时代很多能力（服务发现、LB、配置）下沉到了 K8s，框架只需管好 RPC + 治理中间件，不需要一个重量级容器。

**Q：框架里的"自适应降载"（go-zero 的 adaptive load shedding）是什么？**
A：基于 CPU 负载和请求延迟动态判断，当系统接近过载时**主动丢弃**一部分请求（返回 503），保住核心请求的 P99，避免"雪崩式排队"。本质是**背压（backpressure）**思想。

---

## 面试话术

> **"框架选型我按四步讲：团队规模、性能要求、可观测生态、迁移成本。小团队要效率选 go-zero，开箱即用、goctl 一把梭；中大型团队重视规范和可观测性选 Kratos，B站生产验证、OpenTelemetry 集成顺；追求极致性能和自研治理选 Kitex，代价是要自己拼装治理栈。但要提醒的是：框架是长期负债，中途换等于重写，所以要看三年后的团队规模来定。另外框架的治理能力本质上都是拦截器链，顺序上要保证 Recover 最外层、限流在追踪之后。"**
