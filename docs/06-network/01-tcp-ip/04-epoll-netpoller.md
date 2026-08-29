# epoll 模型与 Go netpoller

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：epoll、IO多路复用、goroutine调度、C10K问题、Reactor模式、netpoller

---

## 🎯 面试官考察意图

- 是否理解 Linux 高并发网络编程的核心机制（epoll）
- 能否解释 Go 如何用同步 API 实现异步 IO（同步的网络编程接口 + epoll 底层）
- 是否了解 select/poll/epoll 的演进和性能差异
- 能否结合 goroutine 说明 Go 如何实现 C10K/C10M 级并发连接

---

## ⚡ 核心答案（30秒）

> **epoll** 是 Linux 处理大量并发网络连接的核心机制，通过事件通知而非轮询来减少 CPU 开销。与 select（O(n)、最大 FD 限制）和 poll（O(n)）相比，epoll 支持百万级文件描述符且只返回就绪 FD，时间复杂度为 O(1)。
>
> Go 的 **netpoller** 在底层封装了 epoll（Linux）、kqueue（macOS/BSD）和 eventport（Solaris）。Go 的 net/http 用同步 API 编写，但底层通过 netpoller 将 goroutine 挂起等待 IO 事件，FD 就绪时唤醒对应 goroutine——这就是"Go 提供了同步的网络编程接口"背后的秘密。

---

## 🔬 深度展开

### 1. I/O 多路复用：从 select 到 epoll

#### 三种模型的对比

```go
// select（最老，1983年引入）
int nfds = select(maxfd+1, &readfds, &writefds, &errorfds, &timeout);
// ① O(n) 每次调用遍历所有 FD
// ② FD_SETSIZE 限制（通常 1024）
// ③ 每次调用后需要重新设置 readfds/writefds（内核无法缓存）

// poll（1995年引入，消除了 FD 数量限制）
struct pollfd events[1024];
int ret = poll(events, nfds, timeout);
// ① O(n)，没有 FD 数量硬性限制
// ② 仍需每次传入完整数组，内核复制开销大

// epoll（2.5.44 内核，2002年）
int epfd = epoll_create(1024);       // 创建 epoll 实例
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);  // 注册 FD
int ready = epoll_wait(epfd, events, maxevents, timeout);
// ① O(1) — 只返回就绪 FD，不需要遍历全部
// ② 红黑树管理注册表，增删改高效
// ③ 就绪列表用链表组织，epoll_wait 只需遍历就绪链表
// ④ 无 FD 数量限制（只受系统内存限制）
```

#### epoll 两种工作模式

| 模式 | 特性 | 适用场景 |
|------|------|---------|
| **LT（Level Triggered）** | 默认模式，只要 FD 就绪就持续通知 | 大多数应用，容错性好 |
| **ET（Edge Triggered）** | 只在状态变化时通知一次，需一次性读完数据 | 高性能场景，如 Nginx |

```c
// ET 模式下必须用非阻塞 IO + 循环读直到 EAGAIN
while (1) {
    n = read(fd, buf, sizeof(buf));
    if (n < 0) {
        if (errno == EAGAIN) break;  // 已读完，退出
        else handle_error();
    }
    process_data(buf, n);
}
```

#### epoll 数据结构（内核实相）

```
epoll 实例的结构：

┌─────────────────────────────────┐
│ epoll instance                   │
│ ┌─────────────────────────────┐ │
│ │ red-black tree               │ │  所有注册的 fd（按 fd 排序）
│ │ fd → epoll_item              │ │  O(log n) 查找
│ └─────────────────────────────┘ │
│                                 │
│ ┌─────────────────────────────┐ │
│ │ ready list                   │ │  就绪 fd 的链表
│ │ fd_ready_1                   │ │  O(1) 遍历，epoll_wait 只读此表
│ │ fd_ready_2                   │ │
│ └─────────────────────────────┘ │
│                                 │
│ ┌─────────────────────────────┐ │
│ │ wait queue                   │ │  等待 io 事件的队列
│ │ - goroutine_1 (on fd_A)      │ │  当 fd 可读时，唤醒对应的等待者
│ │ - goroutine_2 (on fd_B)      │ │
│ └─────────────────────────────┘ │
└─────────────────────────────────┘
```

---

### 2. Go netpoller 工作原理

#### Go 网络模型概览

```
应用层（sync）                runtime
┌──────────┐          ┌────────────────────┐
│ net.Conn │──Read()→  │ netpoller          │
│          │←Write()──│                    │
│ goroutine│◄─────────│ epoll_wait         │
│ 被挂起   │          │ 将就绪 FD 映射回   │
│          │          │ goroutine 并唤醒   │
└──────────┘          └────────────────────┘
                          │
                          ▼ epoll_ctl/kqueue
                         OS 内核
```

关键：Go 开发者用**同步 API**（`conn.Read()` 会阻塞），但运行时自动将阻塞转为**异步 IO + goroutine 挂起**。

