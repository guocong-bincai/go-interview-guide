# TCP 半连接队列、全连接队列与 SYN Flood 防护

> 考察频率：★★★★☆  优先级：P1
> 关键词：SYN 队列、Accept 队列、backlog、somaxconn、tcp_max_syn_backlog、SYN Flood、SYN Cookie、tcp_abort_on_overflow、ListenOverflows

---

## 🎯 面试官考察意图

这是**"TCP 握手"的二次深挖题**。面试官会顺着"三次握手"往下问：

> 服务器收到 SYN 之后，把连接放在哪？→ 半连接队列 → 队列满了怎么办？→ 丢包 / SYN Cookie → 那 Accept 队列是什么？→ `listen()` 的 backlog 参数管哪个队列？

能答清楚"两个队列 + 三个内核参数 + 溢出后的现象"，说明你理解 Linux 网络栈，而不是只背了握手图。

**这道题的实战价值在于：它能解释两个高频线上现象——**
1. 服务没挂但客户端连不上（Connect timeout）；
2. `ss -lnt` 显示 `Send-Q` 有值，`netstat -s` 里 `ListenOverflows` 一直涨。

---

## ⚡ 核心答案（30秒）

**TCP 建连过程中有两个队列：**

| 队列 | 存放什么 | 内核参数 | 溢出后果 |
|------|---------|---------|---------|
| **半连接队列（SYN Queue）** | 收到 SYN、已回 SYN+ACK、但还没收到最终 ACK 的连接（状态 `SYN_RECV`） | `net.ipv4.tcp_max_syn_backlog` | 新 SYN 被丢弃 → 客户端重传 SYN → **连接超时** |
| **全连接队列（Accept Queue）** | 三次握手已完成、等待应用 `accept()` 取走的连接（状态 `ESTABLISHED`） | `min(backlog, net.core.somaxconn)` | 新完成握手的连接被丢弃（默认）或直接回 RST → **连接失败** |

**关键认知：**
> `listen(fd, backlog)` 的 `backlog` 参数**只影响全连接队列**；半连接队列大小由 `tcp_max_syn_backlog` 独立控制。
>
> 而且实际全连接队列长度 = `min(backlog, somaxconn)`，**所以 `somaxconn` 才是隐藏的天花板**。

**Go 里的坑：**
Go 的 `net.Listen("tcp", ...)` 底层调用 `listen(fd, backlog)`，`backlog` 取的是……**`somaxconn` 的值**（Go 源码里 `maxListenerBacklog()` 返回 `somaxconn`）。所以你没法通过代码调大队列，**只能改内核参数**。

---

## 🔬 深度展开

### 1. 完整时序：两个队列在握手过程中的位置

```
客户端                          服务端内核                      应用（Go 进程）
  │                                │                                │
  │ ──── SYN ────────────────────►│
  │                                │ 1. 创建 request_sock
  │                                │ 2. 放入【半连接队列】
  │                                │    (状态 SYN_RECV)
  │                                │
  │ ◄─── SYN + ACK ───────────────│
  │                                │
  │ ──── ACK ────────────────────►│
  │                                │ 3. 握手完成，从半连接队列移出
  │                                │ 4. 放入【全连接队列】
  │                                │    (状态 ESTABLISHED)
  │                                │                                │
  │                                │ ◄─── accept() ─────────────────│ 5. 应用取走
  │                                │ ─── 返回新 fd ────────────────►│
  │                                │                                │
  │ ◄═══════ 数据传输 ════════════►│◄══════════════════════════════►│
```

**两种队列溢出的不同表现：**

| 溢出队列 | 内核行为 | 客户端表现 | 服务端现象 |
|---------|---------|-----------|-----------|
| 半连接队列满 | 丢弃新 SYN（或启用 SYN Cookie） | 反复重传 SYN，最终 **connect timeout** | `netstat -s \| grep -i "SYNs to LISTEN"` 增长 |
| 全连接队列满 | 默认**忽略** ACK，客户端会以为连接建立了，但应用 accept 不到；若 `tcp_abort_on_overflow=1` 则回 RST | 连接"建立成功"但发数据无响应，超时；或直接 RST | `netstat -s \| grep -i overflow` 增长 |

> ⚠️ 全连接队列溢出的默认行为是**静默丢弃 ACK**——这会产生一个非常诡异的现象：客户端 `connect()` 成功（因为它收到了 SYN+ACK，发了 ACK 就认为连上了），但服务端一直不 accept，客户端发请求后石沉大海，直到超时。**这个现象几乎无法从客户端侧定位，必须看服务端指标。**

### 2. 三个关键内核参数

