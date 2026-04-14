# API 网关设计

> 考察频率：★★★★☆  难度：★★★★☆

## 网关核心职责

```
                                    ┌──────────────────┐
                                    │   API Gateway     │
  Client ──────────────────────────→ │                  │
                                    │  ① 路由转发       │
                                    │  ② 鉴权认证       │
                                    │  ③ 限流熔断       │
                                    │  ④ 协议转换       │
                                    │  ⑤ 灰度发布       │
                                    │  ⑥ 日志监控       │
                                    └───────┬──────────┘
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    │                       │                       │
            ┌───────▼───────┐       ┌───────▼───────┐       ┌───────▼───────┐
            │  User Svc     │       │  Order Svc   │       │  Product Svc  │
            │   :8081       │       │   :8082       │       │   :8083       │
            └───────────────┘       └───────────────┘       └───────────────┘
```

### 网关 vs 反向代理

| 维度 | Nginx | API 网关 |
|------|-------|---------|
| 协议 | HTTP/TCP | HTTP/REST/gRPC/Dubbo |
| 功能 | 路由/限流 | 鉴权/协议转换/业务逻辑 |
| 复杂度 | 低 | 高 |
| 适用场景 | 入口负载均衡 | 微服务出口/入口统一管控 |

---

## 核心功能详解

### 1. 路由转发

根据请求路径/参数转发到对应后端服务。

```go
// 基于 path 的路由
type Route struct {
    Path    string
    Method  string
    Backend string // upstream 地址
}

var routes = []Route{
    {"/api/user/*", "GET", "http://user-service:8081"},
    {"/api/order/*", "POST", "http://order-service:8082"},
    {"/api/product/*", "GET", "http://product-service:8083"},
}

func router(c *gin.Context) {
    path := c.Request.URL.Path
    for _, r := range routes {
        if matchPath(r.Path, path) && c.Request.Method == r.Method {
            proxyPass(c, r.Backend, path)
            return
        }
    }
    c.JSON(404, gin.H{"error": "not found"})
}

func matchPath(pattern, path string) bool {
    // 简化实现：支持通配符
    // 实际用 gorilla/mux 或 httprouter
    return strings.HasPrefix(path, strings.TrimSuffix(pattern, "*"))
}
```

### 2. 鉴权认证

#### JWT 验证

```go
func authMiddleware(c *gin.Context) {
    token := c.GetHeader("Authorization")
    if token == "" {
        c.AbortWithStatus(401)
        return
    }

    // Bearer token
    token = strings.TrimPrefix(token, "Bearer ")

    claims, err := validateJWT(token)
    if err != nil {
        c.AbortWithStatus(401)
        return
    }

    // 注入用户信息到 context
    c.Set("userID", claims.UserID)
    c.Set("roles", claims.Roles)
    c.Next()
}

func validateJWT(token string) (*Claims, error) {
    parsedToken, err := jwt.Parse(token, func(t *jwt.Token) (interface{}, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method")
        }
        return []byte(jwtSecret), nil
    })
    if err != nil {
        return nil, err
    }

    if claims, ok := parsedToken.Claims.(jwt.MapClaims); ok && parsedToken.Valid {
        return &Claims{
            UserID: claims["user_id"].(string),
            Roles:  claims["roles"].([]interface{}),
        }, nil
    }
    return nil, errors.New("invalid token")
}
```

#### API Key 验证

```go
func apiKeyMiddleware(c *gin.Context) {
    apiKey := c.GetHeader("X-API-Key")
    if apiKey == "" {
        c.AbortWithStatus(401)
        return
    }

    // 验证 API Key（通常存 Redis）
    valid, err := rdb.HExists(ctx, "api_keys", apiKey).Result()
    if err != nil || !valid {
        c.AbortWithStatus(403)
        return
    }

    // 限权范围
    scopes := rdb.HGet(ctx, "api_key:scopes", apiKey).Result()
    c.Set("scopes", scopes)
    c.Next()
}
```

### 3. 限流

详见 [06-rate-limiter](../05-system-design/06-rate-limiter/06-rate-limiter.md)

网关层限流通常用滑动窗口或令牌桶：

