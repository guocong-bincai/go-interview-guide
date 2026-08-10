# HTTP/1.1 vs HTTP/2 vs HTTP/3

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：多路复用、头部压缩、QUIC、TLS 1.3、Stream、Server Push

## 🎯 面试官考察意图

- 候选人对 HTTP 各版本演进的理解深度
- 能否准确说出每个版本解决了什么问题，又引入了什么新问题
- 是否有实际生产环境中 HTTP 协议选型经验
- 对 QUIC/HTTP/3 的理解程度（新技术方向）

---

## ⚡ 核心答案（30秒）

> HTTP/1.1 每次连接只能一个请求，**队头阻塞**问题严重，虽然可以用多个域名绕过但资源浪费。HTTP/2 引入**多路复用**，一个 TCP 连接可以并行多个 Stream，消除了一部分的队头阻塞，但 TCP 层面的队头阻塞仍然存在（丢包时所有 Stream 都受影响）。HTTP/3 用 **QUIC**（基于 UDP）完全解决了 TCP 队头阻塞，且内置 TLS 1.3，连接建立只需 0-RTT 或 1-RTT。

---

## 🔬 深度展开

### 1. HTTP/1.1 的问题

#### 队头阻塞（Head-of-Line Blocking）

```
浏览器最多6个并发连接（HTTP/1.1 默认行为）
      ↓
      连接1：请求 A ──────────────────> （被阻塞）
      连接2：请求 B ─────> OK
      连接3：请求 C ──────────────────> OK
      连接4：请求 D ─────> OK
      连接5：请求 E ─────> OK
      连接6：请求 F ─────> OK

问题：连接1的响应慢，后面5个连接都要等
解决方案（Hack）：
  ① 域名分片：一个资源拆到多个域名（a.com, b.com, c.com）
  ② 资源合并：多个 JS 合并成一个
  ③ 内联资源：CSS/小图转 Base64 内联
```

#### keep-alive 优化

```go
// HTTP/1.1 默认开启 keep-alive（长连接）
// 一个 TCP 连接可以发多个请求/响应

// 但必须严格按顺序：
// Client: GET /a HTTP/1.1
// Server: HTTP/1.1 200 OK ... [响应 A]
// Client: GET /b HTTP/1.1  ← 必须等收到响应 A 后才能发
// Server: HTTP/1.1 200 OK ... [响应 B]

// Go HTTP Client 默认使用连接池
client := &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        100,
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 10 * time.Second,
    },
}
```

---

### 2. HTTP/2 的突破

#### 多路复用（Multiplexing）

```
HTTP/1.1：                        HTTP/2：
一个 TCP + 多请求（串行）           一个 TCP + 多 Stream（并行）

连接1: A ────────────>             Stream 1: A ───> A ──>
连接2: B ───────>                  Stream 2: B ─> B ──>
连接3: C ───────>                  Stream 3: C ─> C ──>
连接4: D ───────>                  Stream 4: D ─> D ──>

所有 Stream 共用一个 TCP 连接       Stream 之间完全独立
仍受 TCP 队头阻塞影响              解决了应用层队头阻塞
                                    但 TCP 层丢包时仍有影响
```

#### HPACK 头部压缩

```go
// HTTP/2 使用 HPACK 压缩请求头
// 核心思想：静态表 + 动态表 + 哈夫曼编码

// 请求1（完整头部）：
// :method: GET
// :path: /api/users
// :scheme: https
// host: api.example.com
// user-agent: Go-http-client/1.1
// accept: application/json
// [大几百字节]

// 请求2（只用索引，复用表）：
// [索引 2] :method: GET
// [索引 12] :path: /api/users
// [索引 36] :scheme: https
// [索引 53] host: api.example.com
// [索引 64] accept: application/json
// [几十字节，节省 80%+]

// Go 中使用 http/2
import _ "golang.org/x/net/http2"

srv := &http.Server{
    Addr: ":8443",
}
http2.ConfigureServer(srv, &http2.Server{
    MaxConcurrentStreams: 100, // 每个连接最大 Stream 数
})
```

#### Server Push

```go
// 服务器主动推送资源（CSS/JS/图片）
// 注意：Go 标准库不支持 Server Push（需要手动实现）
// Nginx/Envoy 原生支持

// 原理：
// 服务器收到 HTML 请求
// 主动推送 JS 和 CSS（不用等浏览器解析 HTML 后再请求）
// 减少 RTT，提升首屏渲染
```

#### HTTP/2 的问题：TCP 队头阻塞

```
场景：10个 HTTP/2 Stream 在一个 TCP 连接上
问题：Stream 3 的包丢了
影响：
  TCP 层：丢包触发重传，所有 Stream 都停等
  HTTP/2 层：只有 Stream 3 阻塞，其他 Stream 也受影响
  
丢包率 2% 时：
  HTTP/1.1（6连接）：6个独立连接，互不影响
  HTTP/2（1连接）：1个连接受严重影响
  HTTP/3（QUIC）：丢包只影响 Stream 3，其他 Stream 继续
  
结论：弱网环境下 HTTP/2 反而可能比 HTTP/1.1 慢
```

---

### 3. HTTP/3 与 QUIC

#### QUIC 协议栈

```
传统：                        HTTP/3：
┌─────────────┐              ┌─────────────┐
│   HTTP/1.1  │              │   HTTP/3    │
├─────────────┤              ├─────────────┤
│   HTTP/2    │              │   QUIC      │  ← 内置流控、重传、TLS
├─────────────┤              ├─────────────┤
│   TLS 1.2/1.3│             │   UDP       │  ← 无队头阻塞
├─────────────┤              ├─────────────┤
│   TCP        │              │   (内核)     │
├─────────────┤              └─────────────┘
│   IP         │
└─────────────┘
```

