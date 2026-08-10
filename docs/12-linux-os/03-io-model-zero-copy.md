[🏠 首页](../../../README.md) · [📦 Linux / 操作系统基础](../README.md)

---

# I/O 模型与零拷贝

> 考察频率：★★★★☆  优先级：P0
> 关键词：select/poll/epoll/server client reactor、同步/异步/非阻塞、零拷贝、DMA、sendfile

---

## 1. 五种 I/O 模型

### 1.1 阻塞 I/O（BIO）

最传统、最简单的模型：

```
用户进程                           内核
    │
    │──── read(fd) ─────────────────→│  等待数据到达网卡/磁盘
    │                                │
    │←──── 进程阻塞（睡眠）──────────│  数据从外设复制到内核缓冲区
    │                                │
    │←──── 数据返回（解除阻塞）──────│  数据从内核缓冲区复制到用户空间
    │──── 返回 read ────────────────→│
    │
```

**特点：**
- 每连接一个线程：1 万连接 = 1 万线程 → 内存爆炸
- CPU 大部分时间阻塞在内核等待 I/O，利用率极低

### 1.2 非阻塞 I/O（NIO）

轮询方式，不阻塞线程：

```
用户进程                           内核
    │
    │──── read(fd) ─────────────────→│  EAGAIN（立即返回，数据未就绪）
    │←──── 立即返回 ─────────────────│
    │
    │（忙等待或短轮询）               │
    │
    │──── read(fd) ─────────────────→│  返回数据
    │←──── 返回 ─────────────────────│
```

**问题：CPU 空转（短轮询），高并发时仍浪费 CPU**

### 1.3 I/O 多路复用（select / poll / epoll）

**核心思想：一个线程管理多个 FD，通过事件驱动而非忙轮询**

```
select/epoll 流程：
    │
    ├── epoll_create() → 创建 epoll 实例（红黑树 + 双向链表）
    │
    ├── epoll_ctl(ADD) → 将 FD 注册到 epoll 实例
    │
    └── epoll_wait() → 阻塞，直到有 FD 变为可读/可写
                           │
                           ← 返回就绪 FD 列表（每次只处理活跃连接）
```

**select vs poll vs epoll 对比：**

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| FD 数量限制 | **FD_SETSIZE=1024** | 无限制（数组）| 无限制 |
|FD 集合传递| 每次 fd_set 拷贝进内核| 每次 pollfd 数组拷贝| **共享内存，红黑树 O(log n) 管理**|
|时间复杂度| O(n) 遍历所有 FD|O(n) 遍历所有 FD|**O(1) 返回就绪 FD 数量**|
|FD 重复触发| 不能|PollEvents 不能|FD 可设为边缘触发（ET）|
|水平触发 vs 边缘触发| 水平触发（LT）|水平触发（LT）|**LT + ET 两种模式**|

**epoll 工作模式：**

```
水平触发（LT）：数据未读完，持续通知
  read → 返回部分数据 → 再次 epoll_wait → 再次返回

边缘触发（ET）：只通知一次，必须一次性读完
  read → 返回部分数据 → 不再通知，直到下次新数据到达
  → 必须用非阻塞 read + 循环读 + while(err == EAGAIN)
```

**Go 的 netpoller 使用 LT 模式**：Go runtime 内部用 epoll，在 Go 中 channel/conn.Read 是阻塞调用，无需关心 ET/LT。

### 1.4 信号驱动 I/O（SIGIO）

很少使用，流程：内核 → FD 就绪 → 发送 SIGIO → 用户处理

### 1.5 异步 I/O（aio / io_uring）

```
用户发起 io_submit()（立即返回，不等待）
       │
       ▼
 内核处理 I/O，数据到达后写入用户缓冲区
       │
       ▼
  内核发送完成事件（IOCB_COMPLETE）
       │
       ▼
 用户通过 io_getevents() 获取结果
```

**io_uring（Linux 5.1+）**：零拷贝 + 轮询模式，性能极高，是下一代高性能 I/O 框架。
**Go 的 netpoller 目前未使用 io_uring**（Go 1.24 有初步支持讨论）。

---

## 2. Reactor 模型：高性能服务器的秘密

### 2.1 单 Reactor 单线程

