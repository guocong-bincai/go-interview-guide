# TCP 三次握手与四次挥手

> 考察频率：★★★★★  难度：★★★☆☆
> 关键词：三次握手、四次挥手、TIME_WAIT、TCP 状态机、滑动窗口、拥塞控制

## 🎯 面试官考察意图

- 候选人是否理解 TCP 建立连接和断开连接的完整过程
- 能否画出状态转换图并解释每个状态的含义
- 能否从原理层面解释 CLOSE_WAIT、TIME_WAIT 过多的问题
- 是否有实际调优经验（SO_REUSEADDR、TCP_NODELAY 等）

---

## ⚡ 核心答案（30秒）

> **三次握手**：客户端发 SYN，服务器回 SYN+ACK，客户端再发 ACK。目的是同步双方序列号，建立可靠的连接。为什么是三次而不是两次？因为双方都要确认对方的发送和接收能力都正常，两次只能确认一方的能力。

> **四次挥手**：主动关闭方发 FIN，被动关闭方回 ACK，然后处理完剩余数据再发 FIN，主动关闭方回 ACK。TCP 是全双工，需要双向都要确认关闭。四次比三次多是因为服务器收到 FIN 后，数据可能还没发完，所以先回 ACK，等数据发完再发 FIN。

---

## 🔬 深度展开

### 1. 三次握手详解

#### 状态转换图

```
客户端                                    服务器
  │                                        │
  │  ──────── SYN=1, seq=x ────────────>  │  SYN_SENT
  │         (消耗一个序列号)                  │  LISTEN
  │                                        │
  │  <────── SYN=1, ACK=1, seq=y,          │
  │              ack=x+1 ──────────────    │  SYN_RCVD
  │         (消耗一个序列号)                  │
  │                                        │
  │  ──────── ACK=1, seq=x+1,              │
  │              ack=y+1 ──────────────>   │  ESTABLISHED
  │                                        │
  │  ESTABLISHED                           │  ESTABLISHED
  │                                        │
```

#### 为什么序列号从 ISN 开始？

```
ISN（Initial Sequence Number）随机生成，防止：
① 旧连接的包被新连接误收（网络延迟的包）
② 安全性（防止序列号被猜测攻击）

RFC 793 规定：ISN = M + F(local, remote, port1, port2)
M 是一个计时器，每 4μs 增 1
F 是一个哈希算法，保证每个连接有唯一的 ISN
```

#### 第三次握手可以携带数据吗？

```
可以，但通常不携带。

第一次握手：SYN 不能携带数据，但会消耗序列号
第二次握手：SYN+ACK 可以携带数据（少见）
第三次握手：ACK 可以携带数据（常见）

携带数据时：
  客户端 ── SYN, seq=x, data_len=100 ──> 服务器
  客户端 ── ACK, ack=x+1+100 ──────────> 服务器
  
  服务器收到第三次握手后，客户端就已 ESTABLISHED
  第三次握手带数据的优化：HTTP 的 TCP 快速打开（TFO Cookie）
```

#### 半连接队列与全连接队列

```go
// 服务器端两个队列
// 半连接队列（SYN Queue）：完成第一次握手，等待第二次握手
// 全连接队列（Accept Queue）：完成三次握手，等待应用 accept()

// 生产问题：队列满
// ① 半连接队列满：服务器不响应 SYN，服务降级
// ② 全连接队列满：服务器回退，重传 SYN+ACK（Linux 默认行为）

// 查看队列溢出
// netstat -s | grep -i "overflow"
// netstat -s | grep -i "listend"

sysctl net.ipv4.tcp_max_syn_backlog          // 半连接队列上限
sysctl net.core.netdev_max_backlog            // 网卡队列长度
sysctl net.core.somaxconn                     // 全连接队列上限（Accept Queue）
```

---

### 2. 四次挥手详解

#### 状态转换图

```
客户端                                    服务器
  │                                        │
  │  ──────── FIN=1, seq=u ────────────>  │  CLOSE_WAIT
  │         (客户端已无数据发送)              │
  │                                        │
  │  <─────── ACK=1, ack=u+1 ──────────   │  ESTABLISHED
  │                                        │
  │  [此时：客户端进入 FIN_WAIT_2]           │
  │                                        │
  │                                        │  [处理剩余数据...]
  │                                        │
  │  <─────── FIN=1, seq=w ────────────   │  LAST_ACK
  │                                        │
  │  ──────── ACK=1, ack=w+1 ───────────>  │
  │         (进入 TIME_WAIT)               │
  │                                        │
  │  [等待 2MSL]                           │  CLOSED
  │  CLOSED                                │
```

