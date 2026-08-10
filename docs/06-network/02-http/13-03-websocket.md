# WebSocket 原理、升级握手、与 HTTP 长轮询对比

> 考察频率：★★★☆☆  难度：★★★☆☆
> 关键词：升级握手、双向通信、帧格式、心跳、ws/wss

---

## 核心答案（30 秒版）

WebSocket 是**全双工**通信协议，建立在 TCP 之上。HTTP 的 GET/POST 是一问一答，WebSocket 建立后双方可以随时互发消息。

**连接建立过程（HTTP 升级）：**
```
客户端：GET /ws HTTP/1.1
       Upgrade: websocket
       Connection: Upgrade
       Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
       Sec-WebSocket-Version: 13
           →
←  HTTP/1.1 101 Switching Protocols
    Upgrade: websocket
    Connection: Upgrade
    Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
           ↓
   ★ 全双工 WebSocket 连接建立 ★
```

**vs HTTP 长轮询：**
- 长轮询：客户端发请求，服务器等有数据才返回 → 每次请求都有 HTTP 头开销，延迟高
- WebSocket：连接建立后无协议头开销，毫秒级双向通信

---

## 1. 什么是 WebSocket

### 1.1 问题：HTTP 的局限性

HTTP 是**半双工**的：客户端发请求，服务器才能响应。服务器不能主动推送给客户端。

业务场景需求：
- 聊天消息实时推送
- 在线协作编辑（多人同时编辑文档）
- 游戏帧同步
- 股票行情实时更新
- 监控系统告警推送

### 1.2 WebSocket 的解决方案

WebSocket 在 TCP 之上提供**全双工**通道，服务器可以主动发消息给客户端。

| 特性 | HTTP/1.1 | WebSocket |
|------|----------|-----------|
| 通信方向 | 半双工（请求-响应）| 全双工（随时互发）|
| 连接 | 短连接（一次请求一次响应）| 长连接（保持打开）|
| 协议开销 | 每个请求都有 HTTP 头（~500B）| 帧头 2-14B，极低 |
| 服务器推送 | ❌ 无法实现 | ✅ 支持 |
| 实时性 | 秒级（轮询间隔）| 毫秒级 |

---

## 2. WebSocket 握手详解

### 2.1 客户端握手请求

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

关键请求头：
- `Upgrade: websocket` — 请求协议升级
- `Connection: Upgrade` — 保持连接
- `Sec-WebSocket-Key` — 随机 Base64 字符串，用于握手验证
- `Sec-WebSocket-Version: 13` — 协议版本（固定）

### 2.2 服务器握手响应

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

### 2.3 Key 的计算（Sec-WebSocket-Accept）

```
Const: 258EAFA5-E914-47DA-95CA-C5AB0DC85B11

Sec-WebSocket-Key = "dGhlIHNhbXBsZSBub25jZQ=="
Accept = Base64(SHA1(Sec-WebSocket-Key + Const))
       = Base64(SHA1("dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))
       = "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
```

服务器必须验证 Key 格式正确，防止随意伪造 WebSocket 握手。

### 2.4 Go 实现握手验证

```go
import (
    "crypto/sha1"
    "encoding/base64"
    "io"
    "net/http"
)

const wsGUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

func serveWs(w http.ResponseWriter, r *http.Request) {
    // 1. 验证是 WebSocket 升级请求
    if r.Header.Get("Upgrade") != "websocket" {
        http.Error(w, "not websocket", http.StatusBadRequest)
        return
    }
    if r.Header.Get("Connection") != "Upgrade" {
        http.Error(w, "Connection not Upgrade", http.StatusBadRequest)
        return
    }

    // 2. 验证 WebSocket 版本
    if r.Header.Get("Sec-WebSocket-Version") != "13" {
        http.Error(w, "invalid websocket version", http.StatusBadRequest)
        return
    }

    // 3. 计算 Sec-WebSocket-Accept
    key := r.Header.Get("Sec-WebSocket-Key")
    h := sha1.New()
    io.WriteString(h, key+wsGUID)
    accept := base64.StdEncoding.EncodeToString(h.Sum(nil))

    // 4. 返回 101 Switching Protocols
    w.Header().Set("Upgrade", "websocket")
    w.Header().Set("Connection", "Upgrade")
    w.Header().Set("Sec-WebSocket-Accept", accept)

    // 可选：设置子协议
    if r.Header.Get("Sec-WebSocket-Protocol") == "chat" {
        w.Header().Set("Sec-WebSocket-Protocol", "chat")
    }

    w.WriteHeader(101)

    // 5. 之后用 Hijack 获取原始连接
    hijacker, _ := w.(http.Hijacker)
    conn, rw, _ := hijacker.Hijack()
    defer conn.Close()

    // 从这里开始是 WebSocket 帧协议
    // ...
}
```

---

