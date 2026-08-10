[🏠 首页](../../../README.md) · [📦 Linux / 操作系统基础](../README.md)

---

# 进程、线程、协程与调度

> 考察频率：★★★★★  优先级：P0
> 关键词：上下文切换、用户态/内核态、goroutine、轻量级线程、调度实体

---

## 1. 核心概念：进程 vs 线程 vs 协程

### 1.1 进程：资源分配的最小单位

进程是 **操作系统对运行程序的抽象**，拥有独立的：
- **地址空间**（虚拟内存布局：代码段、数据段、堆、栈）
- **文件描述符表**（FD 0~1024 + 继承）
- **信号处理方式**
- ** PCB（Process Control Block）**：PID、状态、寄存器、调度信息

**进程创建代价（以 Linux 为例）：**

```
fork() 系统调用：
  1. 复制 PCB（轻量，但仍需分配内核栈 ~8KB）
  2. 复制父进程地址空间（Copy-on-Write，写时复制）
  3. 复制文件描述符表（指向内核的 struct files_struct）
  4. 分配 PID、设置父 PID

总耗时：约 100~300μs（取决于地址空间大小）
```

### 1.2 线程：CPU 调度的最小单位

线程是**进程内的执行单元**，共享进程的：
- 地址空间（代码段、数据段、堆）
- 文件描述符
- 信号处理

**线程创建代价（Linux/NPTL）：**

```
pthread_create() 系统调用：
  1. 分配线程栈（默认 8MB，可缩小到 512KB~2MB）
  2. 分配 TLS（Thread Local Storage）
  3. 复制共享文件描述符表（引用计数 +1）
  4. 创建 LWP（Light Weight Process，内核线程）

总耗时：约 50~150μs（比进程快，但仍需内核介入）
```

### 1.3 协程（goroutine）：用户态调度

goroutine 是 Go 运行时管理的**用户态协程**，完全在用户空间调度：

| 属性 | 进程 | 线程 | goroutine |
|------|------|------|------------|
| 创建耗时 | 100~300μs | 50~150μs | **~1μs** |
| 初始栈大小 | 固定 8MB+ | 固定 8MB | **2~8KB（动态增长）** |
| 切换模式 | 用户→内核→用户 | 用户→内核→用户 | **纯用户态** |
| 切换开销 | ~3μs | ~1~2μs | **~200ns** |
| 调度者 | OS 内核 | OS 内核 | **Go Runtime** |
| 独立性 | 完全隔离 | 共享地址空间 | 共享地址空间 |

**goroutine 初始栈只有 2KB**，最大可增长到 1GB（Go 1.20+）。相比线程 8MB 的固定栈，goroutine 节省了 4000 倍的初始内存。

---

## 2. 上下文切换：核心切换成本

### 2.1 上下文切换的层次

```
┌─────────────────────────────────────────────────────────┐
│                    OS Kernel                              │
│                                                          │
│  ┌──────────┐    ┌──────────┐                           │
│  │ 进程 A   │ →  │ 进程 B   │   ← 进程上下文切换        │
│  │ (内核栈) │    │ (内核栈) │     内核级 (~3μs)         │
│  │ (PCB)   │    │ (PCB)   │                           │
│  └──────────┘    └──────────┘                           │
│                                                          │
│  线程 A ↔ 线程 B   ← 线程上下文切换                     │
│  同进程线程共享地址空间，只需切换少量寄存器               │
│                                                          │
└─────────────────────────────────────────────────────────┘
        ▲
        │ 用户态
┌───────────────┐
│   用户空间     │
│                │
│ goroutine A ↔ goroutine B  ← goroutine 切换            │
│  纯寄存器级   (~200ns)       无需进入内核                │
└───────────────┘
```

### 2.2 上下文切换保存的内容

**CPU 寄存器（通用寄存器、程序计数器 PC、栈指针 SP）**：
- 线程切换：完整 16 个通用寄存器 + SP + PC + FPU 状态
- goroutine 切换：只需保存 SP + PC + BP + 少量寄存器（运行时会保存 callee-saved 寄存器）

**TLB（Translation Lookaside Buffer）刷新**：
- 进程切换：TLB 必须 flush（虚拟地址空间变了）
- 线程切换：**TLB 不需要 flush**（同一地址空间）
- goroutine 切换：**TLB 不需要 flush**（同一地址空间）

### 2.3 上下文切换的性能数据

```
# 基准测试（Intel Xeon, 3.0GHz, Linux 5.4）
vmstat 1  # 查看 cs（context switch）列

# 每秒 > 10万次上下文切换 → CPU 忙在内核态 → 性能问题

# 典型瓶颈场景：
- epoll_wait + 大量短连接：线程池线程频繁切换
- Go 服务 epoll 少连接：goroutine 切换，代价极低
```

---

## 3. goroutine 与 OS 线程的映射关系

### 3.1 为什么 goroutine 轻量

goroutine 的轻量来自三个设计：

**① 初始栈极小（2KB vs 8MB）**
```go
// goroutine 创建时栈只有 2KB
// 栈增长：按需扩缩，每次 2倍，最多 1GB
// 线程栈：固定 8MB，创建即分配
```

