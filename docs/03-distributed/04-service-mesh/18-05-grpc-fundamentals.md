# gRPC 基础与原理：协议、流式调用、拦截器

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：HTTP/2、Protobuf、四种子类型流、gRPC vs REST、拦截器、负载均衡

---

## 面试官考察意图

这道题考察候选人对**微服务间通信基础设施**的理解。
初级只能说出"gRPC 比 HTTP 快"，高级要能讲清楚 **HTTP/2 的 multiplexing 优势、四种流式调用的区别和适用场景、interceptor 如何用作切面（鉴权/日志/链路追踪）、以及 gRPC 的负载均衡策略**。在 Go 微服务场景中，gRPC 是事实标准 RPC 框架。

---

## 核心答案（30 秒版）

**gRPC = Protobuf 序列化 + HTTP/2 传输 + 强类型 IDL。**

核心优势：
- **性能高：** HTTP/2 二进制协议 + Protobuf 紧凑编码，体积极小
- **多路复用：** 一个 TCP 连接可并发多个请求，无队头阻塞
- **强类型约束：** .proto 文件自动生成代码，编译期检查错误

四种 RPC 模式：
| 类型 | 客户端 → 服务端 | 服务端 → 客户端 | 场景 |
|------|---------------|---------------|------|
| **Unary** | 1 请求 → 1 响应 | 1 响应 | 最简单，同步调用 |
| **Server Streaming** | 1 请求 | N 响应 | 大数据集分批返回 |
| **Client Streaming** | N 请求 | 1 响应 | 大文件分片上传 |
| **Bidirectional Streaming** | N 请求 | N 响应 | 实时双向通信 |

---

## 深度展开

### 一、gRPC 与 HTTP/2 的关系

```
传统 REST (HTTP/1.1):
Client ────req────▶ Server    ← TCP 连接 1
Client ◀──res─────┘           ← 每个请求需要独立连接或复用

gRPC (HTTP/2):
Client ════stream 1═══▶ Server   ← 同一 TCP 连接
        ════stream 2═══▶          ← multiplexing，无队头阻塞
        ════stream 3════◀ Server
        ════stream 4════◀
```

#### HTTP/2 带来的关键改进

| 特性 | HTTP/1.1 | HTTP/2 | 影响 |
|------|---------|--------|------|
| 传输格式 | 文本 | 二进制 | 解析更快，更安全 |
| 多路复用 | ❌ 需 HTTP Keep-Alive | ✅ 原生支持 | 消除队头阻塞 |
| 头部压缩 | ❌ | HPACK 压缩 | 减少重复 Header 体积 |
| 服务器推送 | ❌ | ✅ Server Push | 预加载资源 |
| 流优先级 | ❌ | ✅ | 按重要性调度 |

**gRPC 为什么选择 HTTP/2 而不是 HTTP/3？**
- HTTP/3 基于 QUIC，2018 年才成熟，生态支持不如 HTTP/2
- HTTP/2 + TLS 已经足够好，延迟开销可控
- gRPC-Web 可以通过 proxy 桥接到 HTTP/1.1（浏览器限制不支持 HTTP/2）

---

### 二、Protobuf 序列化

```protobuf
// hello.proto
syntax = "proto3";
package greet;

option go_package = "./pb";

service Greeter {
    rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
    string name = 1;      // field number 必须从 1 开始，唯一
    int32 age = 2;
    repeated string tags = 3;  // repeated = slice
}

message HelloResponse {
    string message = 1;
    bool success = 2;
}
```

**生成代码：**
```bash
protoc --go_out=. --go-grpc_out=. hello.proto
```

**vs JSON 的核心差异：**

| 维度 | Protobuf | JSON |
|------|----------|------|
| 二进制/文本 | 二进制（紧凑） | 文本（可读） |
| 序列化速度 | 极快（结构映射） | 较慢（解析+反射） |
| 体积 | 约为 JSON 的 1/5~1/10 | 较大 |
| 向后兼容 | ✅ field number 不变即可 | ⚠️ 新增字段可能破坏逻辑 |
| Schema 定义 | 有 .proto 强类型约束 | 无 |
| IDE 支持 | 代码自动生成 | 需手写结构体 |

**Proto3 的坑：**
- `optional` 语法：默认值（如 `0`、空字符串）无法与"未设置"区分——用 wrapper 类型（`google.protobuf.StringValue`）解决
- `repeated` 空列表序列化为空数组（而非缺失字段），反序列化时可能丢失语义

---

### 三、四种流式 RPC

#### 1. Unary（普通调用）

```go
func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloResponse, error) {
    return &pb.HelloResponse{Message: "Hello " + req.Name}, nil
}
```

#### 2. Server Stream（服务端流式）

```go
// proto: rpc FlowHello(FlowRequest) returns (stream FlowResponse);

func (s *server) FlowHello(req *pb.FlowRequest, stream pb.Greeter_FlowHelloServer) error {
    for _, name := range req.Names {
        stream.Send(&pb.FlowResponse{Message: "Hello " + name})
        time.Sleep(time.Second)  // 模拟延迟，每条消息逐步返回
    }
    return nil
}
```

**适用场景：** 
- 大批量数据分批返回（数据库查询结果集）
- 实时事件推送（监控告警流）

**Go 端接收：**
```go
client, err := greeterClient.FlowHello(ctx, &pb.FlowRequest{Names: []string{"A", "B", "C"}})
if err != nil { log.Fatal(err) }
for {
    resp, err := client.Recv()
    if err == io.EOF { break }
    if err != nil { log.Fatal(err) }
    fmt.Println(resp.Message)
}
```

#### 3. Client Stream（客户端流式）