```bash
# 1) 半连接队列上限（SYN_RECV 状态的最大数量）
sysctl net.ipv4.tcp_max_syn_backlog
# 默认：内存 > 128MB 时通常为 1024 或 4096

# 2) 全连接队列上限（accept queue，所有监听的全局上限）
sysctl net.core.somaxconn
# 默认：老内核 128，新内核（4.1+）通常 4096

# 3) 全连接队列溢出时的行为
sysctl net.ipv4.tcp_abort_on_overflow
# 0（默认）：静默丢弃 ACK，客户端以为连上了 → 最难排查
# 1：直接回 RST，客户端立刻收到 connection reset → 好排查但体验差
```

**计算实际全连接队列长度：**

```
实际队列长度 = min(listen() 的 backlog 参数, net.core.somaxconn)
```

**Go 程序的实际值：**

```go
// Go 源码 net/sock_posix.go → listenFunc → syscall.Listen(s, backlog)
// backlog 来自 net/sock_linux.go:
func maxListenerBacklog() int {
    // 读取 /proc/sys/net/core/somaxconn
    // 返回 somaxconn 的值（上限 65535）
}
```

也就是说：**Go 会把 backlog 设成 `somaxconn`**，所以实际队列就是 `somaxconn`。想让队列更大？只能调 `somaxconn`：

```bash
sudo sysctl -w net.core.somaxconn=8192
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=8192
```

### 3. 怎么观测队列溢出

```bash
# 1) 看监听套接字的队列情况
ss -lnt
# State   Recv-Q  Send-Q   Local Address:Port
# LISTEN  0       4096     0.0.0.0:8080
#          ↑      ↑
#      当前全连  全连接队列
#      接队列中  的容量上限
#      待 accept
#      的连接数
```

> **重点**：`LISTEN` 状态下，`Recv-Q` 表示**当前全连接队列里等待 accept 的连接数**，`Send-Q` 表示队列容量。如果 `Recv-Q` 长期接近 `Send-Q`，说明 accept 速度跟不上。

```bash
# 2) 看累计溢出次数（最权威）
netstat -s | grep -i -E "listen|overflow|syn"
# 输出示例：
#   12345 times the listen queue of a socket overflowed
#   678   SYNs to LISTEN sockets dropped

# 或者用 nstat
nstat -az | grep -E "ListenOverflows|ListenDrops|TCPReqQFull"
# TcpExtListenOverflows  全连接队列溢出次数
# TcpExtListenDrops      入队失败丢弃次数
# TcpExtTCPReqQFullDrop  半连接队列溢出丢弃的 SYN 数
```

```bash
# 3) 看 SYN_RECV 状态连接数（超过一万基本可判定 SYN Flood）
ss -ant state syn-recv | wc -l
watch -n1 'ss -ant state syn-recv | wc -l'
```

### 4. 溢出的根因分析（重要：通常是应用 bug）

**全连接队列满了，90% 不是内核参数问题，而是应用的 accept 循环有问题。**

```
应用 accept 太慢的常见原因：
1. accept 后同步执行重业务逻辑（阻塞在 accept 循环里）
   ❌ 错误写法：
      for {
          conn, _ := ln.Accept()
          handleBlocking(conn)   // 同步处理，期间无法 accept 新连接！
      }
2. accept 循环里做了耗时初始化（加载配置、连数据库）
3. goroutine 泄漏导致内存/调度压力，accept 被饿死
4. CPU 打满 / GC STW 时间过长，accept 调用被延迟
```

```go
// ✅ 正确写法：accept 后立刻交给 goroutine，主循环只做 accept
for {
    conn, err := ln.Accept()
    if err != nil {
        // 关键：区分临时错误，避免疯狂自旋
        if ne, ok := err.(net.Error); ok && ne.Temporary() {
            time.Sleep(5 * time.Millisecond) // 或指数退避
            continue
        }
        return err
    }
    go handleConn(conn) // 立即返回 accept
}
```

> Go 标准库的 `http.Server.Serve` 已经正确处理了"accept 后开 goroutine"和"临时错误退避"，并且有 `net/http` 的内置连接限制。**如果你自己写 TCP 服务器，这两个点必须抄对。**

### 5. SYN Flood 攻击与防护

**攻击原理：**
攻击者发送大量伪造源 IP 的 SYN 包。服务端回 SYN+ACK 到伪造 IP，永远等不到 ACK，半连接队列被塞满，正常客户端无法建连。

```
攻击者（伪造源IP）                  服务端
  │ ── SYN (src=1.1.1.1) ─────────►│ 放入半连接队列
  │ ── SYN (src=2.2.2.2) ─────────►│ 放入半连接队列
  │ ── SYN (src=3.3.3.3) ─────────►│ 放入半连接队列
  │          ... 成千上万 ...       │ 队列满！
  │                                │ 正常用户的 SYN 被丢弃 ❌
```

**防护手段（从内核到架构）：**