#### 关键状态解析

| 状态 | 含义 | 常见问题 |
|------|------|----------|
| FIN_WAIT_1 | 发出 FIN，等待对方 ACK | 对端不 ACK → 一直停留（网络问题）|
| FIN_WAIT_2 | 收到 ACK，等待对方 FIN | 等待期间对方崩溃 → 需配置 timeout |
| CLOSE_WAIT | 收到 FIN，回复 ACK，等待本地处理 | **最常见问题**：应用层未 close socket |
| LAST_ACK | 被动关闭方发完数据后发 FIN，等待 ACK | 对端不 ACK → 一直停留 |
| TIME_WAIT | 主动关闭方收到 FIN+ACK，发 ACK，等待 2MSL | **最常见问题**：短连接过多导致端口耗尽 |

#### TIME_WAIT 详解（面试高频）

```
TIME_WAIT 存在的两个原因：
① 确保对方收到最后的 ACK（如果 ACK 丢了，对端会重发 FIN）
② 让旧连接的报文在网络中完全消失（下同端口的新连接不受影响）

MSL（Maximum Segment Lifetime）：报文最大生存时间，通常 60 秒
2MSL = 60~120 秒

TIME_WAIT 过多的问题：
  场景：高并发短连接服务（如 HTTP API）
  现象：连接失败，报错 "Cannot assign requested address"
  原因：每个 TIME_WAIT 连接占用一个 port（client port），端口 16bit = 65535
  解决：
    ① 客户端使用连接池复用连接
    ② 服务端主动关闭连接 → 改用 HTTP Keep-Alive
    ③ 调整 net.ipv4.tcp_tw_reuse = 1（让新连接复用 TIME_WAIT 状态的端口）
    ④ 调整 net.ipv4.tcp_fin_timeout（缩短 TIME_WAIT 时长）
```

```go
// Linux 内核参数调优
// /etc/sysctl.conf

# 开启 TIME_WAIT 端口复用（客户端重要）
net.ipv4.tcp_tw_reuse = 1

# FIN_WAIT_2 超时（默认 60s）
net.ipv4.tcp_fin_timeout = 30

# 允许更多的 TIME_WAIT（默认 262144）
net.ipv4.tcp_max_tw_buckets = 500000

# 开启后，服务端主动关闭时重用 TIME_WAIT 端口
net.ipv4.tcp_timestamps = 1  # 需要配合这个参数
```

---

### 3. TCP 状态机全景图

```
                              LISTEN
                                 │
          ┌──────────────────────┴──────────────────────┐
          │                                             │
     SYN_SENT                                        SYN_RCVD
          │                                             │
          │                                             │
          └──────────┐              ┌───────────────────┘
                     │              │
               ┌─────┴──────┐       │
               │            │       │
           ESTABLISHED  CLOSED   ESTABLISHED
               │            │       │
               │            │       │
     ┌─────────┼────────────┐       │
     │         │            │       │
     │    CLOSE_WAIT    CLOSED    │
     │         │                   │
     │    ┌────┴────┐              │
     │    │         │              │
     │ LAST_ACK  CLOSING           │
     │    │         │              │
     │    │         └──────┐       │
     │    │                  │       │
     │    │              FIN_WAIT_1 │
     │    │                  │       │
     │    │    ┌─────────────┼───┐   │
     │    │    │             │   │   │
     │  CLOSED │       ┌─────┴──┐ │   │
     │         │       │        │ │   │
     │    FIN_WAIT_2   │    TIME_WAIT │
     │         │       │        │ │   │
     │         │       │        │ │   │
     │         └───────┘        │ │   │
     │                           │ │   │
     └───────────────────────────┘ │   │
                                   │   │
                              CLOSED ◄─┘
```

---

### 4. 滑动窗口与流量控制

#### 窗口机制

