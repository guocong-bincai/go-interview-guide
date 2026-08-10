# 网络基础高频题：TCP vs UDP / HTTP vs HTTPS / DNS / Cookie vs Session vs Token

> 考察频率：★★★★★  优先级：P1
> 关键词：TCP、UDP、三次握手、四次挥手、滑动窗口、HTTP、HTTPS、TLS、DNS、Cookie、Session、Token、JWT

---

## 面试官考察意图

网络基础是后端工程师的必备知识，高级工程师不仅要能回答表层区别，还要能**从协议设计层面解释原因、从性能角度量化差异、从实际生产问题出发给出解决方案**。这道题考察的是对整个网络协议栈的理解深度。

---

## 核心答案（30 秒版）

| 对比维度 | 答案一句话总结 |
|---------|--------------|
| **TCP vs UDP** | TCP 面向连接、可靠、有拥塞控制；UDP 无连接、不可靠、但性能高 |
| **HTTP vs HTTPS** | HTTPS = HTTP + TLS，提供加密和身份验证 |
| **DNS 解析** | 递归查询：浏览器 → 系统缓存 → 根/顶级/权威 DNS → 逐级返回 |
| **Cookie vs Session** | Cookie 存客户端（体积小、有状态）；Session 存服务端（安全但占用服务器资源） |
| **Session vs Token** | Session 依赖服务端存储；Token（JWT）是无状态的，服务端只验证签名 |

---

## 深度展开

### 1. TCP vs UDP 核心对比

#### 字段对比

| 特性 | TCP | UDP |
|------|-----|-----|
| 连接方式 | 面向连接（三次握手） | 无连接 |
| 可靠性 | 可靠（有确认、重传、排序） | 不可靠（无确认） |
| 传输速度 | 较慢（头部长、拥塞控制） | 快（头部仅 8 字节） |
| 流量控制 | 滑动窗口 | 无 |
| 拥塞控制 | 有（CUBIC/BBR） | 无 |
| 场景 | 文件传输、HTTP/HTTPS、邮件 | DNS、视频流、VoIP、游戏 |
| 首部开销 | 20~60 字节 | 8 字节 |

#### TCP 为什么需要三次握手？

```
目的：同步双方序列号（ISN），确认双方收发能力都正常

      客户端                              服务器
        │                                    │
        │ ──────── SYN, seq=x ─────────────>│  客户端：我要建立连接，我的初始序列号是 x
        │<────── SYN+ACK, seq=y, ack=x+1 ──│  服务器：我收到你的请求，我也准备好了，我的初始序列号是 y，同时确认收到 x
        │ ──────── ACK, seq=x+1, ─────────>│  客户端：我确认收到你的序列号 y，连接建立完成
        │           ack=y+1 ─────────────────>│
        ▼                                    ▼
      ESTABLISHED                        ESTABLISHED
```

**为什么不能两次？**
假设客户端发 SYN 但这个 SYN 在网络中**迷路**了很久才到达服务器（延迟到达），服务器收到后会误以为这是新请求并回复 SYN+ACK。但此时客户端已经放弃了这个连接，不会回复 ACK，服务器就会空等一个永远不会到来的 ACK，浪费资源。

#### TCP 为什么需要四次挥手？

```
      主动关闭方                            被动关闭方
        │                                      │
        │ ──────── FIN, seq=u ─────────────────>│  我发完数据，发起关闭
        │<─────── ACK, ack=u+1 ────────────────│  收到，但我可能还有数据没发完
        │              ...                    │  （这里多出来的一次，是 TCP 全双工特性）
        │<─────── FIN, seq=v ────────────────│  我也发完数据了
        │ ──────── ACK, ack=v+1 ────────────>│  收到，连接关闭
        ▼                                      ▼
      TIME_WAIT（等待 2MSL）                CLOSED
```

**为什么不能合并为三次？**
TCP 是全双工协议，两个方向的数据流是独立的。主动关闭方发 FIN 只表示"我这边数据发完了"，但**服务器可能还有数据在发送中**，所以必须等服务器也发完 FIN 才能关闭。

#### TIME_WAIT 过多的问题

| 问题 | 影响 | 解决方案 |
|------|------|---------|
| 端口被占用 | 短连接高并发时耗尽端口 | `sysctl -w net.ipv4.tcp_tw_reuse=1` |
| 服务器资源占用 | 每个 TIME_WAIT 占用一个连接 | 调小 `tcp_fin_timeout`（如 30s）|
| 内存占用 | 四元组（clientIP:port:serverIP:port）保留 | 客户端用连接池、服务器用 SO_REUSEADDR |

```go
// Go 中设置 SO_REUSEADDR
ln, err := net.Listen("tcp", ":8080")
// net.Listen 底层已设置 SO_REUSEADDR
```

#### 滑动窗口与流量控制

