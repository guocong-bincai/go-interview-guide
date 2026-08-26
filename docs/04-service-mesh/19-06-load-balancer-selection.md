# 负载均衡选型：Nginx vs HAProxy vs Envoy vs Traefik

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：L4/L7、调度算法、WASM、Service Mesh、Kubernetes Gateway API

## 核心答案（30 秒版）

**四层（L4）**用 HAProxy（TCP/UDP，性能最优），**七层（L7）复杂路由**选 Nginx/Apache APISIX（成熟生态），**云原生/Service Mesh 场景**选 Envoy（Kubernetes 标配）。Go 语言实现的 **Traefik** 适合微服务动态发现。

| | Nginx | HAProxy | Envoy | Traefik |
|---|---|---|---|---|
| **语言** | C | C | C++ | Go |
| **层级** | L4+L7 | L4+L7 | L4+L7 | L4+L7 |
| **配置方式** | conf 文件 | cfg/haproxy.cfg | YAML/gRPC | toml/entrypoints |
| **服务发现** | 手动 + lua | 脚本驱动 | xDS API | Docker/K8s 内置 |
| **插件生态** | Lua / WASM | Lua | WASM / FilterChain | Middleware |
| **监控** | Prometheus + OpenTelemetry | 内置 stats | 强大（Metrics/Tracing） | Dashboard + Metrics |
| **TLS 终止** | ✅ | ✅ | ✅ | ✅(自动 HTTPS) |
| **适用场景** | Web 网关、API 网关 | TCP/UDP LB | Service Mesh | K8s Ingress |

---

## 深度展开

### 1. Nginx：老牌王者

```nginx
# nginx.conf — 负载均衡配置示例
http {
    upstream backend {
        # 轮询（默认）— 顺序分发
        least_conn;     # 最少连接数
        
        server 10.0.0.1:8080 weight=3;
        server 10.0.0.2:8080 weight=2;
        server 10.0.0.3:8080 backup;  # 备用节点
        
        keepalive 32;   # 保持上游长连接
    }
    
    server {
        listen 80;
        
        location /api/ {
            proxy_pass http://backend;
            
            # 健康检查（需要 nginx_upstream_check_module 或 openresty）
            health_check interval=5 fails=3 passes=2 uri=/health;
            
            # 限流
            limit_req zone=api burst=10 nodelay;
        }
    }
}
```

**Nginx 核心优势**：事件驱动模型（epoll/kqueue）、高并发能力（万级到十万级 QPS）、成熟的 HTTP 模块生态、Lua 扩展（OpenResty 支持动态逻辑）。

### 2. HAProxy：四层之王

```haproxy
# haproxy.cfg — TCP 负载均衡配置
global
    maxconn 50000
    log stdout format raw local0
    
defaults
    mode tcp              # 纯四层模式（最快）
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend tcp_front
    bind *:5000
    default_backend app_servers

backend app_servers
    balance roundrobin       # 轮询
    # balance leastconn      # 最少连接（长连接场景推荐）
    # balance first          # 填充空的后端
    # balance source         # 源地址 hash（IP 绑定）
    
    option tcp-check         # TCP 健康检查
    tcp-check connect
    tcp-check send PING\r\n
    tcp-check expect string PONG
    
    server srv1 10.0.0.1:8080 check inter 3s fall 3 rise 2
    server srv2 10.0.0.2:8080 check inter 3s fall 3 rise 2
    server srv3 10.0.0.3:8080 check inter 3s fall 3 rise 2
```

**HAProxy 的核心优势**：极低的延迟开销（纯 C 实现）、丰富的调度算法、四层代理（可直接转发 TCP 而不解析 HTTP）、连接复用池。

### 3. Envoy：云原生时代的新星

```yaml
# envoy.yaml — 基础配置
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match: { prefix: "/api" }
                route:
                  cluster: app_cluster
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: app_cluster
    load_assignment:
      cluster_name: app_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: 10.0.0.1, port_value: 8080 }
        - endpoint:
            address:
              socket_address: { address: 10.0.0.2, port_value: 8080 }
    circuit_breakers:
      thresholds:
      - priority: DEFAULT
        max_connections: 10000
        max_pending_requests: 10000
    health_checks:
    - timeout: 5s
      interval: 10s
      healthy_threshold: 1
      unhealthy_threshold: 3
      http_health_check:
        path: /health
```

**Envoy 的独特优势**：
① **xDS API**（CDS/RDS/EDS/LDS）支持动态配置，无需重启；
② **内置分布式追踪**（Jaeger/Datadog 集成）；
③ **高级熔断器**——基于成功/失败比例的 Circuit Breaker；
④ **重试策略**可配置（`retriable_status_codes`, `retry_on`, `num_retries`）；
⑤ **WASM 过滤器**——用 Rust/Go 编写高性能自定义逻辑。

### 4. Traefik：Kubernetes 首选

```toml
# traefik.toml
[entryPoints]
  [entryPoints.web]
    address = ":80"

[providers.kubernetesIngress]
  publishedService = "traefik/traefik"

[routing]
  [[routing.routers]]
    entryPoints = ["web"]
    rule = "Host(`example.com`) && PathPrefix(`/api`)"
    service = "api-service"

[services]
  [[services.api-service.loadBalancer.healthCheck.path]]
    path = "/health"
```

---

## 调度算法对比

