[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 并发编程](../README.md)

---

# Channel 底层原理

## 面试官考察意图

考察候选人对 Go 并发原语的掌握深度。
初级只能说出"channel 用于 goroutine 通信"，高级要能讲清楚 **hchan 结构、发送/接收的汇编流程、select 多路复用实现、以及环形队列/双向链表的具体作用**，并能结合死锁、泄漏等生产问题给出分析。

---

## 核心答案（30 秒版）

Channel 的核心是 **`hchan` 结构体**，包含：
- **环形队列**（ring buffer）：存储数据，FIFO
- **发送/接收索引**：高性能下标移动
- **阻塞 G 的双向链表**：等待接收/发送的 goroutine 队列
- **一把互斥锁**：保证并发安全

发送数据时，如果缓冲区不满，直接入队；如果缓冲区满，goroutine 就会**阻塞**并加入 `sendq` 等待队列，等对端接收时唤醒。

---

## 深度展开

### 1. hchan 核心结构

```go
// src/runtime/chan.go（简化版）
type hchan struct {
    qcount   uint           // 队列中数据个数
    dataqsiz uint           // 环形队列大小，0 表示无缓冲
    buf      unsafe.Pointer // 指向 dataqsiz 元素的底层数组

    elemsize uint16         // 元素大小
    closed   uint32         // channel 是否关闭
    elemtype *_type         // 元素类型

    sendx    uint           // 发送索引（环形队列）
    recvx    uint           // 接收索引（环形队列）

    recvq    waitq          // 等待接收的 goroutine 队列（双向链表）
    sendq    waitq          // 等待发送的 goroutine 队列（双向链表）

    lock     mutex          // 全局锁
}
```

```
有缓冲 channel 内存布局：

hchan {
    buf: [ ] [ ] [ ] [ ] [ ]    ← 环形队列，大小=dataqsiz
          ↑sendx          ↑recvx
    recvq: G1(等待接收)         ← 队列满时发送方在此等待
    sendq: G2(等待发送)         ← 队列空时接收方在此等待
}
```

### 2. 发送流程（ch <- value）

```
ch <- value
    │
    ▼ 加锁
    │
    ├─ recvq 不为空（有人等着接收）
    │      直接 memcpy 到对方 goroutine 的栈
    │      唤醒 recvq 头部 G，状态 → runnable
    │
    ├─ 缓冲区未满
    │      value 写入 buf[sendx]
    │      sendx = (sendx+1) % dataqsiz
    │      qcount++
    │
    └─ 缓冲区已满
           goroutine 阻塞，加入 sendq 尾部
          状态: Gwaiting → 等待接收方唤醒
    │
    ▼ 解锁
```

**核心语义：发送是 COPY，不是引用。** 数据从发送方 goroutine 栈拷贝到 channel 缓冲区（或接收方栈），发送完成后两者独立。

### 3. 接收流程（<-ch）

```go
v := <-ch   // 接收数据
v, ok := <-ch  // 接收并判断 channel 是否关闭
```

```
<-ch
    │
    ▼ 加锁
    │
    ├─ sendq 不为空（有人等着发送）
    │      ├─ 无缓冲 channel：直接 memcpy 发送方数据
    │      └─ 有缓冲 channel：取 buf[recvx]，发送方入队
    │      唤醒 sendq 头部 G
    │
    ├─ 缓冲区有数据
    │      从 buf[recvx] 读取
    │      recvx = (recvx+1) % dataqsiz
    │      qcount--
    │
    └─ 缓冲区为空
           goroutine 阻塞，加入 recvq 尾部
    │
    ▼ 解锁
```

### 4. 死锁（Deadlock）条件

```go
// 死锁场景 1：单 goroutine 向无缓冲 channel 发送后自己等待接收
go func() {
    ch := make(chan int)
    ch <- 1  // 阻塞在这里，因为没有人接收
}()

// 死锁场景 2：循环等待
// A: ch1 <- 1; <-ch2
// B: ch2 <- 1; <-ch1  ← 循环等待

// 死锁场景 3：channel 忘记启动接收
```

**Golang 如何检测死锁？**
runtime 在启动时启动 `sysmon`，如果所有 goroutine 都处于 waiting 状态且没有 sysmon 可运行，程序崩溃并报告死锁。

### 5. Select 多路复用

```go
select {
case v := <-ch1:
    fmt.Println("received", v)
case ch2 <- 10:
    fmt.Println("sent")
default:
    fmt.Println("both blocked")
}
```