#### netpoller 生命周期

```go
// Go 源码中的 netpoller 启动过程

func main() {
    // 1. Netpoll init — 创建 epoll 实例
    sysPollDesc = netpollGenericInit()
    
    // 2. netpollBreakInit — 创建管道用于 Wakeup
    netpollBreakInit()
    
    // 3. netpollWake — 创建后台线程周期性执行
    go netpollWakeupTrigger()  // 定时检查是否有超时
    
    // 4. scheduler 中的 netpoll
    //    当所有 goroutine 都阻塞时，scheduler 调用 netpoll()
}

// 当 goroutine 发生阻塞 IO 时：
func netpollBlock(blocking bool, waitio bool) {
    pd := getNetPollDesc()        // 获取该 fd 的 pollDesc
    pd.set(eventReadable, nil)    // 设置关注事件
    netpollarm(pd, blocking, waitio)  // 注册到 epoll
}

// 当 scheduler 发现所有 g 都在等待时：
func schedule() {
    if !goready(gp, -1) {
        gp.waiting = nil          // 清除等待链
        readyWithTrace(gp)        // 唤醒 goroutine
    }
}
```

#### goroutine 挂起与唤醒的完整流程

```go
// 以 conn.Read() 为例：
conn, _ := net.Dial("tcp", "server:8080")
buf := make([]byte, 1024)
n, err := conn.Read(buf)
// 这个 Read() 做了以下事情：

// Step 1: syscall 尝试读取
bytes, err = syscall.Read(sysFd, buf)  // 可能阻塞

// Step 2: 如果阻塞（EAGAIN/EWOULDBLOCK），挂起 goroutine
if err == syscall.EAGAIN || err == syscall.EWOULDBLOCK {
    netpollblock(netpollop, pd, mode, true)  // 把当前 g 放入 wait queue
    gopark(nil, nil, waitReasonForceGCHandler, traceEvGoBlock, 1)  // 挂起 g
}

// Step 3: 服务器发数据来了 → socket 可读 → epoll_wait 检测到
// Step 4: runtime/netpoll.go → netpoll() → 遍历就绪 fd
// Step 5: 将 fd 对应的事件映射到 goroutine，调用 readywith() 唤醒 g
// Step 6: goroutine 继续执行 Read()，这次直接从缓冲区读取成功
```

#### netpoll 的时间触发

除了 IO 事件，netpoll 还要处理定时器：

```go
// Go 中每个 goroutine 如果有 Timer，也会在 netpoll 中监控
// 这样 sleep/wait/group 等定时器都能精确触发

func netpoll(block bool) []g {
    var tg *gList
    // 先检查超时（timer 到期）
    t := timespec{...}
    if block {
        t.tv_sec = -1  // 无限等待，直到有事件或超时
    }
    
    // epoll_wait 同时监控 IO 事件和 timer 到期
    // 就绪的 fd 和超时的 timer 都会导致对应的 goroutine 被唤醒
}
```

---

### 3. Go 如何支持海量并发连接

#### C10K 问题的 Go 解决方案

```
传统方案（每进程/每线程一个连接）：
CPU 核数有限 → 线程切换成本高 → 最多几千并发连接
线程栈默认 2MB → 10万连接 = 200GB 内存！不可行

Go 的方案：
1. goroutine：初始栈仅 2KB，按需增长，内存占用极低
2. netpoller：epoll 驱动的事件循环，单线程监控百万级 FD
3. M:N 调度：多个 OS 线程复用海量 goroutine
4. 异步读写：不阻塞 OS 线程，充分利用多核

结果：单机轻松处理 10万~100万 并发连接
```

#### Go net/http 并发模型

```go
// 标准库 http.Server 的连接处理：

func (srv *Server) Serve(l net.Listener) error {
    for {
        // accept 是阻塞的，但在 netpoller 管理下不会阻塞 goroutine
        rw, err := l.Accept()
        if err != nil {
            return err
        }
        
        // 每个连接一个 goroutine（轻量！仅几 KB）
        c := srv.newConn(rw)
        go c.serve(ctx)
    }
}
```

```
┌─────────────────────────────────────────────┐
│  Goroutine Pool (调度器管理的 P/M/G)          │
│                                             │
│  G1: accept 连接 #1  ✓ 处理中               │
│  G2: accept 连接 #2  ✓ 已处理完             │
│  G3: read request  ✓ 正在读                 │
│  ...                                        │
│  G10000: idle goroutine                     │
└─────────────────────────────────────────────┘
        ↕ netpoller (epoll_wait)
┌─────────────────────────────────────────────┐
│  File Descriptors (百万级)                   │
│  fd1: conn #1  fd2: conn #2  ... fd100000   │
└─────────────────────────────────────────────┘
```

---

### 4. 常见面试追问

**Q：epoll 为什么比 select 快？**