```
发送方                                          接收方
  │                                                │
  │◄─────── ACK + rwnd ────────────────────────────│  接收方告诉发送方：我还有多少缓冲区可用
  │                                                │
  │  ──── data[seq=1000] ───────────────────────>│
  │  ──── data[seq=1100] ───────────────────────>│  发送方根据 rwnd 调整发送速率
  │  ──── data[seq=1200] ───────────────────────>│
  │                                                │
```

#### TCP 拥塞控制算法

| 算法 | 特点 | 适用场景 |
|------|------|---------|
| CUBIC（默认） | 窗口增长较平滑，RTT 公平性好 | 通用场景 |
| BBR | 不基于丢包，利用带宽延迟积 | 高带宽高延迟、跨洲际 |
| Reno | 早期算法，已较少使用 | - |

```go
// Go 中设置拥塞控制算法（需要加载 cgo 或使用第三方库）
// 环境变量方式：GODEBUG=cc=BBR
```

### 2. HTTP vs HTTPS

#### 对比

| 维度 | HTTP | HTTPS |
|------|------|-------|
| 安全性 | 明文传输，可被窃听/篡改 | TLS 加密，防窃听/篡改/伪造 |
| 端口 | 80 | 443 |
| 协议层 | 直接基于 TCP | HTTP + TLS + TCP |
| 连接建立 | 三次 TCP 握手 | 三次 TCP + 一次 TLS 握手 |
| 性能 | 更快 | 略慢（~5-10%） |
| 证书 | 无 | 需要 CA 证书 |
|  SEO | 不友好 | 友好（Google 优先） |

#### TLS 1.3 握手优化

```
TLS 1.2（2-RTT）：
  TCP 握手 → ClientHello → ServerHello + 证书 + Key Exchange → ChangeCipherSpec → Finished
  耗时：2-RTT + TCP 握手

TLS 1.3（1-RTT，甚至 0-RTT）：
  客户端发 ClientHello（含 Key Share） → 服务器直接发证书 + Finished
  耗时：1-RTT
  
  0-RTT：使用 PSK（Pre-Shared Key），适合之前连接过的场景
```

#### 生产环境 HTTPS 最佳实践

```nginx
# nginx HTTPS 配置最佳实践
server {
    listen 443 ssl http2;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # 启用 TLS 1.3（nginx 1.13+）
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # 优先使用 ECDHE（前向安全）
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers on;
    
    # 开启 OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    
    # HSTS（强制 HTTPS）
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
}
```

### 3. DNS 解析流程

#### 完整解析链路

```
用户浏览器
    │
    ├─① 查询浏览器 DNS 缓存（Chrome 有约 1 分钟缓存）
    │     │
    │     ✗ 未命中
    │     ▼
    ├─② 查询操作系统 DNS 缓存（/etc/hosts 等）
    │     │
    │     ✗ 未命中
    │     ▼
    ├─③ 操作系统发起递归 DNS 查询：
    │     │
    │     ▼
    │   本地域名服务器（LDNS，如 114.114.114.114 或公司 DNS）
    │     │
    │     ├─ 查根域名服务器（全球 13 组 . root server）
    │     │     返回：.com 的权威服务器地址
    │     │
    │     ├─ 查 .com 顶级域名服务器
    │     │     返回：example.com 的权威 DNS 地址
    │     │
    │     └─ 查 example.com 权威 DNS
    │           返回：真实 IP 地址
    │
    ▼
返回 IP 给浏览器
```

#### DNS 记录类型

| 类型 | 作用 | 示例 |
|------|------|------|
| A | 域名 → IPv4 | example.com → 93.184.216.34 |
| AAAA | 域名 → IPv6 | example.com → 2606:2800:220:1:: |
| CNAME | 域名 → 另一个域名 | www.example.com → example.com |
| MX | 域名 → 邮件服务器 | example.com → mail.example.com |
| NS | 域名 → 权威 DNS | example.com → ns1.example.com |
| TXT | 域名 → 文本（SPF/DKIM） | 用于邮件验证 |

#### DNS 使用 UDP 还是 TCP？

- **53 端口**：DNS 主要用 **UDP**（查询/响应在 512 字节内）
- **TCP**：用于 DNS zone transfer（AXFR/IXFR），或响应超过 512 字节时

### 4. Cookie vs Session vs Token

#### Cookie

```
HTTP 无状态协议 → Cookie 解决方案：
浏览器                         服务器
  │                               │
  │<─────── Set-Cookie ──────────│  服务器在响应中设置 Cookie
  │                               │
  │ ──────── Cookie ────────────>│  浏览器自动在后续请求中带上 Cookie
  │                               │
```

**Cookie 的特点**：
- 存储在浏览器（客户端）
- 体积限制：单个 Cookie ≤ 4KB，整个域名下 Cookie 总数建议 < 20
- 可设置 `HttpOnly`（禁止 JS 访问，防止 XSS）
- 可设置 `Secure`（仅 HTTPS 下发送）
- 可设置 `SameSite`（防 CSRF）

