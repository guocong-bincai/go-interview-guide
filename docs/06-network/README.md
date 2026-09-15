# 06 Network 模块

> 📌 共 25 道高频面试题 ｜ ✅ 已按面试频率排序（★★★★★ → ★★★☆☆）

---

## 📋 题目索引（点击直接跳转阅读）

| 序号 | 📄 文件名 | 🔥 频率 | 💡 考点 & 跳转 |
|---|---|---|---|
| 01 | `01-00-network-basics.md` | `★★★★★` | [网络基础高频题：TCP vs UDP / HTTP vs HTTPS / DNS / Cookie vs Session vs Token](./01-00-network-basics.md) |
| 02 | `02-01-tcp-handshake.md` | `★★★★★` | [TCP 三次握手与四次挥手](./01-tcp-ip/02-01-tcp-handshake.md) |
| 03 | `03-03-tcp-sticky.md` | `★★★★★` | [TCP 粘包与拆包](./01-tcp-ip/03-03-tcp-sticky.md) |
| 04 | `04-01-http-versions.md` | `★★★★★` | [HTTP/1.1 vs HTTP/2 vs HTTP/3](./02-http/04-01-http-versions.md) |
| 05 | `04-epoll-netpoller.md` | `★★★★★` | [epoll 模型与 Go netpoller：IO多路复用、goroutine调度、C10K问题](./01-tcp-ip/04-epoll-netpoller.md) |
| 06 | `17-05-http-cache.md` | `★★★★★` | [HTTP 缓存机制：强缓存与协商缓存（Cache-Control / ETag / 304 / Vary）](./02-http/17-05-http-cache.md) |
| 07 | `18-06-cors.md` | `★★★★★` | [CORS 跨域原理与解决方案：同源策略、预检请求与 Go 实战](./02-http/18-06-cors.md) |
| 08 | `05-02-tcp-flow.md` | `★★★★☆` | [TCP 流量控制与拥塞控制](./01-tcp-ip/05-02-tcp-flow.md) |
| 09 | `06-02-https.md` | `★★★★☆` | [HTTPS 原理、TLS 握手流程、证书链与性能优化](./02-http/06-02-https.md) |
| 10 | `07-04-grpc-http2.md` | `★★★★☆` | [gRPC 基于 HTTP/2 的多路复用、流控与帧格式](./02-http/07-04-grpc-http2.md) |
| 11 | `08-01-common-attacks.md` | `★★★★☆` | [Web 安全：常见攻击与防御](./03-security/08-01-common-attacks.md) |
| 12 | `09-02-backend-security.md` | `★★★★☆` | [后端安全体系](./03-security/09-02-backend-security.md) |
| 13 | `10-03-authz-rbac-abac.md` | `★★★★☆` | [RBAC、ABAC 与统一权限设计](./03-security/10-03-authz-rbac-abac.md) |
| 14 | `15-uds-unix-domain.md` | `★★★★☆` | [Unix Domain Socket：UDS vs TCP loopback 性能对比、Go 实战、K8s sidecar](./02-http/15-uds-unix-domain.md) |
| 15 | `16-chunked-transfer.md` | `★★★★☆` | [HTTP Chunked Transfer Encoding：流式响应格式、Go SSE 实现、Nginx 配置陷阱](./02-http/16-chunked-transfer.md) |
| 16 | `19-05-nagle-delayed-ack.md` | `★★★★☆` | [Nagle 算法、延迟确认与 TCP_NODELAY：40ms 延迟毛刺的根因与排查](./01-tcp-ip/19-05-nagle-delayed-ack.md) |
| 17 | `20-06-tcp-backlog-synflood.md` | `★★★★☆` | [TCP 半连接/全连接队列与 SYN Flood 防护：backlog、somaxconn、SYN Cookie](./01-tcp-ip/20-06-tcp-backlog-synflood.md) |
| 18 | `21-07-http-5xx-debug.md` | `★★★★☆` | [HTTP 502 / 504 / 499 排查：Go 服务超时链路全景](./21-07-http-5xx-debug.md) |
| 19 | `22-08-l4-l7-load-balance.md` | `★★★★☆` | [四层 vs 七层负载均衡：LVS/DR、Nginx、健康检查与算法选型](./22-08-l4-l7-load-balance.md) |
| 20 | `11-00-url-to-response.md` | `★★★☆☆` | [输入 URL 到页面展示：后端视角完整链路](./11-00-url-to-response.md) |
| 21 | `12-04-tcp-keepalive.md` | `★★★☆☆` | [TCP Keepalive vs 应用层心跳](./01-tcp-ip/12-04-tcp-keepalive.md) |
| 22 | `13-03-websocket.md` | `★★★☆☆` | [WebSocket 原理、升级握手、与 HTTP 长轮询对比](./02-http/13-03-websocket.md) |
| 23 | `14-01-http2-priority.md` | `★★★☆☆` | [HTTP/2 优先级调度：RFC 9218 优先级信号、Server 调度优化、与 Go 1.27 DisableClientPriority](./03-http2-priority/14-01-http2-priority.md) |
| 24 | `23-09-http-range-resume.md` | `★★★☆☆` | [HTTP Range 请求与断点续传：206、分片上传、秒传与大文件传输](./02-http/23-09-http-range-resume.md) |
| 25 | `24-10-network-troubleshooting.md` | `★★★☆☆` | [网络排查工具箱：tcpdump / ss / mtr / iftop 实战](./24-10-network-troubleshooting.md) |

---

## 🗺️ 学习建议

1. **地基三件套**（必背）：TCP 三次握手四次挥手 → HTTP 版本演进 → HTTPS/TLS 握手
2. **高频实战**：HTTP 缓存、CORS 跨域、502/504/499 排查、Nagle 40ms 毛刺
3. **架构层**：四层/七层负载均衡、TCP 队列与 SYN Flood、epoll/netpoller
4. **动手项**：用 `tcpdump` 抓一次三次握手、用 `ss -lnt` 看一次队列、用 `mtr` 看一次链路质量
