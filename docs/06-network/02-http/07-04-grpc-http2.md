# gRPC 基于 HTTP/2 的多路复用、流控与帧格式

> 考察频率：★★★☆☆  难度：★★★★☆
> 关键词：HTTP/2 帧、Stream Multiplexing、Stream Priority、Flow Control、gRPC 帧

---

## 核心答案（30 秒版）

gRPC 构建在 **HTTP/2** 之上，HTTP/2 的核心能力是**多路复用**：一条 TCP 连接上并行多个 Stream，每个 Stream 内有序，多个 Stream 间乱序。

**HTTP/1.1 的问题**：每个请求需要独占一个 TCP 连接（线头阻塞），或者用管道队列（队头阻塞依然存在）。

**HTTP/2 的解决方案**：
```
TCP 连接 1
    ↓
┌──────────────────────────────────────────┐
│ Stream 1: GET /users/1                   │
│ Stream 2: GET /orders         并行复用   │
│ Stream 3: POST /orders                   │
│ Stream 4: GET /products/1                │
└──────────────────────────────────────────┘
```

gRPC 进一步在 HTTP/2 Stream 上定义了自己的**调用模式**（一元调用、流式调用）。

---

## 1. HTTP/2 核心概念

### 1.1 Stream 与 Frame

- **Stream**：一个双向字节流，在一个 HTTP/2 连接中并行存在多个 Stream（ID 唯一）
- **Frame**：HTTP/2 的最小传输单元，有 `HEADERS` 帧（存 Header）和 `DATA` 帧（存 Body）

```
连接（Connection）
    ↓
Stream 1 (ID=1)
    ├── HEADERS (Stream 1, :method=GET, :path=/a)
    ├── DATA (Stream 1, body...)
    └── HEADERS (Stream 1, :status=200) + DATA (Stream 1, body...)
        ↓
Stream 3 (ID=3)
    ├── HEADERS (Stream 3, :method=POST, :path=/b)
    ├── DATA (Stream 3, body...)
    └── HEADERS (Stream 3, :status=201)
        ↓
Stream 5 (ID=5)
    ...
```

每个 Stream 有唯一的奇数/偶数 ID（客户端发起的为奇数，服务端发起的为偶数）。

### 1.2 HPACK 头部压缩

HTTP/2 用 **HPACK** 压缩 Header，避免每次请求都带完整的 Cookie/User-Agent 等大 Header：

```
静态表（41 种常见 Header）：如 :method: GET, :status: 200
动态表：本次连接中出现过的 Header
哈夫曼编码：高频字符用短编码
```

典型请求压缩效果：
- HTTP/1.1 无压缩：~500-800 字节
- HTTP/2 + HPACK：~50-100 字节

### 1.3 Flow Control（流量控制）

HTTP/2 在**连接级别**和**Stream 级别**都有流量控制：

```go
// 接收方可以告诉发送方：别发太多，等我准备好
WINDOW_UPDATE frame：窗口扩大（允许发送更多）
SETTINGS frame：连接建立时的初始窗口大小（默认 64KB）
```

这避免了快速发送方压垮慢速接收方的问题（TCP 本身有流量控制，但那是端到端的，HTTP/2 在应用层多了一层控制）。

### 1.4 Stream Priority（优先级）

Stream 可以设置权重（1-256），高权重的 Stream 优先获得带宽：

```
Stream 1 (weight=100, 依赖 Stream 3)
Stream 2 (weight=200)  → 优先获得带宽
Stream 3 (weight=50)
```

gRPC 用这个实现**流式调用的优先级**。

---

## 2. HTTP/2 帧格式

```
+-----------------------------------------------+
|  Length (24 bits) | Type (8) | Flags (8) | R |
+-------------------+-----------------+-----------+
|           Stream Identifier (31 bits)          |
+-----------------------------------------------+
|                   Frame Payload                |
+-----------------------------------------------+
```