```go
// proto: rpc RecordHello(stream CountRequest) returns (CountResponse);

func (s *server) RecordHello(stream pb.Greeter_RecordHelloServer) error {
    count := 0
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&pb.CountResponse{Count: int32(count)})
        }
        if err != nil { return err }
        count++
    }
}
```

**适用场景：**
- 大文件上传（分块发送）
- 批量操作聚合计数

#### 4. Bidirectional Stream（双向流式）⭐ 高频考点

```go
// proto: rpc Chat(stream Message) returns (stream Message);

func (s *server) Chat(stream pb.Greeter_ChatServer) error {
    done := make(chan struct{})
    
    // 接收协程
    go func() {
        for {
            req, err := stream.Recv()
            if err == io.EOF { close(done); return }
            if err != nil { log.Fatal(err) }
            fmt.Printf("收到: %s\n", req.Content)
        }
    }()
    
    // 发送协程
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    for {
        select {
        case <-done:
            return
        case t := <-ticker.C:
            stream.Send(&pb.Message{Content: "时间: " + t.Format(time.RFC3339)})
        }
    }
}
```

**实际应用场景：**
- 即时通讯（聊天室）
- 在线游戏状态同步
- 实时协同编辑
- WebSocket 替代方案（gRPC 的流式天然适合长连接场景）

---

### 四、Interceptor（拦截器/中间件）

gRPC 的 interceptor 类似 HTTP middleware，用于实现**横切关注点**（鉴权、日志、链路追踪、重试）。

#### Server-side Interceptor

```go
type authInterceptor struct{}

func (i *authInterceptor) UnaryInterceptor(ctx context.Context, req interface{}, 
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    // 鉴权
    token, ok := metadata.FromIncomingContext(ctx)
    if !ok || len(token["authorization"]) == 0 {
        return nil, status.Error(codes.Unauthenticated, "missing token")
    }
    
    fmt.Printf("[INFO] %s called by authenticated user\n", info.FullMethod)
    
    // 继续处理
    return handler(ctx, req)
}

// 注册
opts := []grpc.Option{
    grpc.StreamInterceptor(...),
    grpc.UnaryInterceptor(auth{}.UnarIntercepto()),
}
srv := grpc.NewServer(opts...)
```

**常用拦截器链：**
```
客户端请求
  │
  ▼
┌─────────────────┐
│ MetricsInterceptor│  ← 记录耗时、QPS
├─────────────────┤
│ TraceInterceptor │  ← 注入 TraceID，上报链路
├─────────────────┤
│ AuthInterceptor  │  ← 校验 Token
├─────────────────┤
│ RateLimitInterceptor│  ← 限流
├─────────────────┤
│   Handler        │  ← 业务逻辑
└─────────────────┘
```

#### Client-side Interceptor

```go
conn, _ := grpc.Dial("localhost:50051", 
    grpc.WithUnaryInterceptor(grpc_retry.UnaryClientInterceptor(
        grpc_retry.WithMax(3),
        grpc_retry.WithBackoff(grpc_retry.BackoffExponential(100*time.Millisecond)),
    )),
)
```

---

### 五、gRPC 负载均衡

#### L4 负载均衡（transport-level）

gRPC 客户端自带简单 L4 负载均衡，基于 **pickfirst** 和 **round_robin** 策略：

```go
import _ "google.golang.org/grpc/resolver/dns"

// round_robin: 轮询选择后端地址
conn, _ := grpc.Dial("dns:///my-service.local:50051",
    grpc.WithDefaultServiceName("my-service"),
)

// pickfirst: 选第一个健康的地址
conn, _ := grpc.Dial("dns:///my-service.local:50051",
    grpc.WithLoadBalancingPolicy("pick_first"),
)
```

**重要：** gRPC 内置 LB 只支持**健康实例间的均匀分发**，不支持权重/会话保持/路径匹配等高级策略。生产环境中通常配合 **Nginx/Istio/K8s Service** 做更精细的路由。

#### 与服务网格结合

```
Client (gRPC with round_robin)
       │
       ▼
   ┌─────────┐     Istio Sidecar
   │ Envoy LB │ ── 支持权重路由、故障转移、重试、超时
   └─────────┘
       │
       ▼
   k8s Pod (你的服务)
```

---

### 六、gRPC vs REST 选型建议

| 维度 | gRPC | REST |
|------|------|------|
| 性能 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 开发体验 | 强类型+自动生成 | 灵活但手工维护 |
| 跨语言 | ✅ 所有语言 | ✅ 所有语言 |
| 浏览器支持 | ❌ 需要 gRPC-Web + proxy | ✅ 原生支持 |
| 缓存 | ❌ HTTP 缓存不友好 | ✅ 天然支持 |
| SEO | ❌ 不适合 | ✅ GET 可索引 |
| 调试 | 需要专用工具（grpcurl） | curl + 浏览器 |
| 文档 | Swagger/OpenAPI 生态完善 | OpenAPI/Swagger 最成熟 |

**选型原则：**
- **内部微服务通信** → gRPC（性能好，强类型约束）
- **对外开放 API** → REST + JSON（兼容性好，浏览器直接调用）
- **IoT/移动端** → gRPC（节省带宽和电量）
- **SEO 友好的网站** → REST（Google 能索引 GET 请求）

---

## 面试话术模板

> "我们内部服务全用 gRPC 通信，好处是 Protobuf 体积小、HTTP/2 多路复用效率高。我们做了四层 interceptor：日志→鉴权→链路追踪→指标上报。对外暴露 REST API，通过 Gateway 做 gRPC↔REST 转换。流式场景用得最多的就是 server stream（比如实时数据推送）。"

---

📌 **扩展阅读：**
- [grpc-go](https://github.com/grpc/grpc-go) 官方源码解读
- HTTP/2 规范 RFC 7540
- Protobuf v3 vs v2 的区别