```go
// Go 中常见的负载均衡算法实现

// 1. Round Robin（轮询）
type RoundRobin struct {
	index int
	servers []string
	mu sync.Mutex
}

func (rr *RoundRobin) Next() string {
	rr.mu.Lock()
	defer rr.mu.Unlock()
	srv := rr.servers[rr.index%len(rr.servers)]
	rr.index++
	return srv
}

// 2. Least Connections（最少连接）
type LeastConn struct {
	mu       sync.Mutex
	servers  map[string]int // server → active connections
}

func (lc *LeastConn) Next() string {
	lc.mu.Lock()
	defer lc.mu.Unlock()
	
	minConn := math.MaxInt32
	var best string
	for s, c := range lc.servers {
		if c < minConn {
			minConn = c
			best = s
		}
	}
	lc.servers[best]++
	return best
}

// 3. Consistent Hashing（一致性哈希，会话保持）
type ConsistentHash struct {
	keys   []uint32
	nodes  map[uint32]string
	hashFn func(string) uint32
}

func (ch *ConsistentHash) Get(key string) string {
	h := ch.hashFn(key)
	idx := sort.Search(len(ch.keys), func(i int) bool { return ch.keys[i] >= h })
	if idx == len(ch.keys) { idx = 0 }
	return ch.nodes[ch.keys[idx]]
}

// 4. Weighted Round Robin（加权轮询）
// 按权重分配请求，权重越高分配的请求越多
```

---

## 选型决策树

```
你的场景需要什么？
├── 纯 TCP/UDP 转发，追求极致性能
│   └── ✅ HAProxy
├── HTTP/HTTPS 反向代理，经典 Web 架构
│   ├── 需要 WAF/复杂 rewrite → ✅ Nginx + OpenResty
│   └── 简单反代 → ✅ Nginx
├── Kubernetes 环境，动态服务发现
│   ├── 已有 Istio/Linkerd → ✅ Envoy（已内置）
│   └── 轻量 Ingress Controller → ✅ Traefik / Nginx Ingress
├── 需要动态路由规则、无重启热更新
│   └── ✅ Envoy（gRPC xDS API）
└── API 网关（认证/限流/鉴权一体化）
    ├── 高性能 + 多协议 → ✅ Apache APISIX
    ├── Java 生态 → ✅ Kong / Spring Cloud Gateway
    └── Go 微服务 → ✅ Traefik / Go-http-server
```

---

## 性能基准参考

```
| 指标           | Nginx | HAProxy | Envoy | Traefik |
|---------------|-------|---------|-------|---------|
| 纯 L4 吞吐    | ~12w  | ~12w    | ~9w   | ~6w     |
| L7 复杂路由   | ~5w   | ~6w     | ~7w   | ~4w     |
| TLS 终止      | ~4w   | ~4w     | ~5w   | ~3w     |
| CPU 利用率    | 低    | 最低    | 中    | 低      |
| 内存占用      | 中    | 低      | 较高  | 低      |
```

*注：数据仅供参考，实际性能取决于硬件、负载类型和配置调优。*

---

## 面试高频追问

**Q：为什么 Kubernetes 默认选择 Envoy 做 Sidecar？**

> Envoy 提供了完整的 xDS API，可以动态下发路由、监听、集群配置而无需重启。它内置了分布式追踪、健康检查、熔断器等功能，与 Istio 的服务网格架构天然契合。此外 Envoy 的过滤链机制允许开发者用 WASM 编写自定义逻辑，灵活性远超 Nginx 的 Lua 方案。

**Q：Nginx 和 HAProxy 怎么选？**

> 如果只需要**四层 TCP/UDP 负载均衡**，HAProxy 更合适——它的协议栈更简单，几乎没有额外开销。如果需要**七层 HTTP 路由、URL 重写、gzip 压缩、SSL/TLS 终止**等高级功能，Nginx 更好。实际上很多架构是两层叠加：Nginx 处理 L7 路由，后端用 HAProxy 做 L4 分流。

**Q：什么是 xDS？为什么它比静态配置好？**

> xDS 是一套配置 API（Cluster Discovery Service, Route Discovery Service 等），由 Control Plane 通过 gRPC 向 Proxy 推送配置。相比 Nginx 读配置文件 reload 的方式，xDS 实现了**零停服热更新**。Control Plane（如 Istio Pilot）实时感知服务变更，立刻推送新配置给所有 Envoy 实例。

---

## 面试话术

**Q：你们的系统用了什么负载均衡？怎么选的？**

> 我们的架构是两层：入口处用 **Nginx** 做七层路由和限流，内部分发用 **HAProxy** 做四层转发——HAProxy 在纯 TCP 模式下几乎没额外开销。Kubernetes 内部流量走 **Envoy Sidecar**，因为需要动态路由和链路追踪。选型原则很直接：简单的场景用最简单的工具，有高级需求的才上复杂方案。**不要为了追热点而用复杂工具，复杂度本身就是成本**。

🗣️ **记忆口诀**：**"四层找 HAProxy，L7 找 Nginx，K8s 全家桶就是 Envoy"**

---

*官方文档：[Nginx Docs](https://nginx.org/en/docs/) · [HAProxy Tech](https://www.haproxy.com/documentation/) · [Envoy Config](https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/basic)*

[🏠 首页](../../../README.md) · [📦 分布式系统](../README.md) · [💎 服务治理](../../docs/03-distributed/04-service-mesh/README.md)
