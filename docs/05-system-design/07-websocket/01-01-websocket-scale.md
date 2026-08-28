# WebSocket 长连接架构：百万级并发在线用户实战

> 考察频率：★★★★☆  优先级：P1  字节/腾讯/游戏公司必考
> 关键词：WebSocket、心跳保活、水平扩展、连接管理、消息路由、断线重连

---

## 核心答案（30 秒版）

| 维度 | 关键设计 | Go 实现要点 |
|------|----------|-------------|
| **协议** | WebSocket（升级自 HTTP）→ TCP 长连接 | `gorilla/websocket` 或标准库 `net/http` upgrade |
| **心跳检测** | 客户端定时 ping，服务端 timeout 清理 | `SetReadDeadline` + `WriteMessage(PingMessage)` |
| **水平扩展** | 多进程/多机器共享连接 → Redis Pub/Sub | `sync.Map`（单机）+ Redis Channel（集群） |
| **消息路由** | 根据 userID 分发到对应 Goroutine | UserID → ServerID 映射表（Redis Hash） |
| **连接上限** | 单进程默认 ~10万，受文件描述符限制 | `ulimit -n` + rlimit 调优 |

**生产最佳实践**：Go 单进程稳态运行 10~50 万长连接，配合 K8s HPA 弹性伸缩。

---

## 深度展开

### 1. WebSocket 握手与连接建立

```
客户端                          WebSocket Server (Go)
  │                                 │
  │── GET /ws?token=xxx ──────────→│
  │── Host: chat.example.com      │
  │── Upgrade: websocket          │
  │── Connection: Upgrade         │
  │── Sec-WebSocket-Key: xxx      │
  │── Sec-WebSocket-Version: 13   │
  │                                 │
  │←── 101 Switching Protocols ───│
  │←── Upgrade: websocket          │
  │←── Connection: Upgrade         │
  │←── Sec-WebSocket-Accept: yyy   │
  │                                 │
  │═══ WebSocket 双向通信通道 ═══    │
  │                                 │
  │── [text/binary frame] ────────→│
  │←── [text/binary frame] ────────│
  │                                 │
```

```go
package websocket

import (
    "fmt"
    "net/http"
    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  4096,   // 4KB 读缓冲区
    WriteBufferSize: 4096,   // 4KB 写缓冲区
    CheckOrigin: func(r *http.Request) bool {
        // 生产环境要校验 Origin
        origin := r.Header.Get("Origin")
        return origin == "https://chat.example.com" || isLocalhost(r)
    },
}

// WSGateway WebSocket 网关
type WSGateway struct {
    connections map[int64]*Client     // userID → Client
    mu          sync.RWMutex
}

func NewWSGateway() *WSGateway {
    return &WSGateway{
        connections: make(map[int64]*Client),
    }
}

func (gw *WSGateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 1. Token 鉴权
    token := r.URL.Query().Get("token")
    userID, err := validateToken(token)
    if err != nil {
        http.Error(w, "unauthorized", http.StatusUnauthorized)
        return
    }

    // 2. WebSocket 升级
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("upgrade error: %v", err)
        return
    }

    // 3. 注册客户端连接
    client := NewClient(userID, conn, gw)
    gw.mu.Lock()
    gw.connections[userID] = client
    gw.mu.Unlock()

    // 启动读写循环
    client.Start()
}
```

### 2. 心跳机制与断线检测

这是面试中最容易被追问的细节——很多候选人只知道"要设心跳"，但说不清楚具体怎么做。