## 3. WebSocket 帧格式

### 3.1 帧结构

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-------------+-+-----------+---------------+-------------------+
|F|R|R|R|  opcode |M|  payload |    payload    |                   |
|I|S|S|S|         |A|  length  |    length     |  Extended payload  |
|N|V|V|V|         |S|  (7+16)  |    (64)       |  length (if 64)   |
|S|V|V|V|         |K|          |               |                   |
+-+-------------+-+-----------+---------------+-------------------+
```

| 字段 | 说明 |
|------|------|
| FIN (1 bit) | 1=消息结束，0=消息未完 |
| RSV1-3 (3 bit) | 扩展用，通常为 0 |
| opcode (4 bit) | 0x0=继续帧，0x1=文本，0x2=二进制，0x8=关闭，0x9=Ping，0xA=Pong |
| MASK (1 bit) | 1=负载掩码（客户端发服务器必须为1）|
| Payload len (7+16/64 bit) | 数据长度 |
| Masking-Key (0/4 bytes) | 掩码密钥（MASK=1时）|
| Payload | 应用数据 |

### 3.2 帧类型

| Opcode | 含义 | 用途 |
|--------|------|------|
| 0x1 | Text frame | 文本消息 |
| 0x2 | Binary frame | 二进制消息 |
| 0x8 | Close frame | 关闭连接 |
| 0x9 | Ping | 心跳（请求）|
| 0xA | Pong | 心跳（响应）|

### 3.3 Go 帧解析

```go
const (
    opcodeText   = 0x1
    opcodeBinary = 0x2
    opcodeClose  = 0x8
    opcodePing   = 0x9
    opcodePong  = 0xA
)

type Frame struct {
    Fin    bool
    Opcode int
    Masked bool
    Mask   [4]byte
    Payload []byte
}

func readFrame(r io.Reader) (*Frame, error) {
    b := make([]byte, 2)
    _, err := io.ReadFull(r, b)
    if err != nil {
        return nil, err
    }

    f := &Frame{}
    f.Fin = (b[0] & 0x80) != 0
    f.Opcode = int(b[0] & 0x0F)
    f.Masked = (b[1] & 0x80) != 0

    payloadLen := int64(b[1] & 0x7F)
    if payloadLen == 126 {
        buf := make([]byte, 2)
        io.ReadFull(r, buf)
        payloadLen = int64(buf[0])<<8 | int64(buf[1])
    } else if payloadLen == 127 {
        buf := make([]byte, 8)
        io.ReadFull(r, buf)
        payloadLen = int64(buf[0])<<56 | int64(buf[1])<<48 |
                     int64(buf[2])<<40 | int64(buf[3])<<32 |
                     int64(buf[4])<<24 | int64(buf[5])<<16 |
                     int64(buf[6])<<8  | int64(buf[7])
    }

    if f.Masked {
        io.ReadFull(r, f.Mask[:])
    }

    f.Payload = make([]byte, payloadLen)
    io.ReadFull(r, f.Payload)

    // 解掩码（客户端发的帧需要）
    if f.Masked {
        for i := range f.Payload {
            f.Payload[i] ^= f.Mask[i%4]
        }
    }

    return f, nil
}
```

---

## 4. WebSocket vs HTTP 长轮询 vs Server-Sent Events

### 4.1 HTTP 长轮询（Long Polling）

```
客户端 → 服务器：有消息吗？
           服务器：没有，等...
           服务器：等...（假设 30 秒后）
           服务器：有了！"hello"
客户端 ← 服务器："hello"
客户端 → 服务器：有消息吗？  ← 立即发下一个请求
           服务器：没有，等...
```

**问题**：
- 每次请求都要带完整的 HTTP 头（~500B）
- 每次"等"都要占用一个 HTTP 连接
- 服务器响应后才能发下一个请求，有延迟

### 4.2 Server-Sent Events (SSE)

服务器用 `text/event-stream` MIME 类型**只推**给客户端：

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream

event: message
data: {"msg": "hello"}

event: message
data: {"msg": "world"}
```

**限制**：只能是服务器→客户端，客户端不能主动发消息给服务器。

### 4.3 对比表

| 特性 | HTTP 长轮询 | SSE | WebSocket |
|------|------------|-----|-----------|
| 方向 | 伪推送（还是轮询）| 单向推送 | 全双工 |
| 协议开销/帧 | ~500B/请求 | ~50B/消息 | 2-14B/帧 |
| 实时性 | 秒级 | 毫秒级 | 毫秒级 |
| 客户端→服务器 | 正常 HTTP 请求 | 需要额外 HTTP | 帧直接发 |
| 复杂度 | 低 | 中 | 中 |
| 兼容性 | 好 | 不支持 IE | 好（IE10+）|
| 防火墙/代理 | 无问题 | 需特殊处理 | 可能被误拦 |