| 字段 | 说明 |
|------|------|
| Length | 帧负载长度（最大 16KB）|
| Type | HEADERS(0x1), DATA(0x0), SETTINGS(0x4), WINDOW_UPDATE(0x9), ... |
| Flags | END_STREAM, END_HEADERS, PADDED, ... |
| Stream ID | 所属的 Stream（0=连接级别帧）|

### 常用帧类型

| Type | 用途 |
|------|------|
| `0x0` DATA | 传输请求/响应body，可带padding |
| `0x1` HEADERS | 传输 Header（伪头部字段）|
| `0x4` SETTINGS | 连接参数（初始窗口大小等）|
| `0x5` PING | 心跳，测量 RTT |
| `0x7` GOAWAY | 优雅关闭，说明错误码 |
| `0x8` WINDOW_UPDATE | 流量控制窗口更新 |
| `0x9` CONTINUATION | HEADERS 帧分片续传 |

---

## 3. gRPC 的 HTTP/2 使用方式

### 3.1 gRPC 调用类型

```
Unary RPC（一元 RPC）：
  Client → Headers + Data (请求) → Server
  Client ← Headers + Data (响应) ← Server

Server Streaming RPC（服务端流）：
  Client → Headers + Data (请求) → Server
  Client ← HEADERS (流开始)
  Client ← DATA (消息1)
  Client ← DATA (消息2)
  ...
  Client ← DATA (最后一条) + HEADERS (trailers)

Client Streaming RPC（客户端流）：
  Client → HEADERS (流开始)
  Client → DATA (消息1)
  Client → DATA (消息2)
  ...
  Client → DATA (最后一条)
  Client ← HEADERS + DATA (响应)

Bidirectional Streaming RPC（双向流）：
  双方都可以在任何时候发送 DATA 帧
```

### 3.2 gRPC 帧格式

gRPC 在 HTTP/2 DATA 帧之上定义了 `Length-Prefixed-Message`：

```
┌─────────────────────────────────────┐
│   压缩标志 (1 byte)  │  消息长度 (4 bytes, big-endian)  │
├─────────────────────────────────────┤
│              消息体 (protobuf 编码)           │
└─────────────────────────────────────┘
```

### 3.3 gRPC 特殊的 HTTP/2 使用

**HEADERS 帧**：
```
:method = POST
:path = /package.Service/MethodName
:scheme = http/https
:authority = server.com
te = trailers（表明支持 trailer 编码）
content-type = application/grpc
grpc-encoding = gzip/snappy/br（压缩算法）
```

**Trailers**（在 RPC 结束时）：
```
grpc-status = 0（0=成功，非0=错误）
grpc-message = "description"
```

### 3.4 Go gRPC 请求/响应流程

```go
// 客户端发送一个 Unary 请求的 HTTP/2 帧序列
// 1. HEADERS (Stream Begin, Request Headers)
// 2. DATA (Request Body, Compressed Flag + Length + Protobuf)
// 3. HEADERS (Stream End, Trailers)

// 服务端响应
// 1. HEADERS (Response Headers)
// 2. DATA (Response Body)
// 3. HEADERS (Stream End, Trailers: grpc-status, grpc-message)
```

---

## 4. 多路复用避免线头阻塞

### 4.1 HTTP/1.1 的问题

```
连接 1：GET /a → 等 2 秒（慢）← 阻塞了
连接 2：GET /b → 必须等连接1释放才能发
```

HTTP/1.1 浏览器通常开 6 个并发连接，但每条连接只能串行处理请求。

### 4.2 HTTP/2 真多路复用

```
连接 1：
  Stream 1: GET /a（等 2 秒）
  Stream 2: GET /b（立即发出，不用等）
  Stream 3: POST /c（立即发出，不用等）
```

所有 Stream 共用一条 TCP 连接，乱序并发。解决了 HTTP/1.1 的队头阻塞。

### 4.3 TCP 层的队头阻塞

HTTP/2 的多路复用解决了应用层的队头阻塞，但 TCP 本身有拥塞控制+丢包重传。如果丢了一个包，TCP 会等重传，导致**这条 TCP 连接上所有 Stream 都卡住**（TCP 层的队头阻塞）。