```go
// 滑动窗口限流（按用户/接口）
func rateLimitMiddleware(c *gin.Context) {
    userID := c.GetString("userID")
    key := fmt.Sprintf("rate:user:%s:%s", userID, c.Request.URL.Path)

    allowed, err := redisRateLimit(key, 100, time.Minute) // 100次/分钟
    if err != nil {
        log.Printf("rate limit error: %v", err)
        c.Next() // 失败时放过
        return
    }

    if !allowed {
        c.AbortWithStatusJSON(429, gin.H{"error": "rate limit exceeded"})
        return
    }
    c.Next()
}
```

### 4. 灰度发布（Canary Release）

#### 按流量比例灰度

```go
func canaryMiddleware(c *gin.Context) {
    // 10% 流量走灰度
    if rand.Float64() < 0.1 {
        c.Set("backend", "http://order-service-canary:8082")
    } else {
        c.Set("backend", "http://order-service:8082")
    }
    c.Next()
}

// nginx canary 配置
// upstream backend {
//     server order-service:8082 weight=90;
//     server order-service-canary:8082 weight=10;
// }
```

#### 按用户灰度

```go
func canaryByUserMiddleware(c *gin.Context) {
    userID := c.GetString("userID")

    // VIP 用户走灰度
    isVIP := rdb.SIsMember(ctx, "vip_users", userID).Val()
    if isVIP {
        c.Set("backend", "http://order-service-canary:8082")
    } else {
        c.Set("backend", "http://order-service:8082")
    }
    c.Next()
}
```

### 5. 协议转换（REST → gRPC）

```go
func restToGrpc(c *gin.Context) {
    // REST: POST /user {"name": "tom"}
    // → gRPC: CreateUser(CreateUserRequest)

    body, _ := io.ReadAll(c.Request.Body)
    var req map[string]interface{}
    json.Unmarshal(body, &req)

    grpcReq := &pb.CreateUserRequest{
        Name:  req["name"].(string),
        Email: req["email"].(string),
    }

    // 调用 gRPC 服务
    resp, err := grpcClient.CreateUser(ctx, grpcReq)
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }

    c.JSON(200, gin.H{
        "id":    resp.Id,
        "name":  resp.Name,
        "email": resp.Email,
    })
}
```

---

## Go 网关实现（基于 Gin）

### 完整中间件链

```go
func main() {
    r := gin.New()

    // 全局中间件
    r.Use(gin.Recovery())
    r.Use(loggingMiddleware())

    // 路由分组：需要鉴权
    auth := r.Group("/api")
    auth.Use(authMiddleware())
    auth.Use(rateLimitMiddleware())
    {
        auth.GET("/user/:id", proxyTo("http://user-service:8081"))
        auth.POST("/order", proxyTo("http://order-service:8082"))
    }

    // 公开接口
    r.GET("/health", healthCheck)

    r.Run(":8080")
}

func proxyTo(backend string) gin.HandlerFunc {
    return func(c *gin.Context) {
        proxy := httputil.NewSingleHostReverseProxy(backendURL(backend))
        c.Request.URL.Path = c.Param("path") // 路径重写
        proxy.ServeHTTP(c.Writer, c.Request)
    }
}
```

---

## 主流网关对比

| 网关 | 语言 | 特点 | 适用场景 |
|------|------|------|---------|
| Kong | Lua/NGINX | 插件丰富 | 通用 |
| APISIX | Lua/NGINX | 支持多协议 | 云原生 |
| Traefik | Go | 自动服务发现 | K8s |
| Spring Cloud Gateway | Java | Spring 生态 | Java 微服务 |
| Envoy | C++ | Service Mesh 核心 | 高度定制 |

---

## 面试话术

**Q：网关和 Sidecar 的区别？**

> 网关是中心化的，所有流量经过一个（或少数几个）网关节点；Sidecar（Envoy）是去中心化的，每个服务实例旁边都有一个 Sidecar，流量在本地就知道路由。Sidecar 更适合 K8s 环境，网关更适合传统微服务架构。

**Q：网关挂了怎么办？**

> 1）网关本身要部署多实例 + 健康检查；2）客户端侧做重试 + 回退；3）核心路径可以绕过网关直连（fallback）。关键是网关要轻量，尽量少的业务逻辑，减少故障点。

**Q：如何防止攻击者绕过网关？**

> 1）后端服务只允许网关 IP 访问（网络隔离）；2）网关统一颁发临时 Token，后端验证 Token 而不是直接暴露接口；3）重要接口加生物特征绑定（设备指纹 + UserID 绑定）。
