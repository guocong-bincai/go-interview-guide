# API 网关设计：鉴权/路由/限流/熔断一体化

> 考察频率：★★★★☆  优先级：P1  阿里/字节必考
> 关键词：API Gateway、鉴权、动态路由、黑白名单、全局限流、链路追踪、Canary Release

---

## 核心答案（30 秒版）

| 功能 | 职责 | 实现要点 |
|------|------|----------|
| **统一入口** | 所有外部请求经过一个节点 | Nginx / Kong / APISIX / 自研 Go 网关 |
| **鉴权认证** | 校验 Token、API Key、签名 | JWT 验签 + 黑名单检查，拦截非法请求 |
| **协议转换** | HTTP → gRPC 等 | ProtoBuf ↔ JSON 自动序列化 |
| **动态路由** | URL → 后端服务映射 | 配置中心推送，支持正则和通配符匹配 |
| **限流熔断** | 保护后端不被打挂 | 令牌桶限流 + 三态机熔断器 |
| **链路追踪** | 请求全链路 traceID | OpenTelemetry TraceID 注入和透传 |

**生产最佳实践**：APISIX（Go + OpenResty）或自研 Go 网关 + etcd 做配置中心。

---

## 深度展开

### 1. 为什么需要 API 网关？

```
没有网关的混乱架构：              有网关的统一架构：

用户 ──→ Service A:8080          用户 ──→ [API Gateway] ──→ 鉴权 → 限流
用户 ──→ Service B:8081                                    │
用户 ──→ Service C:8082                           ┌────────┼────────┐
                                                   ↓        ↓        ↓
                                              Service A  Service B  Service C
```

**核心价值**：
- ✅ 单一入口：统一入口，避免暴露内部服务拓扑
- ✅ 关注点分离：每个业务服务不用自己写鉴权/限流/日志逻辑
- ✅ 安全隔离：网关在 DMZ 区，内部服务不直接对外

### 2. API 网关核心模块

#### 2.1 鉴权认证（Auth Middleware）

```go
package middleware

import (
    "context"
    "net/http"
    "strings"
    "time"
)

// AuthMiddleware 验证 JWT Token，通过则放行，否则返回 401
type AuthMiddleware struct {
    jwtSecret []byte
    blacklist *jwtBlacklist
}

func NewAuthMiddleware(secret string) *AuthMiddleware {
    return &AuthMiddleware{
        jwtSecret: []byte(secret),
        blacklist: newJWTBlacklist(),
    }
}

func (m *AuthMiddleware) Handler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 白名单路径跳过鉴权
        if isWhiteListed(r.URL.Path) {
            next.ServeHTTP(w, r)
            return
        }

        // 提取 Token
        authHeader := r.Header.Get("Authorization")
        token := extractToken(authHeader)
        if token == "" {
            sendError(w, http.StatusUnauthorized, "missing token")
            return
        }

        // 校验 Token
        claims, err := validateJWT(token, m.jwtSecret)
        if err != nil {
            sendError(w, http.StatusUnauthorized, "invalid token")
            return
        }

        // 检查是否在黑名单中
        if m.blacklist.IsBlocked(claims.UserID) {
            sendError(w, http.StatusForbidden, "user banned")
            return
        }

        // 将用户信息注入上下文
        ctx := context.WithValue(r.Context(), "userID", claims.UserID)
        ctx = context.WithValue(ctx, "roles", claims.Roles)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func extractToken(authHeader string) string {
    parts := strings.Split(authHeader, " ")
    if len(parts) != 2 || parts[0] != "Bearer" {
        return ""
    }
    return parts[1]
}

func sendError(w http.ResponseWriter, code int, msg string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    w.Write([]byte(`{"code":` + fmt.Sprintf("%d", code) + `,"msg":"` + msg + `"}`))
}
```

#### 2.2 动态路由引擎

