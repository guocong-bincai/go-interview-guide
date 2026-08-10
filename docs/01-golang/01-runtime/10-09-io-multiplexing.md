# Go 网络 I/O：epoll/kqueue 与 Netpoller 原理

> 面试频率：★★★★☆  优先级：P1  
> 关键词：epoll/kqueue、netpoller、non-blocking I/O、sysmon、网络 goroutine 阻塞与唤醒、I/O 多路复用对比

---

## 面试官考察意图

这道题考察候选人对 Go 网络模型的深层理解，属于区分度极高的追问。
初级候选人只知道"Go 使用 epoll"，但无法解释**为什么 goroutine 阻塞时不阻塞 M、M 怎么从 epoll 事件中感知到 goroutine 的 I/O 就绪、netpoller 在调度循环中的位置**。
高级候选人能结合源码讲清：Go 实际上是用 epoll 管理了大量 fd，但每个 goroutine 不直接对应一个 epoll 监听——而是通过 `netpoll` 统一批量获取就绪事件，再精准唤醒对应 goroutine。
如果还能对比 Go 的方案与传统的 libuv / Java NIO 的差异，说明候选人有系统级网络编程经验。

---

## 核心答案（30 秒版）

Go 的网络 I/O 核心是 **`netpoller`**（Go 1 之前的名字），现代 Go 已将其集成到调度器：

```
Goroutine 发起网络 I/O（connect/read/write）
  → 封装为 pollDesc，尝试非阻塞 I/O
  → 失败 → goroutine 进入 _Gwaiting，加入 netpoller 等待队列
  → sysmon 定期调用 netpoll() 批量获取就绪事件
  → 找到对应 goroutine，标记为 _Grunnable，重新进入调度
```

Linux 上 netpoller 基于 **epoll**（边缘触发），macOS/FreeBSD 基于 **kqueue**，Windows 基于 IOCP。
关键设计：**goroutine 阻塞不阻塞 M**，因为 goroutine 的等待状态存在 runtime 结构里，不在内核的 epoll fd 上。

---

## 深度展开

### 1. 为什么需要 Netpoller？—— 传统 I/O 模型的问题

传统的阻塞 I/O 模型：

```go
// 每个连接一个 goroutine，但 read() 是阻塞系统调用
conn, _ := net.Conn.Read() // M 在这里等数据，M 被卡住
```

问题：**每个阻塞的系统调用都占用一个 OS 线程**，而线程创建/切换成本极高。
如果 10,000 个连接各占一个线程，光线程调度开销就无法承受。

现代解决方案是 **I/O 多路复用**（select/poll/epoll/kqueue）：

```go
// 一个线程同时监控 N 个 fd，有事件就返回
rfds, _, _ := epoll_wait(epfd, events, 10000, timeout)
```

但直接用 epoll 需要：管理 epoll fd、注册 fd、回调函数goroutine 映射——非常繁琐。

Go 的设计哲学是：**让用户像写同步代码一样写网络 I/O**，底层自动处理多路复用。Netpoller 就是这座桥。

---

### 2. pollDesc：goroutine 与 fd 的中间层

每个被 netpoller 管理的 fd 都有一个 `pollDesc` 结构：

```go
// src/runtime/netpoll.go
type pollDesc struct {
    fd       uintptr
    ... 
    rg       uintptr   // 等待读事件的 goroutine pc（0=未注册）
    wg       uintptr   // 等待写事件的 goroutine pc
    link     *pollDesc // 链表：所有被 netpoller 管理的 pollDesc
}
```

关键设计：`rg` 和 `wg` 存的是 **goroutine 的 SP（栈指针）**，用于判断 goroutine 是否仍在等待。
这是 Go 1 的设计（通过 SP 判断），Go 1.22+ 改用结构体字段存 g 指针，更清晰。

goroutine 调用网络操作时的流程：

```go
// 伪代码，描述核心逻辑
func netRead(fd int, b []byte) (int, error) {
    // 1. 尝试非阻塞读取
    n, err := readSyscall(fd, b)
    if err == EAGAIN {
        // 2. 需要等待数据，注册到 netpoller
        pd := pollDescOf(fd)
        pd.rg = getg() // 记录等待的 goroutine
        
        // 3. goroutine 阻塞，调度器把这个 G 切走
        park(gp, "net.read")
        
        // 4. 被 netpoll 唤醒后从这里继续
        return pd.res.n, pd.res.err
    }
    return n, err
}
```

---

### 3. sysmon 与 netpoll 批量唤醒

sysmon 是 Go 运行时的"后台监控线程"（也叫 `m0` 或独立 M），**不绑定任何 P**，每隔 ~10μs 唤醒一次做几件事：

1. 从网络 I/O 超时的 goroutine 中恢复（netpoll）
2. 抢占运行时间过长的 G（调度延迟）
3. 释放自由的 M

netpoll 的核心逻辑在 `src/runtime/netpoll_epoll.go`（Linux）：

```go
// netpoll is called by the scheduler to find runnable goroutines.
// It returns a list of goroutines that are ready to run.
func netpoll(block bool) *g {
    var events [128]epollevent
    
    for {
        n, err := epollwait(epfd, &events[0], int32(len(events)), nextTimeout)
        if n == 0 && !block {
            return nil
        }
        // 遍历就绪事件
        for i := 0; i < int(n); i++ {
            pd := (*pollDesc)(events[i].Data)
            // 找到对应的 goroutine，标记为可运行
            if pd.rg != 0 {
                gp := wakeable(pd.rg) // 恢复 goroutine
                if gp != nil {
                    globrunqput(gp)
                }
            }
            if pd.wg != 0 {
                gp := wakeable(pd.wg)
                if gp != nil {
                    globrunqput(gp)
                }
            }
        }
        if !block {
            break
        }
    }
    return nil
}
```

