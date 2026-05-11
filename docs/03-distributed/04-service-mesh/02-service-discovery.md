# 服务注册与发现：注册中心原理 / 客户端 vs 服务端 / 健康检查

> 考察频率：★★★★☆  优先级：P1
> 关键词：Consul、etcd、Nacos、注册中心、健康检查、客户端发现、服务端发现、心跳机制

---

## 面试官考察意图

这道题考察候选人对"服务治理基础设施"的理解深度。
初级只能说出"注册中心用来做服务发现"，高级要能讲清楚**注册中心的选型依据、客户端发现和服务端发现的优劣对比、健康检查的坑（心跳假阳性、探活过度）、以及 Consul/etcd/Nacos 的实际差异**。这是微服务架构的核心基础设施，必考。

---

## 核心答案（30 秒版）

注册中心三问：

| 问题 | 答案 |
|------|------|
| 为什么需要？ | 动态扩缩容时，调用方不知道目标地址，需要一个"中介"来记录和查询 |
| 客户端发现 vs 服务端发现？ | 客户端发现：Consumer 直接查注册中心，延迟低但语言绑定；服务端发现：LB/网关代理，跨语言但多一跳 |
| 健康检查怎么做？ | 注册时带心跳（TTL），注册中心定时检查，失败则剔除；有"假阳性"风险（网络抖动误杀）|

---

## 深度展开

### 1. 为什么需要注册中心

**核心问题：** 微服务扩缩容时，IP 列表动态变化，调用方无法预知。

```
无注册中心：
  Consumer → hardcode: 10.0.0.1:8080
  问题：节点挂了不知道，还往里发请求

有注册中心：
  Provider 启动 → 注册到 注册中心（IP:PORT）
  Consumer 启动 → 从注册中心拉取 Provider 列表
  Provider 挂了 → 心跳失败 → 注册中心剔除
  Consumer 收到更新后的列表
```

---

### 2. 客户端发现 vs 服务端发现

#### 2.1 客户端发现

**原理：** Consumer 直接查询注册中心，获取 Provider 地址，自己做负载均衡。

```
Consumer（Go/Java） → 注册中心（etcd/Consul）
                    ↘ 服务列表
                    ↘ 本地负载均衡（RoundRobin/Random）
                    
Provider（机器1/2/3）
```

**优点：**
- 延迟低：直连，无中间层
- 灵活性高：自己实现负载均衡策略
- 性能好：注册中心压力小（Consumer 缓存）

**缺点：**
- 语言绑定：各语言 SDK 实现不一致
- 更新滞后：Consumer 本地缓存可能过期（CAP 的 AP 问题）
- 治理复杂：Consumer 需要知道注册中心地址

```go
// Go 使用 Consul 做客户端服务发现
package main

import (
    "github.com/go-kratos/consul/registry"
    "github.com/go-kratos/kratos/v2/registry"
)

func main() {
    // 创建 Consul 注册中心客户端
    consulClient, err := consul.NewClient(
        consul.WithAddress("127.0.0.1:8500"),
    )

    // 创建服务发现客户端
    discovery := registry.NewConsulDiscovery(
        consulClient,
        "user-service", // 服务名
        &registry.Options{
            Region:     "sh",
            Zone:       "zone-a",
            DeployEnv:  "prod",
        },
    )

    // 从 Consul 获取所有健康的 user-service 实例
    instances, err := discovery.Fetch(context.Background())
    // instances: [{IP: "10.0.0.1", Port: 8080}, ...]
}
```

#### 2.2 服务端发现

**原理：** Consumer 通过负载均衡器（NGINX/Envoy/Ingress）访问，LB 内部查询注册中心。

```
Consumer → LB（NGINX/Envoy） → 注册中心（获取列表）→ Provider
                              ↗ 缓存
```

**优点：**
- 跨语言：Consumer 无需注册中心 SDK
- 统一入口：可做统一鉴权、限流、灰度
- 缓存友好：LB 批量查询注册中心，Provider 压力小

**缺点：**
- 多一跳：NGINX → Provider，多一次网络跳转
- 单点问题：LB 挂了全站不可用（需要集群化）
- 功能受限：无法做精细的负载均衡策略

**K8s 的服务发现（服务端发现）：**
```yaml
# Service 资源，Kube-proxy 监听 Endpoint 变化，自动更新 iptables
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user
  ports:
    - port: 80
      targetPort: 8080
  # ClusterIP 由 Kube-DNS 解析
  # kube-dns 监听 Service 和 Endpoint 的变化
---
# 应用侧：通过环境变量或 DNS 访问
# 集群内应用通过 user-service.default.svc.cluster.local 访问
```

