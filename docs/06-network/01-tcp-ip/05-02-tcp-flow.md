# TCP 流量控制与拥塞控制

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：滑动窗口、流量控制、拥塞控制、慢启动、CUBIC、BBR、Reno

## 🎯 面试官考察意图

- 能否说清流量控制（Flow Control）和拥塞控制（Congestion Control）的区别
- 是否理解慢启动、拥塞避免、快速重传、快速恢复各阶段的窗口变化
- 是否了解现代拥塞控制算法（CUBIC、BBR）的特点
- 能否解释生产环境中遇到的网络问题（丢包判断、带宽浪费等）

---

## ⚡ 核心答案（30秒）

> **流量控制**：解决的是发送方太快、接收方来不及处理的问题。通过接收方告知的窗口大小（rwnd）来限制发送方的发送速率，是端到端的控制。
>
> **拥塞控制**：解决的是整个网络拥塞的问题。通过拥塞窗口（cwnd）动态调整发送速率，核心机制是慢启动→拥塞避免→快速重传→快速恢复。常见的拥塞算法有 Reno、CUBIC（Linux 默认）、BBR。

---

## 🔬 深度展开

### 一、流量控制（Flow Control）

#### 工作原理

流量控制是**接收方**向**发送方**反馈自己能处理多少数据的机制，防止发送方超过接收方的处理能力。

```
发送方（发送缓冲区）                接收方（接收缓冲区）
    [  发送窗口  ]    ----TCP报文---->    [  接收窗口  ]
    <---- rwnd ----（TCP头部 Window Size）
```

接收方在 TCP 头部携带 `Window Size` 字段，告知发送方自己还有多少缓冲区空间。

#### 滑动窗口工作过程

```go
// 滑动窗口示意图（简化）
// 窗口大小 = 5
// [ | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 ]
//  ^已确认  ^已发送未确认  ^窗口内未发送  ^窗口外

// 假设窗口大小为5，发送了1-5，收到ACK=3（确认1-2收到）
// 窗口滑动：
// [ 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 ]
//           ^已确认  ^已发送未确认  ^窗口内未发送
// 可以发送 6 了
```

#### Zero Window 问题（窗口枯竭）

如果接收方缓冲区满了，会通知发送方 `Window = 0`，发送方停止发送数据。

```go
// Zero Window 场景
// 接收方：缓冲区满了，通知 Window = 0
// 发送方：收到后停止发送，启动 persist timer
// 接收方：处理完数据后，发送 Window Update（Window > 0）

// Zero Window Probe：即使窗口为0，TCP仍会定期发送1字节探针
// 防止接收方处理完数据后忘记通知（常见bug）
```

#### 糊涂窗口综合征（Small Window Syndrome）

接收方每次只释放1个字节就通知窗口为1，发送方就只发1个字节，效率极低。

**解决方案**：
- 接收方：等待缓冲区达到一定大小（如 MSS）再通知窗口
- 发送方：等待窗口足够大（或达到 MSS）再发送

---

### 二、拥塞控制（Congestion Control）

#### 核心思想

拥塞控制是**整个网络**（不只是接收方）负载过高时的控制机制。通过动态调整 `cwnd`（拥塞窗口）来控制发送速率。

```
实际发送窗口 = min(rwnd（接收窗口）, cwnd（拥塞窗口）)
```

#### 四个核心算法

##### 1. 慢启动（Slow Start）

**目的**：不要一开始就发大量数据，先探测网络容量。

```go
// cwnd 初始值很小（通常为 1 MSS）
// 每收到一个 ACK，cwnd += 1 MSS
// 每经过一个 RTT，cwnd 翻倍

// 示例：
// RTT 0: cwnd = 1 MSS，发送 1 个包
// RTT 1: cwnd = 2 MSS，发送 2 个包
// RTT 2: cwnd = 4 MSS，发送 4 个包
// RTT 3: cwnd = 8 MSS，发送 8 个包
// ...
// 直到达到 ssthresh（慢启动阈值）
```