**② 用户态调度，无系统调用开销**
```go
// goroutine 让出 P（不阻塞 M）的情况：
// 1. channel 阻塞（chan recv/send）
// 2. network I/O（epoll 事件）
// 3. sync 原语（Mutex.Wait）→ park G，运行时不阻塞 OS 线程

// 对比线程：
// 线程阻塞 → OS 调度器介入 → 必须陷入内核（context switch）
```

**③ M 数量与 G 分离**
```
GOMAXPROCS = 8  // P 数量 = 8（并行 goroutine 上限）

// 即使有 10000 个 goroutine 阻塞在 channel：
// M 数量 = ~8（不阻塞的）+ 若干阻塞线程
// goroutine 阻塞不导致 M 阻塞（Hand Off 机制）
```

### 3.2 goroutine 泄漏与排查

goroutine 泄漏会导致内存持续增长：

```go
// 典型泄漏场景
func leak() {
    ch := make(chan int)
    // 忘记 close 或发送
    // goroutine 永久阻塞在此，无法被 GC
    go func() { <-ch }()
}

// 正确做法：使用 context 取消
func notLeak(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case <-ch:
            // 正常处理
        case <-ctx.Done():
            return // context 取消时退出
        }
    }()
}
```

**排查方法：**

```bash
# 1. 查看 goroutine 数量
go tool pprof http://localhost:6060/debug/pprof/goroutine?debug=1

# 2. 压测观察 goroutine 数量是否线性增长（正常应该收敛）
# 3. 使用 goleak 做单元测试中的泄漏检测
import "go.uber.org/goleak"
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

---

## 4. 高频追问

### Q1：goroutine 很多时（百万级别），调度器会不会成为瓶颈？

会。Go 调度器本身是串行的（全局锁保护），但已优化：
- 本地队列（LRQ）减少全局锁竞争
- Work Stealing：空闲 P 从其他 P 的 LRQ 偷 G
- 大量 G 时使用 `P` 的全局队列轮询而非全部竞争

### Q2：goroutine 多了，内存会爆炸吗？

不会。goroutine 栈是动态增长的（2KB→1GB 按需），不像线程固定 8MB。
百万 goroutine ≈ 百万 × 平均 2KB ~ 2GB（可控）。
真正的问题通常是**业务逻辑持有的堆内存**，而非 goroutine 栈。

### Q3：多线程 vs 协程，适用场景是什么？

| 场景 | 方案 | 原因 |
|------|------|------|
| CPU 密集型（计算） | 多线程（利用多核）| goroutine 仍绑定 P，并行度 = GOMAXPROCS |
| I/O 密集型（网络）| goroutine（高并发）| 百万并发也能跑，单线程即可 |
| 实时系统 | 协程（无抢占延迟）| 线程调度延迟不可控 |
| 兼容 C 库 | 线程（pthread）| CGO 调用必须 LockOSThread |

---

## 5. 信号（Signal）：Go 服务优雅退出的底层机制（高频）

> 考察频率：★★★★☆  关键词：SIGTERM/SIGINT/SIGQUIT、SIGURG、SIGPIPE、signal.Notify、优雅停机

### 5.1 核心答案（30 秒版）

信号是内核发给进程的**异步通知**，本质是进程控制机制：

| 信号 | 默认行为 | Go 服务场景 |
|------|---------|------------|
| `SIGTERM` | 终止进程 | **K8s/Docker stop 时先发它**，应用应优雅退出 |
| `SIGINT` | 终止进程 | Ctrl+C，本地调试 |
| `SIGQUIT` | 终止+**core dump** | **Go 程序收到它默认打印所有 goroutine 栈**（排障利器！）|
| `SIGHUP` | 终止 | 经典用法：重载配置（nginx -s reload）|
| `SIGUSR1/2` | 终止 | 自定义：触发 GC dump、切换日志级别 |
| `SIGPIPE` | 终止 | 写已关闭的管道/连接 |
| `SIGKILL/SIGSTOP` | 终止/暂停 | **不可捕获**，OOMKilled 就是发 SIGKILL |
| `SIGURG` | 忽略 | **Go 1.14+ 抢占调度用它**，应用不要捕获 |

### 5.2 Go 中处理信号：signal.Notify

```go
// 标准优雅停机模式
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
    defer stop()

    srv := &http.Server{Addr: ":8080"}
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    <-ctx.Done() // 阻塞直到收到 SIGTERM/SIGINT
    log.Println("收到退出信号，开始优雅停机...")

    // 给存量请求最多 10 秒完成
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Println("强制关闭:", err)
    }
    log.Println("服务已退出")
}
```

### 5.3 Go 信号特性（面试加分点）

**① SIGQUIT 打印 goroutine 栈**：生产环境发现服务卡死，直接 `kill -QUIT <pid>`，日志里会出现所有 goroutine 的堆栈——这是 Go 工程师最常用的「手动 heap 分析」手段，无需 pprof 端口。

**② SIGURG 是 Go 的抢占信号**：Go 1.14 起，runtime 通过向运行中的 goroutine 发送 SIGURG 实现**异步抢占**（解决 GC 等待用户代码让出导致的 STW 过长）。所以用 `signal.Notify` 捕获 SIGURG 会**破坏调度器**，官方明确不建议捕获。

**③ SIGPIPE 的特殊处理**：Go runtime 对写入 **stdout/stderr** 的 SIGPIPE 直接终止进程（符合 Unix 惯例），但对写入其他 fd（如网络连接）的 SIGPIPE 会**转为 EPIPE 错误返回**，不会杀死进程——所以 Go 服务写已关闭的 TCP 连接不会崩。

**④ 信号与 goroutine**：`signal.Notify` 内部有专门 goroutine 接收信号转发到 channel，注意 channel 必须有消费者，否则信号处理 goroutine 阻塞。

### 5.4 面试话术

> 「Go 服务优雅停机我一般用 signal.NotifyContext 监听 SIGTERM/SIGINT，收到后给 http.Server.Shutdown 一个超时上下文，先摘流量、等存量请求完成、再关连接池。另外两个细节：生产排查卡死用 kill -QUIT 拿 goroutine 栈；SIGURG 是 Go 自己抢占用的，不能捕获。」

---

## 6. NUMA 与 CPU 亲和性：多路 CPU 下的 Go 服务优化（2026 升温）

> 考察频率：★★★☆☆  关键词：NUMA node、本地/远端内存、numactl、CPU 绑定、GOMAXPROCS

### 6.1 什么是 NUMA

多路（多 CPU 插槽）服务器上，内存被划分到不同的 **NUMA node**，每个 CPU 访问「本地 node」内存快（几十 ns），访问「远端 node」内存慢（跨 QPI/UPI 总线，延迟可高 1.5~2 倍，带宽受限）：

```
┌──── Socket 0 ────┐        ┌──── Socket 1 ────┐
│ CPU0  CPU1       │  QPI   │ CPU2  CPU3       │
│   ↓    ↓         │◄──────►│   ↓    ↓         │
│ 内存 node0       │ 总线   │ 内存 node1       │
│ （本地访问快）    │        │ （本地访问快）    │
└──────────────────┘        └──────────────────┘
   CPU0 访问 node1 = 远端访问 = 慢