```go
package websocket

import (
    "fmt"
    "log"
    "time"
    "github.com/gorilla/websocket"
)

const (
    heartbeatInterval = 30 * time.Second  // 每 30 秒发一次 ping
    readWait          = 60 * time.Second  // 读取超时（ping-pong 也算活跃）
    writeWait         = 10 * time.Second  // 写入超时
    pongWait          = 90 * time.Second  // 等待 pong 的超时时间
    maxMessageSize    = 65536             // 最大消息 64KB
)

type Client struct {
    userID       int64
    conn         *websocket.Conn
    send         chan []byte            // 发送队列
    gateway      *WSGateway
    writeMu      sync.Mutex             // 写锁，保证不并发写
    lastActive   time.Time
    stopped      chan struct{}          // 停止信号
}

func NewClient(userID int64, conn *websocket.Conn, gw *WSGateway) *Client {
    c := &Client{
        userID: userID,
        conn:   conn,
        send:   make(chan []byte, 256), // 带缓冲，防止阻塞
        gateway: gw,
        stopped: make(chan struct{}),
    }
    c.lastActive = time.Now()
    return c
}

// Start 启动客户端的读循环和写循环
func (c *Client) Start() {
    go c.readPump()
    go c.writePump()
}

// readPump 处理来自客户端的消息和心跳
func (c *Client) readPump() {
    defer func() {
        c.close()
    }()

    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(readWait))
    c.conn.PongHandler(func(message string) error {
        c.conn.SetReadDeadline(time.Now().Add(readWait)) // pong 重置读超时
        return nil
    })

    for {
        messageType, message, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseNormalClosure) {
                log.Printf("[client %d] read error: %v", c.userID, err)
            }
            break
        }
        c.lastActive = time.Now()

        // 处理不同消息类型
        switch messageType {
        case websocket.TextMessage:
            c.gateway.handleTextMessage(c, message)
        case websocket.BinaryMessage:
            c.gateway.handleBinaryMessage(c, message)
        case websocket.CloseMessage:
            log.Printf("[client %d] connection closed by client", c.userID)
            return
        }
    }
}

// writePump 向客户端推送消息
func (c *Client) writePump() {
    ticker := time.NewTicker(heartbeatInterval)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            c.writeMu.Lock()
            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)
            // 批量写入（可缓存多条消息）
            if err := w.Close(); err != nil {
                return
            }
            c.writeMu.Unlock()

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            c.writeMu.Lock()
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                c.writeMu.Unlock()
                return
            }
            c.writeMu.Unlock()
        case <-c.stopped:
            return
        }
    }
}
```

**关键点解释**：

```
readPump:
  └── gorilla/websocket 的 SetReadDeadline(60s) 是总超时
      ├── 如果 60 秒内没收到任何消息 → 超时 → close()
      ├── Ping/Pong 消息也会更新 LastActive → 不算超时 ✅
      └── 普通 text/binary 消息也重置超时 → 正常交互 ✅

writePump:
  └── 每 30 秒发一次 Ping 帧
      ├── 对方回复 Pong → ReadMessage 触发 PongHandler → 重置 deadline ✅
      ├── 对方超过 90 秒没回 Pong → deadline 到期 → close() ❌
      └── 这就是"心跳保活"的本质！不是应用层 ping/pong 协议
```

### 3. 集群水平扩展方案

单机 Goroutine 能扛多少连接？Go 官方测试数据：**单台 4C16G 服务器稳态运行约 30~50 万长连接**。生产建议按 10~20 万/节点规划。

当需要承载百万级连接时，必须做多机部署。**核心问题**：A 服务器的用户想给 B 服务器的用户发消息，怎么路由？

#### 方案 A：Redis Pub/Sub（推荐）

```
┌─────────┐    ┌─────────┐    ┌─────────┐
│Server 1 │    │Server 2 │    │Server 3 │
│UID:1001  │    │UID:2002  │    │UID:3003  │
│UID:1002  │    │UID:2003  │    │UID:3004  │
└────┬────┘    └────┬────┘    └────┬────┘
     │              │              │
     └──────────────┼──────────────┘
                    │
           ┌────────▼────────┐
           │   Redis Cluster  │
           │  Pub/Sub Channel │
           └─────────────────┘

消息流程：
Server 2 要给 UID:1001 发消息
  → Server 2 查 Redis Hash(user->server) 发现 1001 在 Server 1
  → 将消息发给 Redis Channel "broadcast"
  → Server 1 监听该 Channel，收到后投递给本地 Client
```

```go
type ClusterGateway struct {
    localUsers    map[int64]*Client  // 本机的用户连接
    userServers   map[int64]string   // uid -> serverID（存在 Redis）
    serverID      string             // 本机标识
    redisClient   *redis.Client
}

func (gw *ClusterGateway) SendToUser(targetUID int64, message []byte) error {
    // 查询目标用户在哪台服务器上
    targetServer, err := gw.redisClient.HGet(context.Background(),
        "user_server_map", fmt.Sprintf("%d", targetUID)).Result()
    if err != nil {
        return err
    }

    if targetServer == gw.serverID {
        // 同机，直接投递
        if client, ok := gw.localUsers[targetUID]; ok {
            client.SendMessage(message)
            return nil
        }
        return fmt.Errorf("user %d not found locally", targetUID)
    }

    // 异地，通过 Redis Pub/Sub 转发
    payload := clusterMessage{
        FromServer: gw.serverID,
        ToServer:   targetServer,
        ToUID:      targetUID,
        Message:    message,
    }
    data, _ := json.Marshal(payload)
    return gw.redisClient.Publish(context.Background(), "message_forward", data).Err()
}

// 启动消息订阅，接收转发过来的消息
func (gw *ClusterGateway) StartForwardListener() {
    pubsub := gw.redisClient.Subscribe(context.Background(), "message_forward")
    ch := pubsub.Channel()
    go func() {
        for msg := range ch {
            var cm clusterMessage
            json.Unmarshal([]byte(msg.Payload), &cm)
            if cm.ToServer == gw.serverID {
                if client, ok := gw.localUsers[cm.ToUID]; ok {
                    client.SendMessage(cm.Message)
                }
            }
        }
    }()
}
```