```go
// Go 设置 Cookie 示例
http.SetCookie(w, &http.Cookie{
    Name:     "session_id",
    Value:    "abc123",
    HttpOnly: true,       // JS 不可访问
    Secure:   true,       // 仅 HTTPS
    SameSite: http.SameSiteStrictMode,
    MaxAge:   3600,       // 1 小时
    Path:     "/",
    Domain:   "example.com",
})
```

#### Session

```
浏览器                    服务器                    Redis/DB
  │                        │                          │
  │                        │<──── 读取 Session ─────>│
  │<─────── SessionID ──────│  （用户数据存在服务器）     │
  │ ── Cookie: SID ───────>│                          │
  │                        │<──── 查询用户数据 ───────>│
  │                        │
```

**Session 的特点**：
- 用户数据存在**服务器端**（Redis/MySQL 等）
- 只在 Cookie 中存 SessionID（安全）
- 缺点：需要服务端存储，扩展性差（分布式 Session 问题）

```go
// Go Gin 框架使用 Session
import (
    "github.com/gin-contrib/sessions"
    "github.com/gin-contrib/sessions/redis"
)

store, _ := redis.NewStore(10, "tcp", "redis:6379", "", []byte("secret"))
r.Use(sessions.Sessions("mysession", store))

func login(c *gin.Context) {
    session := sessions.Default(c)
    session.Set("user_id", 12345)
    session.Save()
}
```

#### Token（JWT 为例）

```json
// JWT 结构：Header.Payload.Signature
{
  "alg": "HS256",
  "typ": "JWT"
}
.
{
  "user_id": 12345,
  "exp": 1700000000,
  "iat": 1699999900
}
.
HMACSHA256(
  base64url(Header) + "." + base64url(Payload),
  secret
)
```

```
无状态 Token 流程：
1. 用户登录 → 服务器验证 → 用 secret 签名生成 JWT → 返回给客户端
2. 客户端存 JWT（localStorage/Cookie）
3. 后续请求带上 Authorization: Bearer <token>
4. 服务器只用 secret 验证 JWT 签名，不查库（无状态）
```

#### 三者对比

| 维度 | Cookie | Session | JWT Token |
|------|--------|---------|-----------|
| 存储位置 | 浏览器 | 服务器 | 浏览器（通常） |
| 体积 | ≤4KB | 任意 | 可较大（通常 < 8KB） |
| 扩展性 | 好（随请求发送） | 差（需 Session 共享） | 好（无状态） |
| 安全性 | 需防 XSS/CSRF | 较高（数据在服务端） | 防篡改，但防泄露需 HTTPS |
| 失效方式 | 客户端删除 / MaxAge | 服务端删除 | 客户端删除 / 加入黑名单 |
| 适用场景 | 简单状态 | 需要强安全性的登录 | 微服务、API 认证 |

---

## 高频追问

### Q1：TCP 连接如果接收方缓冲区满了，发送方会怎样？

发送方的**滑动窗口会收缩**，发送方根据 `rwnd=0` 停止发送，直到收到接收方窗口更新。当接收方处理完数据后，会发送 `ACK + rwnd > 0`，发送方才恢复发送。

### Q2：HTTPS 抓包原理是什么？

中间人（MITM）在客户端和服务端之间：
1. 客户端以为是连接"服务器"，实际连接的是中间人代理
2. 中间人代理**伪造证书**给客户端（浏览器校验证书链会发现不匹配，除非导入根证书）
3. 如果用户手动信任了根证书，TLS 加密就被绕过

**防御**：不要信任来路不明的根证书

### Q3：JWT 有什么安全问题？

1. **泄露后无法撤回**：服务端没有存储 JWT，泄露后攻击者可以一直使用直到过期
   - 解决：短期 JWT + refresh token；或维护 JWT 黑名单
2. **Payload 是 Base64 编码（可读，非加密）**：不要存敏感信息在 Payload
3. **无密钥更新机制**：密钥轮换时所有 JWT 需要重新签发

### Q4：DNS 污染是什么？如何应对？

DNS 污染：中间人返回错误的 DNS 解析结果（通常将域名解析到恶意 IP）。

应对方案：
- 使用可信的 DNS 服务器（114.114.114.114、Google 8.8.8.8）
- 使用 DNS over HTTPS（DoH）或 DNS over TLS（DoT）
- 在公司内网部署自己的 DNS 递归服务器

---

## 延伸阅读

- [TCP 协议详解 - RFC 793](https://tools.ietf.org/html/rfc793)
- [HTTP/1.1 规范 - RFC 7230](https://tools.ietf.org/html/rfc7230)
- [TLS 1.3 协议 - RFC 8446](https://tools.ietf.org/html/rfc8446)
- [JWT 官网](https://jwt.io/)
- [Cookie 安全配置 - Mozilla MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