**select 实现原理：**
1. 随机遍历所有 case（防止饥饿）
2. 找出已就绪的 channel（发送/接收不阻塞）
3. 如果都没有就绪且有 default → 执行 default
4. 如果都没有就绪且没有 default → **阻塞直到某个 case 就绪**
5. 执行对应的发送/接收，复制数据

```go
// select 源码简化逻辑（src/runtime/select.go）
func selectgo(cases []scase) (int, bool) {
    // 1. 随机乱序所有 case（公平性）
    // 2. 遍历找到第一个不阻塞的 case
    // 3.都没有就绪则 park 当前 goroutine
    //    将自己注册到每个 case channel 的等待队列
    // 4. 某个 channel 就绪时唤醒，执行对应 case
}
```

### 6. 生产问题与排查

**问题 1：goroutine 泄漏**

```go
// 场景：接收方在某个条件下不读了，但 channel 发送方还在等待
func process(ch <-chan Item) {
    for {
        select {
        case item := <-ch:
            if item.important {
                handle(item)
            }
            // important=false 的数据没人收 → 发送方泄漏
        case <-done:
            return
        }
    }
}

// 解决方案：确保每个发送都有对应接收，或使用带超时的 select
```

**问题 2：channel 关闭后仍可读取剩余数据**

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
close(ch)

// 关闭后仍可读取剩余数据：
for v := range ch {  // 读到 1, 2 后自动退出
    fmt.Println(v)
}

// 已关闭 channel 读取零值：
v, ok := <-ch  // ok = false
```

**问题 3：向已关闭 channel 发送 panic**

```go
close(ch)
ch <- 1  // panic: send on closed channel
```

### 7. 关键参数与调优

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `make(chan T)` | 无缓冲 | 同步通信，必须同时有收发两端 |
| `make(chan T, n)` | 缓冲大小 n | 异步通信，缓冲区满才阻塞 |
| 无缓冲 channel | - | 天然同步，交换数据保证happens-before |

```go
// 高性能场景：预估缓冲大小，避免频繁阻塞
ch := make(chan *Request, 1000)  // 预估 QPS * 尾延迟

// 背压机制（Back Pressure）
// 当 ch 满时，发送方知道需要减速，而不是无限入队
select {
case ch <- req:
    // 正常处理
default:
    metrics.Inc("channel_full")
    // 降级处理：拒绝/丢弃/入队到备份 channel
}
```

---

## 高频追问

**Q：channel 和共享内存比，优势在哪里？**

- **更安全**：不用手动管理锁，天然避免数据竞争（race condition）
- **更清晰**：通过CSP模型（"通过通信共享内存，而不是通过共享内存通信"），逻辑更直观
- **Go 调度器天然适配**：发送/接收阻塞时自动换出 goroutine，不需要 busy-wait

**Q：channel 发送和接收数据的成本是多少？**

- 有缓冲 channel（不阻塞）：一次 memcpy（数据大小）
- 无缓冲 channel（阻塞）：需要两次调度切换（G1 阻塞 → 唤醒 G2 → G2 执行），开销约 200-500ns
- 加上锁竞争成本，高并发下需要注意 channel 热点

**Q：nil channel 有什么特殊行为？**

```go
var ch chan int  // nil channel

<-ch   // 永远阻塞
ch <- 1 // 永远阻塞
close(ch) // panic
```

nil channel 在 select 中表现为永远阻塞的分支（跳过该 case）。

**Q：如何保证 channel 的线程安全？**

Channel 内部已经通过 `lock mutex` 保证了发送和接收的原子性。无需外部加锁。但注意：如果把 channel 本身作为共享变量，多个 goroutine 同时 `ch = make(chan int)` 会导致数据竞争。

---

## 延伸阅读

- [Go Channel 源码分析](https://github.com/golang/go/blob/master/src/runtime/chan.go)（官方 runtime 实现）
- [深入理解 Go Channel](https://draven.co.me/2020/03/22/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3-Go-Channel/)（ draven 大佬博客）
- [Go Concurrency Patterns: Channel](https://blog.golang.org/concurrency-timeouts)（官方 blog）
- [Go Data Structures: Channels](https://research.swtch.com/chan)（Russ Cox 经典文章）

---

**[← 上一篇：GMP 调度模型](../01-runtime/01-gmp.md)** · **[下一篇：sync 原语](./02-sync.md)**