#### 方案 B：MQ 广播（适合高频场景）

用 Kafka 替代 Redis Pub/Sub，支持消息持久化和消费重试。

### 4. 离线消息与消息可靠性

```
客户端断线期间收到的消息存储策略：

┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  客户端      │     │  消息服务     │     │  MySQL/Kafka │
│  (离线状态)   │     │  (消息落盘)   │     │  (持久化)     │
└─────────────┘     └──────────────┘     └──────────────┘

1. 客户端上线 → 拉取离线消息列表（从 DB 查询）
2. 标记已读 → 删除旧离线消息
3. 后续消息通过 WebSocket 实时推送
```

```go
// 消息落盘逻辑
func (gw *ClusterGateway) OnSendMessage(senderUID, receiverUID int64, content []byte, msgType string) {
    // 1. 检查目标用户是否在线
    receiverServer, _ := gw.redisClient.HGet(ctx, "user_server", strconv.FormatInt(receiverUID, 10)).Result()

    if receiverServer == gw.serverID {
        // 在线 → WebSocket 推送
        if client, ok := gw.localUsers[receiverUID]; ok {
            client.SendMessage(content)
        }
    }

    // 2. 无论在线与否都落盘（保证不丢消息）
    gw.messageStore.SaveOfflineMessage(receiverUID, senderUID, content, msgType)

    // 3. 如果是群聊，异步发送到 Kafka 做扩散
    if isGroupMessage(content) {
        gw.kafkaProducer.Send(groupTopic, content)
    }
}

// 客户端上线时拉取离线消息
func (gw *ClusterGateway) FetchOfflineMessages(userID int64) ([]*OfflineMsg, error) {
    return gw.messageStore.QueryByUserID(userID, time.Now().Add(-7*24*time.Hour))
}
```

### 5. 资源占用估算与性能优化

```
单连接资源消耗（Go 实测数据）：

- 两个 Goroutine（readPump + writePump） ≈ 2KB × 2 = 4KB（初始栈）
- websocket.Conn 对象 ≈ 2~4KB
- 读/写缓冲区 ≈ 4KB × 2 = 8KB
- 总计约 16~20KB/连接

30 万连接 ≈ 6GB 内存（不含业务数据）
加上 GC overhead 和业务 Goroutine，实际约需 10~12GB

Go 1.2x 优势：GC 停顿更短，后台 GC 线程减少，长连接场景更稳定
```

**压测指标参考**：

| 指标 | 单机值 | 说明 |
|------|--------|------|
| 连接数 | 10~50万 | CPU 2C+ 建议 ≤ 20万 |
| 发送 QPS | ~50万 | 纯本地转发 |
| 消息延迟 P50 | < 1ms | 本地转发 |
| 消息延迟 P99 | < 10ms | 含 Redis Pub/Sub 转发 |
| 带宽（上行） | 取决于消息大小 | 平均 1KB 消息下 ≈ 50MB/s |

### 6. 面试话术

**Q：如何优雅关闭所有 WebSocket 连接？**

> 我们需要两步：1）先对所有连接调用 Close，触发 close handler；2）等待每个连接的 writePump 完成（用 WaitGroup 计数）。在 Go 中要注意：对已经关闭的 conn 再次调用 Write 会导致 panic，所以要先标记为 stopped。对于百万级连接不能串行逐个关闭——会等太久。可以开一个协程池并行关闭，每批处理 1000 个连接。

**Q：WebSocket 和 SSE（Server-Sent Events）怎么选型？**

> WebSocket 是全双工，客户端和服务端都能主动发消息，适合即时聊天、多人协同编辑。SSE 是单向的（服务端推送到客户端），但比 WebSocket 简单得多——基于 HTTP，自带重连机制，兼容性更好。如果只是公告通知、股票行情这种"服务端推"场景，优先选 SSE。需要双向交互才上 WebSocket。