这就是为什么 QUIC（HTTP/3 基于 UDP）被设计出来——每个 Stream 独立重传，不互相影响。

---

## 5. Go + gRPC 性能调优

### 5.1 连接池

```go
// 每个节点维护少量长连接，避免频繁建连
conn, err := grpc.Dial(
    " consul://my-service/grpc?wait=1s",
    grpc.WithInsecure(), // 生产环境用 WithTransportCredentials
    grpc.WithDefaultServiceConfig(`{
        "loadBalancingPolicy": "round_robin",
        "methodConfig": [{
            "name": [{"service": ""}],
            "retryPolicy": {
                "maxAttempts": 3,
                "initialBackoff": "0.1s",
                "maxBackoff": "1s",
                "backoffMultiplier": 2.0,
                "retryableStatusCodes": ["UNAVAILABLE"]
            }
        }]
    }`),
)
```

### 5.2 消息体压缩

```protobuf
option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_swagger) = {
    /* ... */
};

// 服务端开启压缩
var s *grpc.Server
s.EnableBaggageOnServerSide = true // OpenTelemetry 相关

// 或在 Go 客户端配置压缩
conn, _ := grpc.Dial(target,
    grpc.WithDefaultCallOptions(grpc.UseCompressor("gzip")),
)
```

### 5.3 连接 Keepalive

```go
conn, _ := grpc.Dial(
    target,
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle: 5 * time.Minute,
        MaxConnectionAge: 30 * time.Minute,
        Time: 10 * time.Second,
        Timeout: 3 * time.Second,
    }),
)
```

### 5.4 窗口大小调优

gRPC 默认窗口：
- 连接级别：64KB
- Stream 级别：64KB

高并发场景需要调大：

```go
// 客户端
conn, _ := grpc.Dial(
    target,
    grpc.WithInitialWindowSize(1024*1024),  // Stream 窗口
    grpc.WithInitialConnWindowSize(8*1024*1024), // 连接窗口
)
```

---

## 面试话术

**Q：HTTP/2 多路复用和 HTTP/1.1 的管线化（pipelining）有什么区别？**

> HTTP/1.1 的管线化虽然能在一个连接里串行发多个请求，但响应必须按请求顺序返回（队头阻塞）。HTTP/2 的多路复用是响应可以**乱序**返回，只按 Stream ID 区分，互不阻塞。本质区别在于 HTTP/1.1 管线化没有解决响应层面的队头阻塞。

**Q：gRPC 为什么基于 HTTP/2？**

> 1）多路复用，一条连接承载大量并发 RPC；2）头部压缩（HPACK），减少元数据开销；3）流控，在应用层控制发送速度；4）流式调用天然适合 HTTP/2 的双向流帧；5）安全（TLS 是 HTTP/2 的前提），gRPC 继承 HTTPS 的安全性。

**Q：gRPC 在 HTTP/2 之上额外做了哪些事？**

> 1）定义了 Protobuf 序列化格式（application/grpc）；2）定义了 trailers 传递错误码（grpc-status）；3）定义了调用语义（Unary/Server-Stream/Client-Stream/Bidi-Stream）；4）支持自定义拦截器（UnaryServerInterceptor/StreamServerInterceptor）；5）加了连通性检测和负载均衡机制。

**Q：gRPC 能否在 HTTP/1.1 上跑？**

> 正常不行。但 grpc-web 可以——浏览器端用 HTTP/1.1 或 HTTP/2 发送 WebSocket 请求给 grpc-web Proxy，Proxy 再转成标准 gRPC/HTTP2 调用后端。

---

## 延伸阅读

- [gRPC over HTTP/2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)
- [RFC 7540 - HTTP/2](https://tools.ietf.org/html/rfc7540)
- [RFC 7541 - HPACK](https://tools.ietf.org/html/rfc7541)
- [HTTP/2 帧格式详解](https://developers.google.com/web/fundamentals/performance/http2)