**为什么批量处理？**  
`epoll_wait` 一次返回多个就绪 fd，Go 可以一次性唤醒多个 goroutine，充分利用多核并行度。这是 Go 比传统 callback 风格网络库的另一个优势——goroutine 调度完全由运行时管理，不破坏程序结构。

---

### 4. Linux epoll vs macOS kqueue

| 维度 | epoll（Linux） | kqueue（macOS/FreeBSD） |
|------|---------------|----------------------|
| 触发方式 | 边缘触发（ET）+ 水平触发（LT） | 只支持边缘触发（ET） |
| 注册方式 | `epoll_ctl` 添加/修改/删除 | `kevent` 统一接口 |
| 关注类型 | `EPOLLIN`/`EPOLLOUT`/`EPOLLET` | `EVFILT_READ`/`EVFILT_WRITE` |
| 实现文件 | `netpoll_epoll.go` | `netpoll_kqueue.go` |
| Go 内部 | 同一个 epoll 实例管理所有 fd | 每个 P 一个 kqueue fd |

Go 1.22 之后对 kqueue 做了优化，减少了每次 kevent 调用的开销。

---

### 5. 与传统模型的对比

| 模型 | 代表技术 | goroutine 等待时线程行为 | 优点 | 缺点 |
|------|---------|----------------------|------|------|
| 1:1 线程模型 | 传统 PHP/Perl | 线程阻塞 | 简单 | 线程成本高 |
| 多路复用同步 | epoll + 回调 | 单线程轮询 | 高性能 | 代码割裂 |
| **Go Netpoller** | epoll/kqueue + goroutine | **线程空闲可调度其他 G** | 高性能 + 结构清晰 | 实时性稍弱 |
| 异步回调 | libuv / Node.js | 事件循环线程 | 超高并发 | callback hell |

Go 的方案是**同步编程风格 + 非阻塞 I/O + 用户态调度**，融合了两者优点：

```go
// 用户代码：看起来完全同步
resp, err := http.Get("https://example.com")

// 实际执行：goroutine 在 netpoller 等待，线程 M 去调度其他 goroutine
```

---

### 6. 生产经验与常见面试坑

**Q：为什么有时 epoll_wait 返回了 fd 事件，但 goroutine 还是hang住了？**
A：可能是水平触发（LT）vs 边缘触发（ET）的问题。Go 的 netpoller 用的是水平触发（`EPOLLIN`），每次 `epoll_wait` 都会返回。hang 住通常是：
- 对端发送了部分数据，读取逻辑有 bug
- 对端 close 了连接但本地没有处理
- 混合了阻塞 I/O 系统调用（如 `os.Read` 混用）

**Q：netpoller 对延迟的影响？**
A：sysmon 轮询间隔 ~10μs，所以从 fd 就绪到 goroutine 被唤醒最多 ~10μs。这比线程切换（μs~ms 级别）快很多。对于大多数应用可以忽略不计，但 HFT（高频交易）这类场景会关注。

**Q：大量短连接场景下 netpoller 的表现？**
A：每次 connect/disconnect 都要操作 epoll，频繁 `epoll_ctl ADD/DEL` 会有开销。Go 1.22+ 对此做了优化，减少了无效的 ADD/DEL。但更关键的是：HTTP Keep-Alive 和连接池能显著减少连接数。

---

## 高频追问

**Q：goroutine 的网络 I/O 阻塞和 sync.Mutex 的阻塞有什么区别？**

> goroutine 阻塞时，调度器会把 M 上绑定的 P 释放出来调度其他 G——M 不阻塞。
> mutex 阻塞时，goroutine 进入 `sync.mutex` 的等待队列，`g.status = _Gwaiting`，同样不占用 M。
> 两者本质相同：goroutine 状态标记为 waiting，M 可以继续调度其他 G。
> 区别在于唤醒方式：mutex 是 `runtime.unlock` 直接唤醒；网络 I/O 是 `netpoll` 批量唤醒。

**Q：sysmon 抢占和 netpoller 是什么关系？**

> sysmon 是总后台监控，每轮做三件事：netpoll（网络就绪检查）、抢占检查、释放 M。
> netpoller 是网络 I/O 特定的事件循环。
> 关系：sysmon 调用 netpoller 的 `netpoll()` 函数来检查哪些 goroutine 可以被唤醒。

**Q：Go 的方案和 Java NIO 有什么区别？**

> Java NIO 用 `Selector` 管理 `SelectionKey`，注册 channel 到 selector，然后单线程（或少量线程）轮询就绪事件。
> Go 的 netpoller 是运行时内置的，不需要用户手动管理 selector。
> Java 的瓶颈在于 selector 轮询本身（单线程），Go 通过 GMP 调度和批量唤醒让这件事更自然。

---

## 延伸阅读

- [Go 源码：netpoll_epoll.go](https://github.com/golang/go/blob/master/src/runtime/netpoll_epoll.go)
- [Go 源码：netpoll_kqueue.go](https://github.com/golang/go/blob/master/src/runtime/netpoll_kqueue.go)
- [Go 源码：proc.go 中的 schedule/findRunnable](https://github.com/golang/go/blob/master/src/runtime/proc.go)
- [Go 官方博客：The Go netpoller](https://blog.golang.org/netpoller)
- [Go 调度器源码走读](../01-runtime/05-scheduler-source-code.md)
