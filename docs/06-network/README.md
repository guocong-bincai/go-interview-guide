# 06 Network 模块

> 📌 共 17 道高频面试题 ｜ ✅ 已按面试频率排序（★★★★★ → ★☆☆☆）

---

## 📋 题目索引（点击直接跳转阅读）

| 序号 | 📄 文件名 | 🔥 频率 | 💡 考点 & 跳转 |
|---|---|---|---|
| 01 | `01-00-network-basics.md` | `★★★★★` | [网络基础高频题：TCP vs UDP / HTTP vs HTTPS / DNS / Cookie vs Session vs Token](./01-00-network-basics.md) |
| 02 | `02-01-tcp-handshake.md` | `★★★★★` | [TCP 三次握手与四次挥手](./01-tcp-ip/02-01-tcp-handshake.md) |
| 03 | `03-03-tcp-sticky.md` | `★★★★★` | [TCP 粘包与拆包](./01-tcp-ip/03-03-tcp-sticky.md) |
| 04 | `04-01-http-versions.md` | `★★★★★` | [HTTP/1.1 vs HTTP/2 vs HTTP/3](./02-http/04-01-http-versions.md) |
| 05 | `04-epoll-netpoller.md` | `★★★★★` | [epoll 模型与 Go netpoller：IO多路复用、goroutine调度、C10K问题](./01-tcp-ip/04-epoll-netpoller.md) |
| 06 | `05-02-tcp-flow.md` | `★★★★☆` | [TCP 流量控制与拥塞控制](./01-tcp-ip/05-02-tcp-flow.md) |
| 07 | `06-02-https.md` | `★★★★☆` | [HTTPS 原理、TLS 握手流程、证书链与性能优化](./02-http/06-02-https.md) |
| 08 | `07-04-grpc-http2.md` | `★★★★☆` | [gRPC 基于 HTTP/2 的多路复用、流控与帧格式](./02-http/07-04-grpc-http2.md) |
| 09 | `08-01-common-attacks.md` | `★★★★☆` | [Web 安全：常见攻击与防御](./03-security/08-01-common-attacks.md) |
| 10 | `09-02-backend-security.md` | `★★★★☆` | [后端安全体系](./03-security/09-02-backend-security.md) |
| 11 | `10-03-authz-rbac-abac.md` | `★★★★☆` | [RBAC、ABAC 与统一权限设计](./03-security/10-03-authz-rbac-abac.md) |
| 12 | `15-uds-unix-domain.md` | `★★★★☆` | [Unix Domain Socket：UDS vs TCP loopback 性能对比、Go 实战、K8s sidecar](./02-http/15-uds-unix-domain.md) |
| 13 | `16-chunked-transfer.md` | `★★★★☆` | [HTTP Chunked Transfer Encoding：流式响应格式、Go SSE 实现、Nginx 配置陷阱](./02-http/16-chunked-transfer.md) |
| 14 | `11-00-url-to-response.md` | `★★★☆☆` | [输入 URL 到页面展示：后端视角完整链路](./11-00-url-to-response.md) |
| 15 | `12-04-tcp-keepalive.md` | `★★★☆☆` | [TCP Keepalive vs 应用层心跳](./01-tcp-ip/12-04-tcp-keepalive.md) |
| 16 | `13-03-websocket.md` | `★★★☆☆` | [WebSocket 原理、升级握手、与 HTTP 长轮询对比](./02-http/13-03-websocket.md) |
| 17 | `14-01-http2-priority.md` | `★★★☆☆` | [HTTP/2 优先级调度：RFC 9218 优先级信号、Server 调度优化、与 Go 1.27 DisableClientPriority](./03-http2-priority/14-01-http2-priority.md) |

---