```go
package router

import (
    "regexp"
    "strings"
)

// Route 表示一条路由规则
type Route struct {
    Pattern   string       // /api/users/:id
    Methods   []string     // ["GET", "POST"]
    Target    string       // http://user-service:8081
    Plugins   map[string]interface{} // rate_limit, auth, etc.
    Priority  int          // 优先级，数值越小越优先
}

// Router 基于 Trie 树的高效路由查找
type Router struct {
    root *node
}

type node struct {
    pattern  string
    route    *Route
    children map[string]*node  // "/" 分隔的子节点
}

func (r *Router) Register(route *Route) {
    n := r.root
    parts := strings.Split(strings.TrimPrefix(route.Pattern, "/"), "/")
    for _, part := range parts {
        if child, ok := n.children[part]; ok {
            n = child
        } else {
            newNode := &node{children: make(map[string]*node)}
            n.children[part] = newNode
            n = newNode
        }
    }
    n.route = route
}

func (r *Router) Find(method, path string) (*Route, map[string]string, bool) {
    parts := strings.Split(strings.TrimPrefix(path, "/"), "/")
    return r.match(r.root, method, parts, 0, make(map[string]string))
}

func (r *Router) match(n *node, method string, parts []string, depth int, params map[string]string) (*Route, map[string]string, bool) {
    if depth == len(parts) {
        if n.route != nil && contains(n.route.Methods, method) {
            return n.route, params, true
        }
        return nil, nil, false
    }

    part := parts[depth]

    // 精确匹配
    if child, ok := n.children[part]; ok {
        if route, p, hit := child.match(child, method, parts, depth+1, params); hit {
            return route, p, true
        }
    }

    // 变量匹配（:id 形式）
    for name, child := range n.children {
        if strings.HasPrefix(name, ":") {
            clonedParams := make(map[string]string)
            for k, v := range params {
                clonedParams[k] = v
            }
            clonedParams[name[1:]] = part // 去掉 ":" 前缀
            if route, p, hit := child.match(child, method, parts, depth+1, clonedParams); hit {
                return route, p, true
            }
        }
    }

    return nil, nil, false
}
```

#### 2.3 全局限流 + 接口级限流

```go
package gateway

import (
    "sync"
    "time"

    "golang.org/x/time/rate"
)

// RateLimiterConfig 限流配置
type RateLimitConfig struct {
    QPS         float64    // 每秒最大请求数
    Burst       int        // 突发容量
    Global      bool       // 是否全局限流
    PathPattern string     // 接口路径匹配模式
}

// GatewayRateLimiter 网关限流器
type GatewayRateLimiter struct {
    mu          sync.RWMutex
    global      *rate.Limiter  // 全局限流
    perPath     map[string]*rate.Limiter // 按接口限流
    configs     []RateLimitConfig
}

func NewGatewayRateLimiter(configs []RateLimitConfig) *GatewayRateLimiter {
    g := &GatewayRateLimiter{
        perPath: make(map[string]*rate.Limiter),
        configs: configs,
    }

    // 创建全局限流器
    for _, cfg := range configs {
        if cfg.Global {
            g.global = rate.NewLimiter(rate.Limit(cfg.QPS), cfg.Burst)
        }
        // 创建各接口限流器
        limiter := rate.NewLimiter(rate.Limit(cfg.QPS), cfg.Burst)
        g.perPath[cfg.PathPattern] = limiter
    }

    return g
}

func (g *GatewayRateLimiter) Allow(requestPath string) bool {
    // 先过全局限流
    if g.global != nil && !g.global.Allow() {
        return false
    }

    // 再过关口限流
    g.mu.RLock()
    limiter, exists := g.perPath[requestPath]
    g.mu.RUnlock()

    if exists && !limiter.Allow() {
        return false
    }

    return true
}
```

#### 2.4 熔断器集成

```go
package gateway

import (
    "github.com/sony/gobreaker"
)

// CircuitBreaker 为每个后端服务维护独立的熔断器
type GatewayCircuitBreaker struct {
    services map[string]*gobreaker.CircuitBreaker
}

func NewCircuitBreaker() *GatewayCircuitBreaker {
    cb := &GatewayCircuitBreaker{
        services: make(map[string]*gobreaker.CircuitBreaker),
    }

    // 默认配置：5次失败后熔断，30秒半开探测
    settings := gobreaker.Settings{
        Name:          "gateway-default",
        MaxRequests:   3,
        Interval:      60 * time.Second,
        Timeout:       30 * time.Second,
        ReadyToTrip:   func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures >= 5
        },
        OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
            log.Printf("[circuit-breaker] %s: %s → %s", name, from, to)
        },
    }

    return cb
}

func (cb *GatewayCircuitBreaker) GetOrRegister(serviceName string) *gobreaker.CircuitBreaker {
    if breaker, ok := cb.services[serviceName]; ok {
        return breaker
    }
    cb.services[serviceName] = gobreaker.NewCircuitBreaker(settings)
    return cb.services[serviceName]
}
```

