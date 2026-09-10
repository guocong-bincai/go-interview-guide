# API Gateway 架构设计：原理、选型与 Go 实现

> 考察频率：★★★★★  优先级：P0（微服务必考，逢面必问）
> 关键词：路由转发、鉴权认证、限流熔断、插件化、Sidecar vs Library、负载均衡

---

## 面试官考察意图

这道题考察候选人对 **微服务架构入口层** 的系统理解。
初级只知道"Gateway 就是统一入口"，高级要能讲清楚 **API Gateway 的完整职责边界、Go 层面的手写实现方案、与 Sidecar Proxy（Envoy/Kong）的选型权衡、动态路由的实现原理、以及网关的高可用架构**。这是从后端工程师跳到系统架构师的必经之路。

---

## 核心答案（30秒版）

**API Gateway = 所有外部请求的统一入口 + 横切关注点的集中处理。**

五大核心职责：
1. **路由分发**：按 URL/Host/Header 将请求转发到下游微服务
2. **身份鉴权**：JWT/OAuth2 Token 验证、权限校验
3. **流量治理**：限流、熔断、降级（保护下游不被打垮）
4. **可观测性**：日志记录、指标收集、链路追踪（TraceID 注入）
5. **协议转换**：HTTP→gRPC、WebSocket↔MQTT、JSON↔Protobuf

**核心选型权衡**：
| 维度 | Library（Gin/Echo 内嵌） | Sidecar（Envoy/Kong/Istio） |
|------|------------------------|---------------------------|
| 性能 | 高（无额外网络跳数） | 略低（多一跳 TCP） |
| 语言无关 | ❌ 只支持 Go | ✅ 任何语言都可接入 |
| 配置热更新 | 需重启或复杂实现 | 原生支持（xDS API） |
| 运维复杂度 | 低（就是一个 HTTP Server） | 高（需要 K8s + Istio 体系） |
| 适用场景 | 中小规模 Go 微服务集群 | 大规模异构服务网格 |

---

## 深度展开

### 1. API Gateway 的职责边界

#### 1.1 应该放在 Gateway 层的

```
┌──────────────────────────────────────────────────────┐
│              API Gateway                             │
│                                                      │
│  ✅ 通用横切功能：                                    │
│  ├── 🔐 认证鉴权（JWT Verify / OAuth2 Introspect）    │
│  ├── 🚦 限流（全局限速 / IP限速 / 用户限速）         │
│  ├── 🔍 链路追踪（注入 TraceID / SpanID）             │
│  ├── 📝 访问日志（统一格式输出）                       │
│  ├── 🔄 HTTPS 卸载 / TLS 终止                        │
│  ├── 🛡️ WAF / XSS / SQL Injection 防护               │
│  └── 📊 指标收集（QPS / 延迟 / 错误率 → Prometheus）  │
│                                                      │
│  ⚠️ 业务相关功能（按需放置）：                         │
│  ├── 协议转换（HTTP ↔ gRPC）                         │
│  ├── 缓存代理（读请求直接返回缓存结果）                │
│  ├── BFF 聚合（前端需要多个服务的数据，合并返回）      │
│  └── 格式适配（不同客户端版本适配不同响应结构）         │
│                                                      │
│  ❌ 不应该放在 Gateway 层的：                          │
│  ├── 核心业务逻辑（订单创建、支付处理等）              │
│  ├── 数据持久化操作                                  │
│  └── 跨服务的业务编排（用 Saga/Camunda 等编排引擎）   │
└──────────────────────────────────────────────────────┘
```

#### 1.2 为什么不能把所有逻辑都塞进 Gateway？

```
错误做法：把 50+ 个微服务的业务逻辑都搬到 Gateway
后果：
1. Gateway 变成单体巨兽，无法独立扩展
2. 任何一个接口故障都会影响整个 Gateway
3. 部署耦合——改一个业务就要重新部署 Gateway
4. 性能瓶颈——Gateway 成为单点

正确做法：Gateway 只做"门面"，业务逻辑留在各自的服务中
```

---

### 2. Go 手写 API Gateway 实现

#### 2.1 最小可用 Gateway（基于 Chi Router）

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "time"

    "github.com/go-chi/chi/v5"
)

// ServiceRegistry 存储可用的微服务地址
type ServiceRegistry struct {
    services map[string]string // routePrefix -> upstreamURL
}

func (r *ServiceRegistry) GetUpstream(route string) (string, bool) {
    // 精确匹配优先，其次前缀匹配
    if url, ok := r.services[route]; ok {
        return url, true
    }
    for prefix, url := range r.services {
        if len(route) >= len(prefix) && route[:len(prefix)] == prefix {
            return url, true
        }
    }
    return "", false
}

// Gateway 是 API Gateway 的核心实现
type Gateway struct {
    registry     *ServiceRegistry
    httpClient   *http.Client
    rateLimiter  RateLimiter     // 限流器
    authChecker  AuthMiddleware  // 鉴权中间件
    maxBodySize  int64           // 最大请求体大小
    readTimeout  time.Duration
    writeTimeout time.Duration
}