```
┌─────────────────────────────────────┐
│              Reactor                │
│  epoll_wait                         │
│  ┌──────────────┐                   │
│  │ Acceptor     │ ← 监听 socket    │
│  │ Handler      │ ← 处理已连接 FD  │
│  └──────────────┘                   │
└─────────────────────────────────────┘
```

Redis / Go net/http（单 goroutine）使用此模式。

### 2.2 单 Reactor 多线程（Netty 经典模式）

```
┌────────────────────────────────────────────┐
│  MainReactor（1线程）→ 监听 accept        │
│       │                                    │
│       ▼ 分发到 SubReactor（多个 NIO 线程） │
│  SubReactor 线程池（N 个）                 │
│  epoll_wait → 处理读写事件                 │
└────────────────────────────────────────────┘
```

### 2.3 Go 的 net.Listener 实现

```go
// Go 的 net.Listener.Accept 是怎么工作的？
// 底层：runtime.netpoll → epoll_ctl ADD → epoll wait

// 每个 TCP 连接由一个 goroutine 处理（轻量）
ln, _ := net.Listen("tcp", ":8080")
for {
    conn, _ := ln.Accept()        // goroutine 阻塞，不占线程
    go handleConn(conn)           // 每个连接一个 goroutine
}

// 连接数 10万 → 10万 goroutine
// 但不是 10万 OS 线程（M = GOMAXPROCS = 8）
// goroutine 让出 P 时，Epoll wait 负责唤醒
```

---

## 3. 零拷贝：减少数据复制次数

### 3.1 传统 I/O 的 4 次拷贝

```
磁盘数据 → 内核缓冲区（Page Cache）→ 用户缓冲区 → Socket 缓冲区 → 网卡
         DMA 拷贝①         ②           ③           ④ CPU拷贝
```

```bash
# 传统 read + write 流程（4次拷贝，2次系统调用）：
fd = open("file")
socket = socket()
read(fd, buf)        # 磁盘→内核buf（DMA），内核buf→用户buf（CPU拷贝）
write(socket, buf)   # 用户buf→Socket buf（CPU拷贝），网卡发送
```

### 3.2 sendfile：零拷贝的第一次飞跃

Linux 2.1 引入 `sendfile()` 系统调用：

```c
// 内核 2.1+
#include <sys/sendfile.h>
sendfile(out_fd, in_fd, &offset, count);
// 数据：磁盘 → Page Cache → 网卡（全程在内核态，0次用户态拷贝）
```

```
sendfile 流程（3次拷贝，1次系统调用）：
磁盘 → Page Cache（DMA①）→ Socket 缓冲区（DMA②）→ 网卡
                              内核内部直接传递，无用户态介入
```

**splice / tee / vmsplice**：Linux 2.6 引入，通过管道中转实现内核内部零拷贝。

### 3.3 DMA 辅助拷贝：关键硬件支持

```
没有 DMA 时：CPU 必须参与每次 I/O 拷贝（CPU bottleneck）
有 DMA 时：DMA 控制器（独立硬件）负责 I/O 拷贝
           CPU 只负责设置 DMA 描述符，开销极低

零拷贝 = CPU 0 参与的拷贝（依赖 DMA + 内核内部转发）
```

### 3.4 Go 中的零拷贝

**① net.FileConnection（Linux）使用 sendfile：**
```go
// net.TCPConn.File() 返回 os.File（利用 Linux sendfile）
conn, _ := listener.Accept()
file, _ := conn.(*net.TCPConn).File()
n, _ := file.ReadFrom(os.Stdout) // 内部调用 sendfile(2)
```

**② Go 的 HTTP/2 和 gRPC 使用 splice 实现 Linux 内部的零拷贝数据传输**

**③ io.Copy 的优化路径：**
```
io.Copy(dst, src)
    ↓
判断 dst 是否是 *net.TCPConn → 使用 syscall.Sendfile（Linux）
判断 src 是否是 *os.File → 同上
否则 → 传统 read/write
```

---

## 4. Go I/O 模型与 netpoller

### 4.1 Go 网络 I/O 全流程

```go
conn.Read([]byte{})
    │
    ▼
netfd.Read()         ← 包装 syscall.Read
    │
    ▼
runtime.pollDesc.runtime_pollRead()
    │
    ├─ 快速路径：FD 可读 → 直接返回（epoll 返回 ready）
    │
    └─ 慢路径：FD 不可读
           │  park goroutine（放入等待队列）
           │  epoll_ctl(DEL)  // 取消监听
           │  让出 P 给其他 G
           │
           ▼  某时刻网络数据到达，epoll 通知
           │  goready(G) → 把 G 放回 P 的 runqueue
           │  epoll_ctl(ADD) // 重新监听
           ▼
        goroutine 被调度执行，FD 此时可读，Read 成功
```