#### 2.3 对比总结

| 对比项 | 客户端发现 | 服务端发现 |
|-------|---------|---------|
| 延迟 | 低（直连） | 高（多一跳） |
| 跨语言 | ❌（各语言SDK） | ✅（HTTP/DNS） |
| 复杂度 | Consumer 复杂 | LB 复杂 |
| 缓存 | Consumer 本地缓存 | LB 集中缓存 |
| 适用场景 | K8s 外的异构系统 | K8s / 云原生 |
| 代表方案 | Consul + SDK | Kubernetes Service / Envoy + Consul |

---

### 3. 健康检查：心跳与探活

健康检查是注册中心的"眼睛"，决定了一个节点何时被剔除。

#### 3.1 心跳机制

**TTL（Time To Live）心跳：** Provider 定期向注册中心发送心跳，超过 TTL 未收到则视为下线。

```go
// Consul TTL 检查
type RegisterService struct {
    Name    string
    ID      string
    Port    int
    Tags    []string
    Check   *AgentServiceCheck
}

// 注册时带上 TTL 健康检查
check := &AgentServiceCheck{
    TTL: "10s", // 每 10 秒心跳一次
    // 或者用 HTTP 检查
    HTTP: "http://10.0.0.1:8080/health",
    Interval: "5s",
}

client.Agent().ServiceRegister(&RegisterService{
    Name: "user-service",
    ID:   "user-service-01", // 集群内唯一
    Port: 8080,
    Check: check,
})

// Provider 进程需要定期调用 agent.ServicePassTTL 续心跳
client.Agent().ServicePassTTL("user-service-01", "心跳正常")
```

**问题：心跳链路故障 ≠ Provider 故障（假阳性）**

```
Provider 正常（网络正常）
  → 注册中心正常收到心跳 → 不剔除 ✓

Provider 正常（注册中心网络抖动）
  → 注册中心丢失心跳
  → 误剔除 Provider ❌（假阳性）

Provider 挂了（进程崩溃）
  → 无心跳
  → 注册中心正确剔除 ✓
```

**解决方案：**
1. **设置合理 TTL**：太短容易误杀（网络抖动），太长导致故障节点存留太久
2. **连续多次失败才剔除**：而不是一次失败就剔除
3. **探活 + 人工确认**：Consul 支持脚本检查，执行自定义脚本判断是否真的挂了

#### 3.2 健康检查类型

| 类型 | 原理 | 适用场景 |
|------|------|---------|
| **TCP 探活** | 注册中心尝试连接 IP:Port | 服务端点明确 |
| **HTTP 检查** | GET /health，返回 200 才健康 | Web 服务（有 /health 端点）|
| **TTL 检查** | Provider 定期调用续命 API | 无法外部访问的服务 |
| **脚本检查** | 执行自定义脚本，返回 0/1 | 复杂业务逻辑判断 |
| **gRPC 检查** | 发送 HealthCheck RPC | gRPC 服务（Go 1.23+ 原生支持）|

```go
// Go 标准库的 grpc/health 检查
import "google.golang.org/grpc/health/grpc_health_v1"

funcGRPCHealthCheck(conn *grpc.ClientConn) bool {
    client := grpc_health_v1.NewHealthClient(conn)
    resp, err := client.Check(context.Background(), &grpc_health_v1.HealthCheckRequest{
        Service: "user-service",
    })
    return err == nil && resp.Status == grpc_health_v1.HealthCheckResponse_SERVING
}
```

#### 3.3 优雅下线：防止服务剔除延迟

**问题：** Pod 即将被 K8s 终止，但注册中心还不知道，还在路由流量到该 Pod。

**解法：优雅关闭（Graceful Shutdown）**

```go
// Go 服务优雅关闭
func main() {
    // 1. 启动时注册服务
    registerService()

    // 2. 监听 SIGTERM 信号（K8s 终止 Pod 发 SIGTERM）
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        <-sigCh
        log.Println("收到终止信号，开始优雅关闭")

        // Step 1: 停止接收新请求（从注册中心摘除）
        deregisterService() // 立即从 Consul/etcd 注销

        // Step 2: 等待已有请求处理完毕（给 K8s 时间做流量切换）
        // 等待时间要小于 terminationGracePeriodSeconds（默认 30s）
        time.Sleep(10 * time.Second)

        // Step 3: 关闭服务
        srv.Shutdown(context.Background())
    }()

    // 启动 HTTP/gRPC 服务器（设置 WriteTimeout）
    srv := &http.Server{
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 30 * time.Second, // 足够长，让已有请求处理完
        IdleTimeout:  60 * time.Second,
    }
}
```