// NewGateway 创建 Gateway 实例
func NewGateway(registry *ServiceRegistry) *Gateway {
    return &Gateway{
        registry:   registry,
        httpClient: &http.Client{
            Timeout: 10 * time.Second,
        },
        maxBodySize:  10 << 20,          // 10MB
        readTimeout:  15 * time.Second,
        writeTimeout: 15 * time.Second,
    }
}

// ServeHTTP 实现 http.Handler 接口
func (g *Gateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 1. 鉴权（检查 JWT Token）
    if err := g.authChecker.Validate(r); err != nil {
        http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
        return
    }

    // 2. 限流
    if !g.rateLimiter.Allow(r.RemoteAddr) {
        http.Error(w, `{"error":"rate limited"}`, http.StatusTooManyRequests)
        return
    }

    // 3. 路由查找
    serviceURL, ok := g.registry.GetUpstream(r.URL.Path)
    if !ok {
        http.Error(w, `{"error":"not found"}`, http.StatusNotFound)
        return
    }

    // 4. 构建上游请求
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()

    upstreamReq, err := http.NewRequestWithContext(ctx, r.Method, serviceURL+r.URL.Path, r.Body)
    if err != nil {
        http.Error(w, `{"error":"internal error"}`, http.StatusInternalServerError)
        return
    }

    // 5. 复制必要 Headers
    upstreamReq.Header = make(http.Header)
    for k, vv := range r.Header {
        if k != "Host" {
            upstreamReq.Header[k] = vv
        }
    }
    upstreamReq.Host = r.Host

    // 6. 转发请求
    resp, err := g.httpClient.Do(upstreamReq)
    if err != nil {
        log.Printf("upstream request failed: %v", err)
        http.Error(w, `{"error":"service unavailable"}`, http.StatusBadGateway)
        return
    }
    defer resp.Body.Close()

    // 7. 复制响应状态和 Headers
    for k, vv := range resp.Header {
        w.Header()[k] = vv
    }
    w.WriteHeader(resp.StatusCode)

    // 8. 流式转写 Body（避免内存放大）
    io.Copy(w, resp.Body)
}
```

#### 2.2 中间件链式调用

```go
// Middleware 定义
type Middleware func(http.Handler) http.Handler

// Chain 组装中间件
func Chain(ms ...Middleware) Middleware {
    return func(next http.Handler) http.Handler {
        for i := len(ms) - 1; i >= 0; i-- {
            next = ms[i](next)
        }
        return next
    }
}

// 鉴权中间件
func AuthMiddleware(tokenProvider TokenProvider) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := extractToken(r)
            if token == "" {
                http.Error(w, `{"error":"missing token"}`, http.StatusUnauthorized)
                return
            }
            claims, err := tokenProvider.Verify(token)
            if err != nil {
                http.Error(w, `{"error":"invalid token"}`, http.StatusUnauthorized)
                return
            }
            // 将用户信息存入 Context
            ctx := context.WithValue(r.Context(), "user", claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// 日志中间件
func LoggingMiddleware(logger *log.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            responseWriter := &responseWriterWrapper{ResponseWriter: w, statusCode: 200}
            
            next.ServeHTTP(responseWriter, r)
            
            logger.Printf("%s %s %s %d %v",
                r.RemoteAddr, r.Method, r.URL.Path,
                responseWriter.statusCode,
                time.Since(start),
            )
        })
    }
}

