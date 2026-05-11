# gRPC 原理与 Protobuf

> 考察频率：★★★★☆  难度：★★★☆☆

## 什么是 gRPC

gRPC 是 Google 主导的高性能 RPC 框架，基于 HTTP/2 传输，Protocol Buffers 序列化。

### vs REST

| 特性 | gRPC | REST |
|------|------|------|
| 传输协议 | HTTP/2 | HTTP/1.1 |
| 序列化 | Protobuf | JSON |
| 传输效率 | 高（二进制） | 低（文本） |
| 代码生成 | 原生支持 | 需 Swagger |
| 双向流 | 支持 | 不支持 |
| 浏览器支持 | 需要 grpc-web | 原生支持 |

---

## Protocol Buffers

### 定义 proto 文件

```protobuf
syntax = "proto3";

package user;

option go_package = "pb/user";

// 用户服务
service UserService {
    // 简单 RPC
    rpc GetUser(GetUserRequest) returns (User);
    // 服务端流
    rpc ListUsers(ListRequest) returns (stream User);
    // 客户端流
    rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);
    // 双向流
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    repeated string roles = 4;  // 数组
}

message GetUserRequest {
    int64 id = 1;
}

message ListRequest {
    int32 page_size = 1;
    string page_token = 2;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message CreateUsersResponse {
    int32 created = 1;
}
```

### Protobuf 编码原理

#### Varint（变长整数）

小数字用更少字节：
- 0-127 用 1 字节
- 128-16383 用 2 字节
- 负数用 10 字节（fixed32 更高效）

```
数字 300 编码：1010 1100 0000 0010
                  └─ 2-byte varint
```

#### Tag-Length 结构

每个字段编码为 `Tag + Length + Value`：
```
Tag = (field_number << 3) | wire_type
```

wire_type：
- 0：Varint
- 1：64-bit
- 2：Length-delimited（字符串、数组）
- 5：32-bit

#### 示例编码

```go
message User {
    int64 id = 1;    // tag=8 (1<<3|0), value=varint
    string name = 2; // tag=18 (2<<3|2), length+value
}
```

### JSON vs Protobuf

| 对比 | JSON | Protobuf |
|------|------|----------|
| 大小 | 大（文本） | 小（二进制） |
| 解析速度 | 慢 | 快 |
| 可读性 | 高 | 低（需 proto 文件） |
| 跨语言 | 好 | 需要生成代码 |

---

## HTTP/2 特性

### 多路复用

一条 TCP 连接上多个请求并发：
```
Stream 1: HEADERS + DATA
Stream 2: HEADERS + DATA (并发)
Stream 3: HEADERS + DATA (并发)
```

### 头部压缩（HPACK）

```
静态表：常用头部（:method GET）
动态表：本次连接中出现的头部
Huffman 编码：重复字符串压缩
```

### 流控制

- **客户端到服务器**：告诉对方可以发送多少
- **服务器到客户端**：反之亦然
- 避免快发送方压垮慢接收方

---

## gRPC 调用流程

```
Client                        Server
  │                            │
  │  1. HTTP/2 TCP Connection  │
  │ ──────────────────────────→│
  │                            │
  │  2. Protobuf Request       │
  │ ──────────────────────────→│
  │                            │
  │  3. Protobuf Response      │
  │ ←───────────────────────── │
  │                            │
```

### Go 服务端实现

```go
// 定义服务
type UserServiceServer struct {
    pb.UnimplementedUserServiceServer
}

func (s *UserServiceServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, err := getUserFromDB(req.Id)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "user not found")
    }
    return user, nil
}

// 注册服务
func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer()
    pb.RegisterUserServiceServer(s, &UserServiceServer{})
    s.Serve(lis)
}
```

### Go 客户端调用

```go
conn, _ := grpc.Dial("localhost:50051", grpc.WithInsecure())
defer conn.Close()

client := pb.NewUserServiceClient(conn)
resp, err := client.GetUser(context.Background(), &pb.GetUserRequest{Id: 123})
```

---

## 四种 RPC 模式

### 1. 简单 RPC（Unary）

```protobuf
rpc GetUser(GetUserRequest) returns (User);
```

### 2. 服务端流（Server Streaming）

```protobuf
rpc ListUsers(ListRequest) returns (stream User);
```

```go
// 服务端
func (s *UserServiceServer) ListUsers(req *pb.ListRequest, stream pb.UserService_ListUsersServer) error {
    users := getAllUsers()
    for _, user := range users {
        stream.Send(user)
    }
    return nil
}
```