| 层级 | 手段 | 说明 |
|------|------|------|
| 内核 | **SYN Cookie** | 不分配半连接资源，把状态编码进 ISN。开启后即使队列满也能正常建连 |
| 内核 | `tcp_syncookies=1` | 默认通常已开（`net.ipv4.tcp_syncookies`） |
| 内核 | `tcp_synack_retries=2` | 减少 SYN+ACK 重传次数，加快半连接释放 |
| 内核 | `tcp_max_syn_backlog` 调大 | 只能缓解，治标 |
| 网络 | **云 WAF / DDoS 高防** | 云端清洗，最大流量在云外拦掉 |
| 网络 | **CDN 回源保护** | 隐藏源站 IP |
| 应用 | **限流（令牌桶）** | 应用层兜底限流 |
| 架构 | **四层负载均衡前置** | LVS/DPDK 集群抗流量 |

```bash
# 查看是否已开启 SYN Cookie
sysctl net.ipv4.tcp_syncookies
# 1 = 开启

# 优化 SYN Flood 抗性
sudo sysctl -w net.ipv4.tcp_syncookies=1
sudo sysctl -w net.ipv4.tcp_synack_retries=2       # 默认 5，减少重传
sudo sysctl -w net.ipv4.tcp_syn_retries=3          # 客户端 SYN 重传次数
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=8192
sudo sysctl -w net.core.somaxconn=8192
```

**SYN Cookie 原理（高频追问点）：**

```
正常流程：收到 SYN → 分配 request_sock 内存 → 记住状态 → 回 SYN+ACK
SYN Cookie：收到 SYN →
  1. 用 源IP/源端口/目的IP/目的端口/时间戳/secret 做一个加密哈希
  2. 把哈希值编码到 SYN+ACK 的初始序列号（ISN）里
  3. 不分配任何内存，直接丢弃本地状态
  收到 ACK 时 →
  1. 从 ACK 的 seq-1 里解出 cookie
  2. 重新计算哈希验证，验证通过才分配连接资源
```

**代价**：SYN Cookie 会丢失一些 TCP 选项（如窗口缩放、SACK）的协商能力，所以**只有在队列满时才启用**（`net.ipv4.tcp_syncookies=1` 的语义是"仅在溢出时使用"），而不是全程开启。

### 6. Nginx / 网关层的相关配置

```nginx
# Nginx 的 backlog 配置（对应 listen 的 backlog 参数）
server {
    listen 80 backlog=8192;
    # 实际生效值仍是 min(8192, somaxconn)
}

# Nginx 的 keepalive 连接池（针对上游 Go 服务）
upstream backend {
    server 127.0.0.1:8080;
    keepalive 256;              # 保持 256 个空闲长连接到后端
    keepalive_timeout 60s;
    keepalive_requests 10000;
}
```

> 上游连接池不足时，Nginx 会频繁建连，**反过来加剧后端全连接队列压力**。这是"连接池 → 队列溢出"的典型链路。

### 7. 完整排查决策树

```
客户端报连接超时 / 连接失败
        │
        ├─ 服务端进程还在吗？ ────► ps / health check
        │
        ├─ 端口在监听吗？ ─────► ss -lnt | grep :8080
        │
        ├─ 队列溢出计数在涨吗？ ──► nstat -az | grep -E "ListenOverflow|ListenDrop"
        │        │
        │        ├─ TcpExtListenOverflows 涨
        │        │     └─ 全连接队列满
        │        │         ├─ Recv-Q 长期接近 Send-Q → accept 慢
        │        │         │    └─ 查应用：accept 循环是否被阻塞？CPU 打满？GC？
        │        │         └─ 服务处理慢导致积压 → 查 pprof / 慢请求
        │        │
        │        └─ TcpExtTCPReqQFullDrop 涨
        │              └─ 半连接队列满
        │                  ├─ SYN_RECV 数异常多 → 疑似 SYN Flood
        │                  └─ 确认 SYN Cookie 已开启 + 调大 backlog
        │
        └─ 都是 0？ ──► 不是队列问题，查防火墙 / iptables / conntrack 表满
```

**别忘了一个常见凶手：`nf_conntrack` 表满**

```bash
# conntrack 表满会导致新连接被直接丢弃，现象和队列溢出类似
dmesg | grep -i "nf_conntrack: table full"
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_count
```

```bash
# 临时调大
sudo sysctl -w net.netfilter.nf_conntrack_max=1048576
sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=600
```

---

## ❓ 高频追问

### Q1：`listen()` 的 backlog 参数到底管哪个队列？

**传统答案**：只影响全连接队列（accept queue）。

**现代 Linux（4.1+）的实际情况更微妙**：`backlog` 会影响全连接队列，同时内核也用 `backlog` 参与计算半连接队列的容量（`tcp_max_syn_backlog` 与 `backlog` 取较小值的逻辑在不同内核版本有差异）。