```
发送方                                 接收方
  │                                     │
  │  ──── 发送窗口（可连续发送） ────────>  │
  │                                     │
  │  [AAAAA|BBBBB|CCCCC|DDDDD|EEEEE]    │
  │      ↑        ↑       ↑      ↑      │
  │   已发送     已发送   已发   未发    │
  │   已ACK     已ACK   未ACK           │
  │                                     │
  │  <─── ACK + 窗口更新 ─────────────  │
  │       (ack=10, win=5000)            │
  │                                     │
```

#### Zero Window（零窗口）

```
场景：接收方缓冲区满了，无法接收更多数据
      ↓
接收方发 win=0（零窗口）
      ↓
发送方停止发送，启动 Zero Window Probe 定时器
      ↓
定期发 ZWP 包探测窗口大小
      ↓
窗口恢复后，接收方发 win>0
      ↓
发送方继续发送

Go 中的表现：
  服务端 read buffer 满了 → 客户端 write 阻塞
  HTTP/1.1 的 Content-Length 不匹配会导致服务端读不到完整数据
```

---

### 5. 拥塞控制（补充）

```
TCP 拥塞控制四算法（RFC 5681）：

① 慢启动（Slow Start）
   cwnd = 1 MSS → 指数增长至 ssthresh
   目的：探测网络带宽，避免一开始就发太多

② 拥塞避免（Congestion Avoidance）
   cwnd 线性增长（每 RTT +1）
   
③ 快速重传（Fast Retransmit）
   收到 3 个重复 ACK → 不等超时立即重传丢失的包

④ 快速恢复（Fast Recovery）
   快速重传后，cwnd 减半，线性增长

Go 中的意义：
  高并发场景下，如果 RTT 很大，cwnd 增长慢
  跨机房调用（20ms RTT）vs 同机房（0.5ms RTT）
  带宽相同的情况下，同机房吞吐量是跨机房的 40 倍
```

---

## ❓ 高频追问

**Q：四次挥手为什么不能变成三次？**

> 因为 TCP 是全双工，两个方向是独立的。服务器收到客户端的 FIN，只是说"客户端到服务器的方向关闭了"，但服务器可能还有数据要发给客户端。所以服务器先回 ACK（关闭了接收），等数据发完再发 FIN（关闭服务器到客户端的方向）。两次挥手不够。

**Q：TIME_WAIT 过多怎么排查和处理？**

> `netstat -an | grep TIME_WAIT | wc -l` 确认数量。`ss -s` 更高效。原因：短连接过多。方案：1）服务端开启 `tcp_tw_reuse`；2）客户端使用 HTTP Keep-Alive；3）应用层做连接池；4）调小 `tcp_fin_timeout`；5）如果是主动关闭方太多，考虑改为被动关闭（但要注意 CLOSE_WAIT）。

**Q：CLOSE_WAIT 过多怎么排查？**

> `netstat -an | grep CLOSE_WAIT | wc -l`。原因：应用层拿到 socket 后没有调用 close()。常见场景：代码 bug、资源泄漏。Go 中：defer resp.Body.Close() 没写、HTTP 客户端 response 没读完就丢弃。排查：用 `lsof -p <pid>` 看打开的 fd 数量。

**Q：TCP_NODELAY 的作用？**

> 默认 Nagle 算法会攒小包合并发送（减少网络小包），但对延迟敏感的场景（Redis、RPC）有害。开启 TCP_NODELAY 后立即发送。Go 的 net.Dialer 默认关闭 Nagle（TCP_NODELAY=true）。

**Q：三次握手期间发生了什么？**

> 服务器收到 SYN 后创建 TCB（Transmission Control Block），加入半连接队列（SYN Queue）。返回 SYN+ACK 后进入 SYN_RCVD。收到第三次 ACK 后，从半连接队列移到全连接队列（Accept Queue），应用层 accept() 取走。SYN Cookie 是对抗 SYN Flood 的机制：不用队列存储，而是用算法验证。

---

## 📚 延伸阅读

- [RFC 793 - TCP 规范](https://tools.ietf.org/html/rfc793)
- [RFC 1323 - TCP 扩展（时间戳、窗口缩放）](https://tools.ietf.org/html/rfc1323)
- [TCP 的性能问题汇总](https://packetlife.net/blog/2011/jun/2/ tcp-state-changes/)
- [Go 标准库 net 包分析](https://pkg.go.dev/net)