**慢启动结束条件**：
- cwnd ≥ ssthresh：进入拥塞避免
- 发生丢包：进入快速重传

##### 2. 拥塞避免（Congestion Avoidance）

**目的**：在接近网络容量时，缓慢增加发送速率。

```go
// 每收到一个 ACK，cwnd += 1 MSS/cwnd
// 即每经过一个 RTT，cwnd += 1 MSS

// 示例（cwnd = ssthresh = 8 MSS）：
// RTT 0: cwnd = 8 MSS
// RTT 1: cwnd = 9 MSS（+1）
// RTT 2: cwnd = 10 MSS（+1）
// ...
```

##### 3. 快速重传（Fast Retransmit）

当发送方收到**3个重复 ACK**（说明有数据包丢失，但网络还通），不等超时就立即重传丢失的包。

```
原始数据：1, 2, 3, 4, 5, 6
丢失的包：3
收到 ACK：ack=3, ack=3, ack=3（3个重复ACK）
          -> 立即重传数据包 3（不等超时）
```

##### 4. 快速恢复（Fast Recovery）

收到3个重复ACK后：
- ssthresh = cwnd / 2（阈值减半）
- cwnd = ssthresh + 3 MSS（加3个包，替代已离开网络的3个包）
- 进入拥塞避免

```go
// Reno 快速恢复：
// cwnd = cwnd/2 + 3
// 每收到一个重复 ACK，cwnd += 1
// 收到新 ACK，cwnd = ssthresh，进入拥塞避免
```

#### 经典算法对比

| 算法 | 特点 | 适用场景 |
|------|------|---------|
| **Reno** | 经典，丢包驱动 | 有线网络，低丢包 |
| **CUBIC** | Linux 默认，窗口增长平稳 | 通用场景，高速网络 |
| **BBR** | 不依赖丢包，吞吐量优先 | 长肥管道，高带宽高延迟 |
| **Westwood** | 基于带宽估计，适合无线 | 无线网络 |

#### CUBIC 算法（Linux 默认）

CUBIC 使用三次多项式函数调整 cwnd，不再是线性增长/减半：

```go
// CUBIC 核心公式
// W(t) = C * (t - K)^3 + Wmax
// C: 缩放因子
// t: 从上次拥塞到现在的时间
// K: 计算出的拐点时间
// Wmax: 上次拥塞时的窗口大小

// 特点：
// - 在 Wmax 附近增长慢（给网络恢复时间）
// - 快速增长到 Wmax（充分利用带宽）
// - 超过 Wmax 后继续增长（因为网络可能还能承受）
```

#### BBR 算法（Bottleneck Bandwidth and RTT）

BBR 不依赖丢包来判断拥塞，而是实时估算带宽和 RTT：

```go
// BBR 核心思想
// 发送速率 = Bottleneck Bandwidth / RTT
// 不等丢包就调整（主动探测）

// BBR 四阶段：
// 1. Startup（启动）：快速增长带宽
// 2. Drain（排空）：发现在扩容，慢慢排空
// 3. Probe BW（探测带宽）：周期性增加/减少探测
// 4. Probe RTT：短暂降低窗口探测 RTT

// 优势：对丢包不敏感，高带宽网络表现好
// 劣势：在 bufferbloat（缓冲膨胀）场景可能加剧问题
```

---

### 三、生产问题排查

#### 问题1：TCP 连接正常但速度慢

**可能原因**：
- 接收方窗口太小（缓冲区设置不当）
- 网络拥塞（丢包导致大量重传）
- 带宽被占满

**排查方法**：
```bash
# 查看 TCP 连接状态
netstat -ant | awk '/^tcp/ {s[$NF]++} END {for(k in s) print k, s[k]}'

# 查看重传率
ss -i
# 如果 RetransSegs > 1%，说明有丢包

# 查看带宽
iperf3 -c <server_ip>
```