#### QUIC 核心特性

```go
// ① 连接建立：1-RTT（首次）或 0-RTT（复用）
// 首次握手：Client Hello（含 TLS） → Server Hello + 证书 + Finished
// 0-RTT：用上次会话票据直接发加密数据（可能重放攻击，需谨慎）

// ② Stream 级别流控
// TCP 是连接级别的流控
// QUIC 是 Stream 级别的流控 + 连接级别的流控
// 每个 Stream 独立流量控制，互不影响

// ③ 连接迁移（Connection Migration）
// 场景：WiFi → 4G 切换，IP 变了
// TCP：连接断开，需要重建
// QUIC：用 Connection ID（CID）标识连接
//       客户端 IP 变了 → 用相同 CID 继续通信
//       网络切换对应用层透明（0 毫秒中断）

// ④ 丢包恢复：每个 Stream 独立重传
// TCP：一个包丢了，所有 Stream 等
// QUIC：只重传当前 Stream 的包，其他 Stream 继续

// ⑤ 内置 TLS 1.3
// QUIC = UDP + TLS 1.3（不像 TCP+TLS 那样分离）
// TLS 握手和 QUIC 握手合并，减少 RTT
```

#### HTTP/3 在 Go 中的支持

```go
// Go 标准库 net/http 不支持 HTTP/3
// 需要使用第三方库：quic-go

import (
    "github.com/quic-go/quic-go/http3"
)

// 使用 quic-go 构建 HTTP/3 服务
srv := &http3.Server{
    Addr: ":8443",
    Handler: mux,
}
srv.ListenAndServe()

// 客户端
client := &http.Client{
    Transport: &http3.RoundTripper{},
}
```

---

### 4. 协议对比表

| 特性 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|----------|--------|--------|
| 传输层 | TCP | TCP | **UDP** |
| 多路复用 | ❌ | ✅ Stream | ✅ Stream（独立流控）|
| 头部压缩 | ❌ | HPACK | **QPACK** |
| 服务端推送 | ❌ | ✅ | ✅ |
| 队头阻塞 | ❌ 应用层 | ⚠️ TCP层 | ✅ 无 |
| 连接迁移 | ❌ | ❌ | ✅ CID 机制 |
| TLS | 独立 | 独立 | **内置 QUIC** |
| 首包即加密 | ❌ | ❌ | ✅ 0-RTT |
| 连接建立 RTT | 1 RTT + TLS | 1 RTT + TLS | **0~1 RTT** |
| Browser 支持 | ✅ 100% | ✅ 97% | ✅ 75%+ |

---

### 5. 生产环境选型

```go
// Go HTTP 服务最佳实践：同时支持 HTTP/1.1、HTTP/2、HTTP/3

// Nginx 层做协议协商（ALPN）
// upstream backend {
//     server 127.0.0.1:8080;
// }

// Nginx 配置支持 HTTP/2
// server {
//     listen 443 ssl http2;
//     ssl_certificate cert.pem;
//     ssl_certificate_key key.pem;
// }

// Go 服务只负责处理请求
func main() {
    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  30 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    }
    srv.ListenAndServe()
}

// 何时用 HTTP/2：
//   ① 资源多（CSS/JS/图片 > 30 个）
//   ② 高并发 API
//   ③ 移动端弱网（HTTP/2 更好）

// 何时用 HTTP/3：
//   ① 高丢包率网络（4G/公网）
//   ② 重视首屏加载速度
//   ③ 已有 quic-go 集成经验
```

---

## ❓ 高频追问

**Q：HTTP/2 的 Server Push 为什么没有被广泛使用？**

> 浏览器缓存机制会拒绝重复推送的资源，而服务器不知道资源是否已被缓存。推送的资源可能根本不需要。HTTP/2 规范本身也有争议。实际生产中，CDN + 资源内联/合并更可靠。

**Q：0-RTT 的重放攻击是什么？如何防范？**

> 0-RTT 用上次会话的票据直接发数据，如果攻击者截获了这个数据包（离线攻击），可以在新会话中重放，导致非幂等请求被执行。防范：1）服务端记录所有 0-RTT Ticket 的使用时间戳，拒绝旧票据；2）业务层使用幂等请求；3）重要操作禁用 0-RTT。

**Q：HTTP/2 可以同时传大文件和小文件吗？**

> 可以，但大文件会挤压小文件的带宽（TCP 层公平，但不区分 Stream 优先级）。Go 的 http2 库支持 SETTINGS_NO_RFC7540_PRIORITIES 可以禁用优先级机制。实际建议：静态资源走 CDN（HTTP/1.1 或 HTTP/2），API 走 HTTP/2，实时数据走 WebSocket/GRPC。

**Q：HTTP/3 真的比 HTTP/2 快吗？**

> 在丢包率 > 1% 的网络中，HTTP/3 明显更快。在丢包率 < 1% 的高质量网络中，两者差异不大。在 localhost/局域网中，HTTP/2 可能更快（QUIC 的软件实现有额外 CPU 开销）。生产环境：先测，再决定。

---

## 📚 延伸阅读

- [RFC 9113 - HTTP/2](https://www.rfc-editor.org/rfc/rfc9113)
- [RFC 9114 - HTTP/3](https://www.rfc-editor.org/rfc/rfc9114)
- [RFC 9000 - QUIC](https://www.rfc-editor.org/rfc/rfc9000)
- [quic-go 库](https://github.com/quic-go/quic-go)
- [Chromium HTTP/3 实验数据](https://www.chromium.org/ QUIC)