// 限流中间件
func RateLimitMiddleware(limiter RateLimiter) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow(r.RemoteAddr) {
                http.Error(w, `{"error":"rate limited"}`, http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// 使用示例：
// middleware := Chain(
//     LoggingMiddleware(log.Default()),
//     AuthMiddleware(tokenProvider),
//     RateLimitMiddleware(rateLimiter),
// )
// chiHandler := router.NewRouter()
// // ... 注册路由 ...
// http.ListenAndServe(":8080", middleware(chiHandler))
```

---

### 3. 动态路由配置

静态配置的路由表在微服务扩缩容时不够灵活。工业级 Gateway 通常支持动态路由：

```go
// 从 Consul / etcd 获取服务列表并动态刷新
type DynamicRouter struct {
    client     ServiceDiscoveryClient
    cache      map[string][]UpstreamEndpoint
    refreshCh  chan struct{}
}

func (r *DynamicRouter) Refresh() error {
    endpoints, err := r.client.GetAllEndpoints("order-service")
    if err != nil {
        return err
    }
    
    // 简单的健康检查过滤
    healthy := make([]UpstreamEndpoint, 0, len(endpoints))
    for _, ep := range endpoints {
        if r.healthCheck(ep) {
            healthy = append(healthy, ep)
        }
    }
    
    r.cache["order-service"] = healthy
    return nil
}

// RoundRobin 负载均衡选择上游
func (r *DynamicRouter) NextEndpoint(serviceName string) (*UpstreamEndpoint, error) {
    endpoints, ok := r.cache[serviceName]
    if !ok || len(endpoints) == 0 {
        return nil, fmt.Errorf("no healthy endpoints for %s", serviceName)
    }
    
    // 简单轮询（生产环境用一致性哈希或带权重的）
    atomic.AddUint64(&r.counter, 1)
    idx := atomic.LoadUint64(&r.counter) % uint64(len(endpoints))
    
    return &endpoints[idx], nil
}
```

---

### 4. Gateway vs Sidecar Proxy 对比

#### 4.1 架构差异

```
Library 模式（Go 内嵌 Gateway）：
┌───────────┐    ┌──────────────────┐
│ Client    │    │                  │
└─────┬─────┘    │   API Gateway    │  ← Go 进程，和业务同进程
      │ HTTP     │  （Chi/Gin）     │
      ├────────▶ │                  │
      │          └────────┬─────────┘
      │                   │ HTTP/gRPC
      │          ┌────────▼─────────┐
      │          │   Order Service  │
      │          │   Payment Svc    │
      └─────────▶│   User Service   │
                 └──────────────────┘

Sidecar 模式（K8s Pod + Envoy）：
┌──────────────────────────────────┐
│ Pod                              │
│  ┌────────┐   ┌──────────────┐   │
│  │App     │◀─▶│ Envoy Sidecar│   │ ← 独立 sidecar 容器
│  │(业务)  │   │ (L7 Proxy)   │   │
│  └────────┘   └──────────────┘   │
└──────────────────────────────────┘
```

#### 4.2 选型决策矩阵

| 因素 | 选 Library | 选 Sidecar |
|------|----------|----------|
| **团队规模** | <50 人 | >100 人 |
| **语言栈** | 纯 Go | 多语言异构 |
| **K8s 基础设施** | 可选 | 已有 |
| **运维能力** | 一般 | 强（有 K8s + Istio 团队） |
| **网关 QPS** | <10万 | >10万 |
| **安全合规要求** | 标准 | 极高（需要 mTLS、审计） |
| **预算成本** | 低 | 高（Istio 资源开销大） |

**经验法则**：如果你们的 Go 微服务集群 < 50 个服务、总 QPS < 5万、团队没有专职的 Platform Engineering，那么用 Go 内嵌 Gateway 就够了。上 Envoy/Istio 的成本远大于收益。

---

### 5. 面试高频追问

**Q：Gateway 挂了怎么办？**

> Gateway 本身就是单点，必须做多副本部署 + 健康检查。具体策略分两层：
> 1. **横向扩展**：K8s HPA 根据 CPU/QPS 自动扩容，至少保证 3~5 个副本。
> 2. **兜底降级**：如果全部 Gateway 不可用，可以通过修改 DNS/Balancer 直接把请求打到后端服务——虽然少了鉴权和限流，但核心功能可用。这就是所谓的 "fail-open" 策略。

**Q：你写的这个 Gateway 有什么性能瓶颈？**

> 三个主要瓶颈：
> 1. **Body 拷贝**：`io.Copy` 是全量拷贝，如果 upstream 返回大文件会导致 Gateway 内存放大。解决方案是用 `io.TeeReader` 或直接流式透传，或者限制 maxResponseSize。
> 2. **并发连接数**：单个 Gateway 进程的并发受限于 GOMAXPROCS，高并发场景下需要水平扩展。
> 3. **DNS 解析**：每次请求都做 DNS 解析有延迟。应该用本地缓存，配合 Consul 的 watch 机制做增量更新。

**Q：你在什么场景下会考虑用 Kong / Envoy 而不是自己写的？**

> 当满足以下任一条件时：
> - 微服务超过 50 个且多语言混合
> - 需要动态配置变更而不重新部署（Envoy 的 xDS API 可以做到毫秒级更新）
> - 有严格的 mTLS 和安全审计需求
> - 团队已经有成熟的 K8s + Istio 平台，复用现成组件更划算

---

## 面试话术

**Q：怎么设计一个微服务 API Gateway？**

> 我会从三层来设计：
> 
> **第一层：接入层**——Nginx/ALB 做 L7 负载均衡，HTTPS 卸载，DDoS 基础防护。这一步确保请求可靠到达 Gateway。
> 
> **第二层：Gateway 层**——Go 实现的 API Gateway，负责路由分发、JWT 鉴权、IP限速、日志记录和链路追踪注入。每个微服务有自己的路由前缀（如 `/api/order/*` → order-service）。
> 
> **第三层：服务发现层**——Consul 或 etcd 维护服务注册表，Gateway 定期拉取最新的 endpoint 列表并缓存，结合健康检查过滤掉不健康的实例。
> 
> 🗣️ **记忆口诀**：**"先LoadBalancer再接入，鉴权限流加日志，服务发现靠Consul，横向扩到能抗住"**

---

*参考文档：[go-chi/chi](https://github.com/go-chi/chi) · [Kong Gateway](https://docs.konghq.com/) · [Envoy Documentation](https://www.envoyproxy.io/docs/envoy/) · [Service Mesh Design Patterns](https://microservices.io/patterns/service-mesh.html)*

🏠 [首页](../../../README.md) · 📦 [分布式系统](../README.md)