#### 问题2：高并发下吞吐量上不去

**常见原因**：拥塞窗口太小，连接建立初期太慢

**解决方案**：
- 使用连接池复用连接
- 调整 initcwnd（初始窗口）：`ip route change ... initcwnd 10`
- 使用 HTTP/2 或 QUIC（多路复用，减少握手开销）

#### Go 中调整 TCP 参数

```go
// 设置 TCP 连接读写缓冲区大小
conn, err := net.Dial("tcp", "addr")
tcpConn := conn.(*net.TCPConn)

// 设置读写缓冲区（单位：字节）
// 通常设置为 4KB - 64KB
tcpConn.SetReadBuffer(64 * 1024)
tcpConn.SetWriteBuffer(64 * 1024)

// SetNoDelay 禁用 Nagle 算法，小数据包立即发送
// 对延迟敏感的场景（如 RPC）很有用
tcpConn.SetNoDelay(true)
```

---

## 💻 面试手写题

### 题目：滑动窗口实现

实现一个固定大小的滑动窗口限流器（Token Bucket 的滑动窗口版本）。

```go
package main

import (
	"container/list"
	"fmt"
	"sync"
	"time"
)

// SlidingWindowLimiter 滑动窗口限流器
// 窗口大小 window，时间窗口内最多允许 count 个请求
type SlidingWindowLimiter struct {
	mu      sync.Mutex
	window  time.Duration // 窗口大小
	max     int64         // 窗口内最大请求数
	hits    *list.List     // 存储请求时间戳
}

// NewSlidingWindowLimiter 创建限流器
func NewSlidingWindowLimiter(window time.Duration, max int64) *SlidingWindowLimiter {
	return &SlidingWindowLimiter{
		window: window,
		max:    max,
		hits:   list.New(),
	}
}

// Allow 检查是否允许请求通过
func (l *SlidingWindowLimiter) Allow() bool {
	l.mu.Lock()
	defer l.mu.Unlock()

	now := time.Now()
	cutoff := now.Add(-l.window)

	// 移除窗口外的请求
	for l.hits.Len() > 0 && l.hits.Front().Value.(time.Time).Before(cutoff) {
		l.hits.Remove(l.hits.Front())
	}

	// 检查窗口内请求数
	if int64(l.hits.Len()) < l.max {
		l.hits.PushBack(now)
		return true
	}
	return false
}

// AllowWithCount 返回剩余请求数
func (l *SlidingWindowLimiter) AllowWithCount() (bool, int64) {
	l.mu.Lock()
	defer l.mu.Unlock()

	now := time.Now()
	cutoff := now.Add(-l.window)

	for l.hits.Len() > 0 && l.hits.Front().Value.(time.Time).Before(cutoff) {
		l.hits.Remove(l.hits.Front())
	}

	remaining := l.max - int64(l.hits.Len())
	if remaining > 0 {
		l.hits.PushBack(now)
		return true, remaining
	}
	return false, 0
}

func main() {
	limiter := NewSlidingWindowLimiter(time.Second, 3)

	for i := 0; i < 5; i++ {
		ok, remain := limiter.AllowWithCount()
		fmt.Printf("请求 %d: %v, 剩余: %d\n", i+1, ok, remain)
	}

	// 等待窗口滑动
	time.Sleep(time.Second)

	ok, _ := limiter.AllowWithCount()
	fmt.Printf("等待1秒后: %v\n", ok)
}
```

---

## 📋 总结 checklist

- [ ] 能说清流量控制和拥塞控制的本质区别
- [ ] 能画出慢启动→拥塞避免→快速重传→快速恢复的状态转换图
- [ ] 知道 cwnd 和 rwnd 的关系
- [ ] 了解 CUBIC（Linux 默认）和 BBR 的核心区别
- [ ] 能解释为什么 BBR 在高带宽高延迟网络表现更好
- [ ] 生产中遇到过哪些 TCP 调优问题