---

## 5. Go WebSocket 实战（gorilla/websocket）

### 5.1 服务端

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        // ⚠️ 生产环境要验证 Origin，防止跨站 WebSocket 劫持
        return true
    },
}

func wsHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("upgrade failed: %v", err)
        return
    }
    defer conn.Close()

    for {
        // 读取消息
        msgType, msg, err := conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                log.Printf("error: %v", err)
            }
            break
        }

        log.Printf("recv: %s", msg)

        // 原样发回
        if err := conn.WriteMessage(msgType, msg); err != nil {
            log.Printf("write failed: %v", err)
            break
        }
    }
}

func main() {
    http.HandleFunc("/ws", wsHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 5.2 客户端

```go
package main

import (
    "fmt"
    "log"
    "github.com/gorilla/websocket"
)

func main() {
    // wss:// 才是加密的 WebSocket
    conn, _, err := websocket.DefaultDialer.Dial("ws://localhost:8080/ws", nil)
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    done := make(chan struct{})

    // 读协程
    go func() {
        defer close(done)
        for {
            _, msg, err := conn.ReadMessage()
            if err != nil {
                log.Printf("read error: %v", err)
                return
            }
            log.Printf("recv: %s", msg)
        }
    }()

    // 发消息
    for i := 0; i < 5; i++ {
        msg := fmt.Sprintf("hello %d", i)
        if err := conn.WriteMessage(websocket.TextMessage, []byte(msg)); err != nil {
            log.Printf("write error: %v", err)
            return
        }
    }

    conn.WriteMessage(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""))

    <-done
}
```

### 5.3 心跳机制

WebSocket 协议本身没有心跳，需要应用层实现：

```go
type PingPongHandler struct {
    conn    *websocket.Conn
    writeMu sync.Mutex
}

func (h *PingPongHandler) startPingPong(timeout time.Duration) {
    h.conn.SetPongHandler(func(appData string) error {
        // 收到 Pong，说明连接还活着，重置计时器
        return nil
    })

    ticker := time.NewTicker(timeout)
    defer ticker.Stop()

    for {
        <-ticker.C
        h.writeMu.Lock()
        err := h.conn.WriteControl(websocket.PingMessage, nil, time.Now().Add(timeout/2))
        h.writeMu.Unlock()
        if err != nil {
            log.Printf("ping failed: %v", err)
            return
        }
    }
}
```

### 5.4 跨站 WebSocket 劫持（CSWSH）

浏览器中，WebSocket 不受 CORS 限制，但 `Origin` 头仍会发送。恶意网站可以诱导用户连接你的 WebSocket 服务。

**防护**：

```go
var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        // 白名单
        allowedOrigins := map[string]bool{
            "https://example.com": true,
            "https://app.example.com": true,
        }
        return allowedOrigins[origin]
    },
}
```

---

## 6. wss://（WebSocket over TLS）

生产环境必须用 `wss://`，防止中间人攻击：

```go
// 服务端
http.ListenAndServeTLS(":8443", "server.crt", "server.key", nil)

// 客户端
conn, _, err := websocket.DefaultDialer.Dial("wss://example.com/ws", nil)
```

---

## 面试话术

**Q：WebSocket 握手为什么用 HTTP？**

> 为了兼容现有 HTTP 基础设施（负载均衡器、反向代理、防火墙）。很多设备只允许 80/443 端口的 HTTP 流量，用 HTTP 做握手可以让 WebSocket 更容易穿透这些设备。握手完成后才切换到 WebSocket 帧协议。

**Q：WebSocket 连接断开怎么感知？**

> 1）读消息时检测到 `io.EOF` 或 `websocket.CloseError`；2）设置读超时，超时则认为断开；3）应用层心跳（Ping/Pong），超时没收到响应则断开。需要心跳+重连机制配合。

**Q：WebSocket 和 Socket.io 有什么区别？**

> Socket.io 是封装了 WebSocket 的库，增加了：自动降级（WebSocket 不可用时降级到 HTTP 长轮询）、心跳、断线重连、房间概念。缺点是协议不标准，Go/Java 客户端不能直接对接。纯 WebSocket 则是标准协议，所有语言客户端都兼容。

**Q：Nginx 能代理 WebSocket 吗？**

> 能，但要配置 `proxy_http_version 1.1` 和 `proxy_set_header Upgrade $http_upgrade` 和 `proxy_set_header Connection "upgrade"`。Nginx 1.3+ 支持 WebSocket 代理。但要注意 Nginx 默认 60 秒不活动会关闭连接，需要加 `proxy_read_timeout 86400`。

---

## 延伸阅读

- [RFC 6455 - WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
- [WebSocket MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [gorilla/websocket](https://github.com/gorilla/websocket)
- [WebSocket vs SSE vs Long Polling](https://stackoverflow.com/questions/5195452/)