### 3. 客户端流（Client Streaming）

```protobuf
rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);
```

### 4. 双向流（Bidirectional Streaming）

```protobuf
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```

---

## 拦截器（Interceptor）

### 服务端拦截器

```go
// 日志拦截器
func loggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    log.Printf("gRPC call: %s", info.FullMethod)
    return handler(ctx, req)
}

// 认证拦截器
func authInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Errorf(codes.Unauthenticated, "missing metadata")
    }
    token := md.Get("authorization")
    if !validateToken(token) {
        return nil, status.Errorf(codes.Unauthenticated, "invalid token")
    }
    return handler(ctx, req)
}

// 注册
s := grpc.NewServer(
    grpc.UnaryInterceptor(loggingInterceptor),
    grpc.ChainUnaryInterceptor(loggingInterceptor, authInterceptor),
)
```

---

## 常见问题

### Q：gRPC 如何做负载均衡？

- **客户端负载均衡**：客户端感知所有服务端地址
- **服务端代理**：Envoy/BFF 层代理
- **gRPC-LB**：基于 Google 内部协议

### Q：Protobuf 如何保证兼容性？

> 通过保留字段和严格的 Tag 管理。旧字段不能删除，只能 `reserved`。新增字段要用新的 Tag，不会影响旧版本解析。

### Q：为什么 gRPC 推荐用 pb 包名？

> 避免与 Go 标准库冲突，且明确标识这是 Protobuf 生成的代码。

---

## 附录：RPC 基础高频题

> **Q：RPC vs REST vs GraphQL 对比？**

| 维度 | RPC | REST | GraphQL |
|------|-----|------|---------|
| 设计理念 | 远程调用像本地函数 | 资源为导向，HTTP 语义 | 按需获取，客户端驱动 |
| 协议 | 自定义（gRPC/Hessian） | HTTP/1.1~2 | HTTP |
| 数据格式 | Protobuf/JSON | JSON/XML | JSON |
| 性能 | 高（gRPC + Protobuf） | 中等 | 中等 |
| 调试 | 困难（二进制） | 容易（JSON 明文）| 容易（JSON 明文）|
| 类型安全 | 强（IDL 代码生成） | 弱（无契约）| 中等（Schema）|
| 适用场景 | 微服务内部 | 微服务外部 API | 前端按需获取多资源 |

**gRPC 适用场景**：高并发微服务内部通信，支持双向流。
**REST 适用场景**：对外开放 API、Webhook、跨语言异构系统。
**GraphQL 适用场景**：移动端/前端按需取字段，减少 over-fetching。

---

> **Q：Protobuf vs JSON 性能对比？**

| 维度 | Protobuf | JSON |
|------|----------|------|
| 编码方式 | 二进制（字段编号）| 文本（字段名）|
| 体积 | 小（1,2 等短数字 1 字节）| 大（字段名重复传输）|
| 解析速度 | 快（内存直接映射，无需字符串解析）| 慢（字符串解析 + 类型转换）|
| 可读性 | 差（需解码）| 好（人类可读）|
| 跨语言 | 好（IDL 生成各语言代码）| 好（通用文本格式）|
| 兼容性 | 好（字段可添加/废弃）| 好（天然兼容）|

**性能数据（典型场景）**：
- Protobuf 序列化速度比 JSON 快 **5~10 倍**
- 序列化后体积比 JSON 小 **2~5 倍**（字段越多差距越大）

---

> **Q：gRPC 四种通信模式？**

| 模式 | 名称 | 适用场景 |
|------|------|----------|
| **Unary RPC** | 一元调用 | 请求-响应型 API（大多数场景）|
| **Server Streaming** | 服务端流 | 大数据响应、实时推送（聊天、监控）|
| **Client Streaming** | 客户端流 | 文件上传、日志批量提交 |
| **Bidirectional Streaming** | 双向流 | 实时交互（语音/视频协作、游戏）|

```protobuf
// Unary：客户端发一个请求，服务端返回一个响应
rpc GetUser(GetUserRequest) returns (User);

// Server Streaming：客户端发一个请求，服务端返回流
rpc ListUsers(ListRequest) returns (stream User);

// Client Streaming：客户端发送流，服务端返回一个响应
rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);

// Bidirectional Streaming：双方都可发流
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```

**实现复杂度排序**：Unary < Server Streaming < Client Streaming < Bidirectional Streaming。
**生产使用建议**：双向流最难做，要处理乱序、重连、背压，生产中优先考虑 Unary 或 Server Streaming。