---

### 4. 注册中心选型：Consul / etcd / Nacos

| 对比项 | Consul | etcd | Nacos |
|-------|--------|------|-------|
| **CAP 模型** | CP（强一致） | CP（强一致） | CP/AP 可配置 |
| **一致性协议** | Raft（Gossip 集群内） | Raft | Raft（配置）/ Distro（注册） |
| **多数据中心** | ✅ 原生支持 WAN Gossip | ❌ 需 federation | ❌ |
| **健康检查** | 丰富（HTTP/TCP/TTL/script） | 只支持 lease 心跳 | 丰富 |
| **KV 存储** | ✅ 强 | ✅ 强 | ✅ 强 |
| **运维复杂度** | 中（Agent + Server 分离） | 低（纯 Go，单二进制） | 中（Java，依赖 MySQL） |
| **Go 生态** | ✅ 成熟（go-kit/consul） | ✅ 成熟（clientv3） | 一般 |
| **适用场景** | 混合云、K8s、DC 感知 | K8s 元数据、配置中心 | 阿里云原生、中小规模 |

```go
// Go 使用 etcd 做注册中心（服务注册 + 发现）
package main

import (
    "context"
    "time"

    "go.etcd.io/etcd/client/v3"
    "go.etcd.io/etcd/client/v3/naming/endpoints"
)

func main() {
    cli, _ := clientv3.NewFromURL("http://localhost:2379")

    // ========== 服务注册 ==========
    em, _ := endpoints.NewManager(cli, "user-service") // 前缀：user-service/

    // 注册实例：10.0.0.1:8080
    em.AddEndpoint(context.TODO(), "user-service/instance-1",
        endpoints.Endpoint{Addr: "10.0.0.1:8080"})

    // 设置租约（TTL），心跳续约
    leaseResp, _ := cli.Grant(context.TODO(), 10)
    em.AddEndpoint(context.TODO(), "user-service/instance-1",
        endpoints.Endpoint{Addr: "10.0.0.1:8080"},
        clientv3.WithLease(leaseResp.ID))

    // 续约心跳（Provider 进程内定期调用）
    go func() {
        ch, _ := cli.KeepAlive(context.TODO(), leaseResp.ID)
        for range ch {
            // 心跳续上
        }
    }()

    // ========== 服务发现 ==========
    // 列出所有实例
    eps, _ := em.List(context.TODO())
    for _, ep := range eps.Endpoints {
        println(ep.Addr)
    }
}
```

---

## 面试话术

**Q：Consul 和 etcd 做注册中心有什么区别？**
> 「Consul 的优势在于多数据中心原生支持（WAN Gossip），适合跨机房、跨云的场景，而且健康检查能力最丰富。etcd 的优势是纯 Go、运维简单，天然和 K8s 共用一个集群，适合 K8s 生态内做服务发现。实际选型看场景：如果是 K8s 里的微服务，用 etcd；如果是异构系统、需要跨 DC，用 Consul。」

**Q：注册中心挂了怎么办？**
> 「注册中心是 CAP 系统，通常选 CP（强一致）。挂了之后的处理分两种：1）Consumer 端有本地缓存，即使注册中心挂了，Consumer 可以用缓存继续工作（只读场景）；2）注册中心不可写入时，新启动的 Provider 无法注册，但已有的 Provider 仍在工作，不会影响正在处理的请求。关键防护是注册中心本身做高可用——etcd 集群 3 节点、Consul 集群 3 节点，任意一个节点挂了不影响整体服务。」

**Q：服务发现时如何避免调用到已下线的服务？**
> 「三层保护：1）Provider 下线前主动注销（graceful deregister），让注册中心立即知道；2）Consumer 端用短 TTL 心跳 + 本地缓存更新，本地缓存 TTL 设为心跳 TTL 的 1/3，这样即使注册中心没及时剔除，Consumer 也能较快更新列表；3）Consumer 调用时做超时 + 重试兜底，如果调用失败（连接超时），从本地缓存剔除该节点，然后重试其他节点。」

---

## 相关文章

- Go 服务注册与注销（实际 SDK 用法）→ `docs/04-microservices/01-rpc/01-grpc.md`
- 熔断与限流 → `docs/03-distributed/04-service-mesh/02-circuit-breaker/02-circuit-breaker.md`