> select 每次调用都要把完整的 FD 集合从用户态复制到内核态，内核遍历所有 FD 检查就绪状态（O(n)），调用结束后用户代码再遍历找出哪些就绪。epoll 只需在 epoll_ctl 时注册一次（O(log n)），epoll_wait 只返回就绪的 FD（O(1)），无需遍历全部。

**Q：ET 模式和 LT 模式有什么区别？**

> LT 模式下只要 FD 可读写就会反复通知，即使上次没读完；ET 模式只在状态改变（从不就绪到就绪）时通知一次，必须用非阻塞 IO 一次性读完。ET 性能更高但容易漏事件，LT 更安全但可能产生重复通知。

**Q：Go 的 netpoller 是什么时刻触发的？**

> ① 所有用户 goroutine 进入 waiting 状态时，runtime 调用 netpoll() 阻塞等待 IO 事件；② 每个 P（处理器）每秒 ~1 次主动调用 netpoll 检查超时事件；③ netpollBreak 可用于立即唤醒 netpoll（例如 GC 标记阶段）。

**Q：Go 标准库不支持 Windows epoll？用什么？**

> Go 使用平台抽象层：Linux 用 epoll，macOS/BSD 用 kqueue，Windows 用 IOCP（Completion Port），Solaris 用 event port。统一表现为 Go 代码中的 `net.Conn`。

**Q：什么是 IO_uring？Go 会用吗？**

> IO_uring 是 Linux 5.1 引入的新一代异步 IO API，比 epoll 更高效——它通过共享内存环形缓冲区减少系统调用次数。目前 Go 尚未原生支持 IO_uring，社区有讨论但尚未合入。

---

### 5. Go 实战：自定义 netpoll 示例

```go
package main

import (
    "encoding/binary"
    "fmt"
    "log"
    "net"
)

// 简单的高并发 echo 服务器
// 展示 Go net/http 背后 netpoller 的工作方式

type EchoServer struct {
    listeners map[string]net.Listener
}

func NewEchoServer() *EchoServer {
    return &EchoServer{
        listeners: make(map[string]net.Listener),
    }
}

func (s *EchoServer) Listen(addr string) error {
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    s.listeners[addr] = ln
    log.Printf("Listening on %s", addr)
    return nil
}

func (s *EchoServer) Start() error {
    // 阻塞等待信号量来停止服务器
    select {}
}

func (s *EchoServer) handleConn(conn net.Conn) {
    defer conn.Close()
    
    // Buffer pool 回收对象
    const BufSize = 4096
    buf := make([]byte, BufSize)
    
    for {
        // 这段代码表面是同步阻塞的 Read
        // 实际上底层是通过 netpoller 异步完成的
        // goroutine 在此处会被挂起，直到 socket 可读
        
        n, err := conn.Read(buf)
        if err != nil {
            return
        }
        
        _, err = conn.Write(buf[:n])
        if err != nil {
            return
        }
    }
}
```

---

## ❓ 高频追问

**Q：select/poll/epoll 各自适合什么场景？**

> select：连接数少（<1024），兼容性好（所有 Unix-like）；poll：中等规模连接（千级别），不想受制于 FD_SETSIZE；epoll：大规模并发（万~百万级），现代 Linux 生产环境首选。Go 底层自动选择最佳方案。

**Q：Go 的 `bufio.Scanner` 和手动 `conn.Read()` 在处理半包上有区别吗？**

> `bufio.Scanner` 内置了缓冲区和半包处理逻辑，底层仍通过 netpoller 异步读取。但需要注意 Scanner 的默认最大 token 大小（64KB），超过则报错。对于二进制协议推荐使用 `bufio.Reader` 或自定义缓冲区。

**Q：如何排查 Go 服务中 goroutine 泄漏导致的连接堆积？**

> 用 pprof：`http.DefaultServeMux.Handle("/debug/pprof/goroutine", pprof.Handler("goroutine"))`。如果发现某个连接对应的 goroutine 永远不被唤醒，通常是 netpoller 层面出了问题（比如关闭了 fd 但没有通知 netpoll 移除它）。

---

## 📋 总结 checklist

- [ ] 能说清 select/poll/epoll 的性能差异（O(n) vs O(1)）
- [ ] 理解 epoll 的红黑树 + 就绪链表设计
- [ ] 知道 LT 和 ET 模式的区别和使用场景
- [ ] 理解 Go netpoller 如何将同步 API 变为异步 IO
- [ ] 能说清 goroutine 阻塞时如何被 netpoller 管理和唤醒
- [ ] 理解 Go 为何能支持海量并发连接（轻量 goroutine + epoll）

---

## 📚 延伸阅读

- [Go Runtime Source: runtime/netpoll.go](https://github.com/golang/go/blob/master/src/runtime/netpoll.go)
- [epoll man page - man 7 epoll](https://man7.org/linux/man-pages/man7/epoll.7.html)
- [What every programmer should know about memory - Ulrich Drepper](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
- [Linux epoll: A deep dive](https://blog.packagecloud.io/eng/2016/05/16/tuning-linux-tcp-ip-stack-for-high-throughput/)