### 4.2 goroutine 调度与 I/O 整合

```
P0 正在运行 goroutine A（阻塞在 channel recv）
       │
       ▼
 channel recv 内部：gopark() → G 休眠，P 空闲
       │
       ▼
 netpoller（独立 goroutine）唤醒后：goready(G) → G 放回 P0.runnext
       │
       ▼
 P0 再次调度到 G → G 继续执行
```

**关键结论：goroutine 阻塞不阻塞 M，M 可以继续执行其他 G。**
这就是 Go 高并发的秘密：**网络 I/O 阻塞 → goroutine park → P 绑定其他 G 运行**。

---

## 5. 高频追问

### Q1：epoll 的边缘触发（ET）为什么比水平触发（LT）性能高？

LT 模式下，epoll_wait 每次都返回同一 FD，直到数据被读完：
- 一次事件可能被多次通知 → 处理函数可能被多次调用
- 容易漏处理或重复处理

ET 模式下，只通知一次：
- 必须一次性把数据读完（用 while 循环）
- 减少无效系统调用
- 但编程复杂度高（必须非阻塞 read + 循环读）

Go 的 netpoller 使用 LT，在 Go 侧 goroutine 自然就是"读完才返回"，不需要手动循环。

### Q2：select 有 FD 数量限制（1024），怎么解决？

现代生产环境应使用 epoll（Linux）：
```go
// Go 标准库自动使用 epoll（无论 Linux/FreeBSD/macOS）
conn, err := net.Dial("tcp", "localhost:8080")
// 底层自动选择 kqueue（macOS/FreeBSD）或 epoll（Linux）
```

### Q3：Go 的连接数上限受什么限制？

| 限制 | 来源 | 解决方法 |
|------|------|---------|
| FD 上限 | `ulimit -n`（默认 1024）| `ulimit -n 65535` |
| 内存 | 每连接 goroutine 栈 2KB × N | Go 无需担心（goroutine 很轻）|
| GOMAXPROCS | 并行 goroutine 上限 | 设为 CPU 核数 |
| 端口号 | 客户端短连接耗尽端口（65535）| 使用连接池或长连接 |

---

## 6. io_uring：为什么它被称为"革命性"的 I/O 接口（2026 高频）

> 考察频率：★★★☆☆（2026 年热度上升）  关键词：SQ/CQ 环形队列、免系统调用、libaio vs io_uring、Go 生态现状

### 6.1 面试官考察意图

面试官看到简历上写过「高性能存储 / 消息中间件 / 网络框架」时，极大概率追问 io_uring。初级只会背「io_uring 快」，高级要讲清楚：**为什么快（环形队列免 syscall）**、**和 epoll 解决的不是同一个问题**、**为什么 Go 官方 netpoller 至今没用它**。

### 6.2 核心答案（30 秒版）

| 维度 | epoll | io_uring |
|------|-------|----------|
| 解决的问题 | **就绪通知**（告诉我哪个 fd 可读可写）| **整条 I/O 链路异步化**（提交+完成都免阻塞）|
| 系统调用 | 每次读写仍需 read/write syscall | `io_uring_enter` 批量提交，读写由内核异步执行 |
| 用户态/内核态交互 | 通过回调/就绪队列 | **共享内存环形队列（SQ/CQ）**，mmap 映射，无数据拷贝 |
| 数据拷贝 | 传统拷贝 | 支持固定缓冲区（registered buffers）零拷贝 |
| 适用 | 高并发网络连接管理（Go netpoller 正是如此）| 磁盘 I/O、存储引擎、数据库、网络收发一体 |

**一句话记忆：epoll 解决「等」的问题，io_uring 解决「做」的问题。**

### 6.3 原理：两个环形队列 + 三段系统调用

