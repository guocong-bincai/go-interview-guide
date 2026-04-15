# TCP Keepalive vs 应用层心跳

> 考察频率：★★★☆☆  难度：★★★☆☆

## 核心问题

网络探活考察什么：
1. **TCP Keepalive**：系统级保活、检测 dead peer
2. **应用层心跳**：业务级探活、双向检测
3. **选型策略**：什么时候用哪个

---

## 1. TCP Keepalive

### 原理

TCP Keepalive 是操作系统层面的机制，检测空闲连接是否存活：

```
客户端                          服务端
   |                               |
   |---(无数据)---空闲超时--------->|
   |                               |
   |<----Keepalive Probe----------|
   |                               |
   |---ACK--->|                   |
   |          (正常，继续通信)     |
```

**参数**：
- `tcp_keepalive_time`：连接空闲多久后开始探测（默认 7200 秒 = 2 小时）
- `tcp_keepalive_intvl`：探测间隔（默认 75 秒）
- `tcp_keepalive_probes`：探测次数（默认 9 次）

### Go 设置 Keepalive

```go
// 服务端：设置 TCP Keepalive
listener, err := net.Listen("tcp", ":8080")
for {
    conn, err := listener.Accept()
    
    // 设置 Keepalive 参数
    tcpConn := conn.(*net.TCPConn)
    tcpConn.SetKeepAlive(true)
    tcpConn.SetKeepAlivePeriod(30 * time.Second)  // 30 秒探测一次
    
    go handleConn(conn)
}
```

### HTTP/2 Keepalive

```go
// HTTP/2 的 ping 帧作为 keepalive
// grpc 底层用的是 HTTP/2
```

---

## 2. 应用层心跳

### 为什么需要应用层心跳

TCP Keepalive 的问题：
1. **时间太长**：2 小时才开始探测
2. **检测不准确**：进程卡住但 OS 活着，连接是活的但服务已死
3. **不双向**：只检测一端

### WebSocket 心跳

```go
// 应用层心跳：ping/pong
const (
    OP_PING = 0x89
    OP_PONG = 0x8A
)

// 客户端：定时发 ping
func (c *Client) startHeartbeat() {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            if err := c.conn.SetWriteDeadline(time.Now().Add(5 * time.Second)); err != nil {
                c.Close()
                return
            }
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                c.Close()
                return
            }
        case <-c.done:
            return
        }
    }
}

// 服务端：收到 ping 回复 pong
func (c *Client) handleMessage(msgType int, data []byte) error {
    switch msgType {
    case websocket.PingMessage:
        return c.conn.WriteMessage(websocket.PongMessage, data)
    case websocket.PongMessage:
        // 心跳响应，可以更新最近活跃时间
        c.lastActive = time.Now()
    }
    return nil
}
```

### RPC 心跳

```go
// gRPC 健康检查
// 客户端定期发 health check 请求
// 服务端回复 healthy/unhealthy

// 服务端实现
func (s *Server) Watch(req *health.HealthCheckRequest, stream health.Health_WatchServer) error {
    for {
        stream.Send(&health.HealthCheckResponse{
            Status: health.HealthCheckResponse_SERVING,
        })
        time.Sleep(5 * time.Second)
    }
}

// 客户端：超时断开重连
func (c *Client) keepAlive() {
    for {
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        resp, err := c.healthClient.Check(ctx, &health.HealthCheckRequest{})
        cancel()
        
        if err != nil || resp.Status != health.HealthCheckResponse_SERVING {
            c.reconnect()  // 断线重连
        }
        
        time.Sleep(30 * time.Second)
    }
}
```

### MQTT 心跳

```go
// MQTT 的 keepalive 机制
// 客户端设置 keepalive interval（如 60s）
// 如果服务器 1.5 倍时间没收到任何消息，认为客户端死了

// Go 客户端
options := mqtt.NewClientOptions()
options.SetKeepAlive(60 * time.Second)  // 60s
options.SetPingTimeout(10 * time.Second)
```

---

## 3. 对比与选型

| 特性 | TCP Keepalive | 应用层心跳 |
|------|--------------|-----------|
| 层级 | 传输层 | 应用层 |
| 检测范围 | OS + 网络 | OS + 进程 + 网络 |
| 响应速度 | 慢（小时级） | 快（秒级） |
| 资源消耗 | 低 | 略高 |
| 可靠性 | 一般 | 高 |
| 跨语言 | 原生支持 | 需要自己实现 |

### 选型建议

```markdown
# 什么时候用 TCP Keepalive
- 长连接保活（如 SSH）
- 对实时性要求不高
- 不想自己实现心跳逻辑

# 什么时候用应用层心跳
- WebSocket / gRPC 等长连接
- 需要检测"服务活着但卡住"的情况
- 需要双向检测（客户端知道服务端挂了，反之亦然）
- 需要自定义检测逻辑（如业务指标）

# 最佳实践：两者结合
- TCP Keepalive 作为网络层兜底
- 应用层心跳作为业务层保活
```

---

## 4. 实际案例

### 微服务保活

```go
// 服务注册 + 心跳续约（如 etcd）
// 客户端定期续约（心跳），服务端超时未收到则剔除

// etcd client
cli, err := clientv3.New(clientv3.Config{
    Endpoints:   []string{"localhost:2379"},
    DialTimeout: 5 * time.Second,
})

// 注册服务，每 5 秒续约
go func() {
    ticker := time.NewTicker(5 * time.Second)
    for range ticker.C {
        leaseResp, _ := cli.Grant(context.Background(), 10)
        cli.Put(context.Background(),
            "/services/my-service/instance-1",
            "localhost:8080",
            clientv3.WithLease(leaseResp.ID),
        )
    }
}()
```

---

## 5. 面试话术

**Q：TCP Keepalive 和应用层心跳有什么区别？**

> TCP Keepalive 是 OS 层的，只检测网络连通性和对端机器是否活着，时间太长（小时级）。应用层心跳是业务层的，可以检测进程是否卡住、队列是否满了等业务指标，响应更快（秒级）。实际生产环境两者结合用。

**Q：WebSocket 怎么防止断连？**

> 双向心跳 + 断线重连。客户端定时发 ping，服务端回复 pong 并更新活跃时间。如果超过 N 次没收到响应，判定断连，触发重连逻辑。同时要处理好重连风暴（大量客户端同时断线同时重连），可以用指数退避。

---

## 总结

| 场景 | 推荐方案 |
|------|---------|
| SSH 长连接 | TCP Keepalive |
| WebSocket | 应用层 ping/pong |
| gRPC | HTTP/2 ping + health check |
| 微服务注册 | etcd 续约心跳 |
| MQTT | MQTT keepalive |