### 3. 灰度发布 / Canary Release

```
                    ┌─────────────┐
                    │   Gateway   │
                    └──────┬──────┘
                           │
                   ┌───────┼────────┐
                  /        |         \
            90%流量        |     10%流量
            v1版本         |    v2版本（灰度）
           Server A         |    Server B
```

```go
// 基于 Header 的版本路由
func VersionRoutingHandler(next http.Handler) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        version := r.Header.Get("X-Api-Version")

        var targetBackend string
        switch version {
        case "v2":
            targetBackend = "https://api-v2.example.com"  // 新版本
        default:
            targetBackend = "https://api.example.com"    // 默认版本
        }

        // 通过反向代理转发到对应版本
        proxy := &reverseProxy{
            Director: func(req *http.Request) {
                req.URL.Scheme = "https"
                req.URL.Host = strings.TrimPrefix(targetBackend, "https://")
                req.Host = strings.TrimPrefix(targetBackend, "https://")
            },
        }
        proxy.ServeHTTP(w, r)
    }
}

// 更高级的做法：基于客户端特征的渐进发布
type CanayReleaseMatcher struct {
    percentage int // 0-100
    whitelist  map[string]bool // 白名单用户ID
}

func (m *CanayReleaseMatcher) IsV2(user string) bool {
    if m.whitelist[user] {
        return true
    }
    // 随机采样
    rand.Seed(time.Now().UnixNano())
    return rand.Intn(100) < m.percentage
}
```

### 4. 响应聚合与批量处理

```go
// 场景：首页需要同时调用用户信息、商品列表、推荐三个微服务
// 传统方式串行调用：总耗时 = t1 + t2 + t3
// 网关层并行聚合：总耗时 = max(t1, t2, t3)

type BatchResponse struct {
    User     *UserDTO     `json:"user"`
    Products []ProductDTO `json:"products"`
    Recs     []RecItem    `json:"recommendations"`
}

func (gw *Gateway) AggregateHomePage(ctx context.Context, userID string) (*BatchResponse, error) {
    type result struct {
        data interface{}
        err  error
    }

    ch := make(chan result, 3)

    // 并发调用三个下游服务
    go func() {
        user, err := gw.userSvc.GetUser(ctx, userID)
        ch <- result{data: user, err: err}
    }()

    go func() {
        products, err := gw.productSvc.ListNewProducts(ctx)
        ch <- result{data: products, err: err}
    }()

    go func() {
        recs, err := gw.recSvc.RecommendForUser(ctx, userID)
        ch <- result{data: recs, err: err}
    }()

    // 收集结果（带超时控制）
    resp := &BatchResponse{}
    timer := time.NewTimer(500 * time.Millisecond)

    var pending int
    for i := 0; i < 3; i++ {
        select {
        case r := <-ch:
            if r.err != nil {
                // 降级：某个服务挂了不影响其他服务的返回
                pending--
                continue
            }
            switch d := r.data.(type) {
            case *UserDTO:
                resp.User = d
            case []ProductDTO:
                resp.Products = d
            case []RecItem:
                resp.Recs = d
            }
            pending--
        case <-timer.C:
            // 超时，返回部分数据
            break
        }
    }

    return resp, nil
}
```

### 5. 面试话术

**Q：你们为什么要自建网关而不是直接用 K8s Ingress？**

> Ingress 主要解决的是四层负载均衡和健康检查，但对于 API 网关需要的业务级能力（如 JWT 鉴权、动态路由、接口级限流、协议转换）支持有限。我们最初用的 Nginx Ingress Controller，后来需要更多定制化的鉴权逻辑和灰度发布策略时才迁移到了自研 Go 网关。网关的核心优势是：可以在代码层面灵活编排各种中间件链，而不需要依赖外部配置。

**Q：网关如何处理高可用问题？**

> 网关本身不能成为单点故障。我们部署了至少 3 个网关实例，前面加 LVS/Nginx 做四层负载均衡。网关是无状态的，不保存会话信息——所有的 Token 校验都是无状态的 JWT 验签，路由配置通过 etcd 或 Nacos 实时分发。这样任意一个网关实例挂掉，流量会自动切到其他实例上。另外，网关后面通常还有熔断保护，如果某个后端服务不可用，网关侧的熔断器会快速拒绝请求，防止雪崩。