```
用户态                                       内核态
┌─────────────────────┐         ┌──────────────────────┐
│  SQ（Submission）    │  mmap   │  内核线程/IRQ 处理     │
│  提交队列（环形）     │◄───────►│  从 SQ 取请求执行      │
│  ┌──┬──┬──┬──┐      │  共享    │                      │
│  │S1│S2│S3│  │      │  内存    │  执行 read/write/     │
│  └──┴──┴──┴──┘      │         │  open/fsync/...       │
│                     │         │                      │
│  CQ（Completion）    │◄───────►│  完成后写回结果        │
│  完成队列（环形）     │         │                      │
│  ┌──┬──┬──┬──┐      │         │                      │
│  │C1│C2│  │  │      │         │                      │
│  └──┴──┴──┴──┘      │         └──────────────────────┘
└─────────────────────┘
```

**三段接口：**

```c
// 1. io_uring_setup：创建 ring，返回 fd，内核把 SQ/CQ 映射到用户态地址空间
struct io_uring_params params;
int ring_fd = io_uring_setup(256, &params);   // 队列深度 256

// 2. 用户态直接往 SQ 写 SQE（Submission Queue Entry），无需每次 syscall
//    一次 io_uring_enter 可以批量提交多个请求，减少上下文切换
io_uring_enter(ring_fd, to_submit=128, min_complete=1, 0);

// 3. 完成后内核把结果写入 CQ，用户态轮询 CQ 即可（甚至可配合 IOPOLL 免中断）
```

**性能关键点：**
1. **免 syscall 风暴**：万级 QPS 的传统 read/write = 每秒数万次系统调用；io_uring 一次 enter 批量提交，系统调用次数降低 2~3 个数量级
2. **真正异步**：libaio 对 buffered I/O 经常退化为同步阻塞；io_uring 从设计上就是异步的
3. **固定缓冲区**：`IORING_REGISTER_BUFFERS` 预注册缓冲区，内核直接操作，省去页表遍历与引用计数
4. **IOPOLL**：NVMe 场景下内核轮询硬件完成队列，延迟可到微秒级

### 6.4 高频追问

**Q1：epoll + 线程池 和 io_uring 差在哪？**

epoll 只告诉你「fd 就绪了」，真正的 read/write 还是要应用层发起系统调用并阻塞在读写上（除非用非阻塞 + 用户态拷贝）。io_uring 把「提交→执行→完成」整条链路由内核异步处理，应用层只碰两个环形队列。所以 io_uring 尤其适合**磁盘 I/O 密集**场景（存储、数据库、日志），而网络场景 epoll 已经够好。

**Q2：为什么 Go 官方 netpoller 不用 io_uring？**

Go 的 goroutine 模型天然便宜，一个 goroutine 阻塞在一个 epoll 事件上几乎零成本，epoll 已经能让 Go 支撑百万连接。io_uring 的收益主要在高频磁盘 I/O 和极致延迟场景；Go 官方在 net 层引入 io_uring 的收益有限，复杂度却很高（ring 的并发访问、内存模型），目前（Go 1.26）netpoller 仍基于 epoll。

**Q3：Go 生态里哪里用到了 io_uring？**

- **go-nitro**（字节跳动开源）：Go 的 io_uring 库，主打低延迟网络
- **uring-go**：更完整的 Go io_uring 绑定
- 存储/数据库类 C 项目直接用 liburing；Kafka 等 JVM 项目通过 JNI 使用
- 如果只是写业务 API，**不要为了炫技引入 io_uring**——epoll + goroutine 在绝大多数场景足够

### 6.5 面试话术（30 秒说完）

> 「io_uring 的核心是把提交队列和完成队列通过 mmap 共享给内核，应用层批量提交请求后可以继续做别的，内核异步执行完写回完成队列。相比 epoll 只解决就绪通知、读写还得自己调 syscall，io_uring 把整条 I/O 链路异步化了，特别适合磁盘 I/O 密集场景。Go 官方 netpoller 没用它，是因为 goroutine + epoll 已经能把网络并发做得很好，io_uring 的收益在存储引擎这类场景更明显。」

---

## 延伸阅读

- [The Secret of epoll Performance](https://idea.popcount.org/2017-02-20-epoll-the-api-is-os-agnostic-but-the-implementation-is-not/)
- [io_uring vs epoll](https://www.scylladb.com/2021/01/28/how-io_uring-will-change-how-we-write-software/)
- [io_uring(7) man page](https://man7.org/linux/man-pages/man7/io_uring.7.html)
- [Go netpoller 源码分析](https://github.com/golang/go/blob/master/src/runtime/netpoll_epoll.go)