**面试建议回答**：
> 严格来说 backlog 管的是全连接队列，实际生效值还要和 `somaxconn` 取 min。半连接队列由 `tcp_max_syn_backlog` 控制。老版本内核还有 "backlog 只按 3/4 计算" 的细节（防止 syn flood 时的内存放大），所以生产中不要靠参数精确控制队列，而是**监控溢出计数**。

### Q2：为什么全连接队列溢出时客户端反而"连接成功"了？

三次握手在**服务端内核**完成，不需要应用的 accept 参与。ACK 到达时内核把连接放进全连接队列并标记 ESTABLISHED，客户端侧看到的是"连接已建立"。

如果此时队列满：
- `tcp_abort_on_overflow=0`（默认）→ 内核**丢弃这个 ACK**，连接在服务端处于"客户端已发 ACK 但服务端不认"的状态；客户端发数据后，服务端认为连接无效（回到 SYN_RECV 状态等待重传的 ACK），最终客户端超时。
- `tcp_abort_on_overflow=1` → 回 RST，客户端立刻报 `connection reset by peer`。

**所以：默认配置下，队列溢出是一个"客户端显示连接成功、但请求无响应"的诡异现象。**

### Q3：Go 服务需要设置 backlog 吗？

**不需要，也设不了。** Go 的 `net.Listen` 不接受 backlog 参数，内部固定用 `maxListenerBacklog()`（即 `somaxconn`）。想要更大的队列只能改内核参数。

如果确实需要精细控制，可以用 `golang.org/x/sys/unix` 手动 `Socket` + `Bind` + `Listen`，再 `os.NewFile` + `net.FileListener` 转成 `net.Listener`：

```go
// 手动控制 backlog（一般不需要，除非明确知道自己在做什么）
fd, _ := unix.Socket(unix.AF_INET, unix.SOCK_STREAM, 0)
unix.SetsockoptInt(fd, unix.SOL_SOCKET, unix.SO_REUSEADDR, 1)
sa := &unix.SockaddrInet4{Port: 8080}
unix.Bind(fd, sa)
unix.Listen(fd, 8192) // ← 自定义 backlog
f := os.NewFile(uintptr(fd), "listener")
ln, _ := net.FileListener(f)
```

### Q4：SYN Cookie 有什么缺点，为什么不默认全程开启？

| 缺点 | 说明 |
|------|------|
| 丢失 TCP 选项 | 窗口缩放、SACK、时间戳等选项无法在 SYN 阶段协商成功，可能影响长肥管道（LFN）吞吐 |
| 每个 ACK 都要算哈希 | 轻微 CPU 开销（可忽略） |
| 无法兼容某些中间设备 | 少见 |

所以 Linux 的 `tcp_syncookies=1` 语义是 **"只在半连接队列溢出时才启用"**，兼顾性能与抗攻击。

### Q5：如何区分"队列满"和"CPU 打满导致 accept 慢"？

两者都会让 `Recv-Q` 堆积。判断依据：

| 指标 | 队列满 | CPU 打满 |
|------|-------|---------|
| `ListenOverflows` | 持续增长 | 也增长（因为 accept 跟不上） |
| CPU 使用率 | 可能正常 | 明显 100% |
| pprof profile | 无明显热点 | 有明显热点（GC、锁、计算） |
| goroutine 数 | 正常 | 可能暴涨 |
| 请求延迟 | 排队等待 | 全线变慢 |

**正确做法**：先看 CPU/GC 指标，再看队列指标，最后看应用 profile。**不要一上手就调 `somaxconn`**——那只是把问题推后。

---

## 📋 总结 checklist

- [ ] 能说出两个队列的名字、状态和对应参数
- [ ] 知道 `listen` 的 backlog 只影响全连接队列，且受 `somaxconn` 限制
- [ ] 知道 Go 内部把 backlog 设成 `somaxconn`
- [ ] 知道全连接队列溢出默认是"静默丢弃 ACK"而**不是**回 RST
- [ ] 会用 `ss -lnt` 看 `Recv-Q`/`Send-Q`，会用 `nstat` 看溢出计数
- [ ] 知道队列满的根因通常是 accept 循环被阻塞，而非内核参数
- [ ] 能讲清 SYN Cookie 的原理和代价
- [ ] 记得排查时考虑 `nf_conntrack` 表满

---

## 📚 延伸阅读

- [Linux 内核文档: ip-sysctl.txt](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)
- [RFC 4987: TCP SYN Flooding Attacks and Common Mitigations](https://www.rfc-editor.org/rfc/rfc4987)
- [Cloudflare: SYN flood attack](https://www.cloudflare.com/learning/ddos/syn-flood-ddos-attack/)
- [Go net: maxListenerBacklog 源码](https://github.com/golang/go/blob/master/src/net/sock_linux.go)