```

### 6.2 查看与绑定

```bash
# 查看 NUMA 拓扑
numactl --hardware
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 ... 15
# node 0 size: 128 GB
# node 1 cpus: 16 17 18 ... 31
# node 1 size: 128 GB

# 绑定进程到 node0 运行（numactl 执行）
numactl --cpunodebind=0 --membind=0 ./your-app

# 查看进程当前 NUMA 分配
numastat -p <pid>
# 查看是否有跨 node 分配（node1 列非 0 说明发生了远端访问）
```

### 6.3 对 Go 服务的影响（面试核心）

**① 跨 node 内存访问拖慢性能**：Go 的 GC、对象分配可能把对象放在「离运行 CPU 很远」的 node，导致远端内存访问成为热点。典型症状：同样的 QPS，多路机器上性能不如单路，且 pprof 显示 `runtime.memmove`、`mallocgc` 耗时异常。

**② GOMAXPROCS 必须感知容器配额**：Go 1.19 起默认根据 **cgroup CPU quota** 自动设置 GOMAXPROCS（而不是物理核数）。在 64 核宿主机上给容器 4 核 quota，如果不感知会把 GOMAXPROCS 设为 64，导致大量线程争抢、上下文切换爆炸。

```go
// 手动查看/设置（一般不用，runtime 已自动）
fmt.Println("GOMAXPROCS =", runtime.GOMAXPROCS(0))
// 容器场景可用 automaxprocs 库兜底
// import _ "go.uber.org/automaxprocs"
```

**③ CPU 亲和性（taskset）**：把 CPU 密集型服务绑定到固定 CPU 核，减少 cache 失效和调度迁移：

```bash
taskset -c 0-3 ./your-app   # 只允许跑在 0~3 核
```

**④ 网卡中断绑定**：高吞吐网络服务把网卡 IRQ 绑定到与业务 CPU 同一 node，否则中断处理与业务跨 node 通信，吞吐明显下降（`/proc/irq/<irq>/smp_affinity`）。

### 6.4 面试话术

> 「NUMA 的核心是内存访问延迟不对称，本地快远端慢。Go 服务在多路机器上我主要注意三点：一是 GOMAXPROCS 要感知 cgroup quota，Go 1.19+ 已自动做；二是 CPU 密集服务用 taskset 绑定核；三是高吞吐场景把网卡中断和业务 CPU 绑到同一 node。排查跨 node 访问用 numastat 看 node 分布。」

---

## 延伸阅读

- [Go Runtime Scheduler 源码分析](https://github.com/golang/go/blob/master/src/runtime/proc.go)
- [Linux Context Switch 性能测试](https://blog.fluentbit.io/context-switching-overhead/)
- [Go 官方 os/signal 文档](https://pkg.go.dev/os/signal)
- [Linux NUMA 架构详解](https://www.kernel.org/doc/html/latest/vm/numa.html)
