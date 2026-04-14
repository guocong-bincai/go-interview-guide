[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 运行时原理](../README.md)

---

# GMP 调度模型

## 面试官考察意图

考察候选人对 Go 运行时的理解深度。
初级候选人只能说"goroutine 比线程轻量"，高级候选人要能讲清楚**为什么轻量、调度器如何工作、遇到阻塞时发生了什么**，并结合实际问题（goroutine 泄漏、调度延迟）给出生产经验。

---

## 核心答案（30 秒版）

Go 使用 **GMP 模型**实现用户态协程调度：

- **G（Goroutine）**：用户态协程，初始栈 2~8KB，按需增长
- **M（Machine）**：OS 线程，真正执行代码的载体
- **P（Processor）**：逻辑处理器，持有本地运行队列，数量默认等于 `GOMAXPROCS`

调度核心思想：**M 必须绑定 P 才能执行 G**，P 的数量决定并行度上限。
G 不直接绑定 OS 线程，由运行时在 M 上多路复用，避免了线程创建/销毁的系统调用开销。

---

## 深度展开

### 1. 三者关系与数量

```
全局运行队列 (GRQ)
      │
      ▼
 ┌────────────────────────────────────┐
 │  P0          P1          P2        │
 │  LRQ         LRQ         LRQ       │  ← 每个 P 有本地队列，最多 256 个 G
 │  │           │           │         │
 │  M0          M1          M2        │  ← M 与 P 1:1 绑定运行
 └────────────────────────────────────┘
       │
       ▼  M 阻塞时 P 会解绑，找空闲 M 或新建 M
   OS Thread
```

| 组件 | 典型数量 | 上限 |
|------|----------|------|
| G | 百万级 | 内存限制 |
| M | 数十~数百 | `runtime/debug.SetMaxThreads`，默认 10000 |
| P | = `GOMAXPROCS` | = CPU 核数 |

### 2. G 的生命周期

```
go func(){}()
    │
    ▼
  _Grunnable   ← 放入 P 的本地队列或全局队列
    │
    ▼  P 调度
  _Grunning    ← M 执行 G
    │
    ├──► 主动让出（runtime.Gosched / channel 阻塞）→ _Grunnable
    ├──► 系统调用阻塞  → _Gsyscall
    ├──► 网络 I/O 阻塞 → _Gwaiting（交给 netpoller）
    └──► 执行结束      → _Gdead（栈被回收复用）
```

### 3. Work Stealing（工作窃取）

当某个 P 的本地队列为空时，不会立即让 M 休眠，而是：

1. 先检查**全局队列**（每隔 61 次调度强制检查一次，避免全局队列饥饿）
2. 从**其他 P 的本地队列尾部**偷取一半的 G
3. 检查 **netpoller**，看是否有就绪的网络 I/O G
4. 以上都没有，M 进入休眠，P 归还到空闲列表

```go
// runtime/proc.go 核心逻辑（简化）
func findRunnable() (gp *g) {
    // 1. 本地队列
    if gp = runqget(_p_); gp != nil { return }
    // 2. 全局队列（每 61 次检查）
    if gp = globrunqget(_p_, 0); gp != nil { return }
    // 3. netpoller
    if gp = netpoll(0); gp != nil { return }
    // 4. work stealing
    for i := 0; i < 4; i++ {
        gp = runqsteal(_p_, victim, stealRunNextG)
        if gp != nil { return }
    }
    return nil
}
```

### 4. Hand Off（移交机制）

当 M 执行的 G 发生**系统调用阻塞**时：

```
M0 调用 syscall
    │
    ▼
sysmon 检测到 M0 阻塞（超过 20μs）
    │
    ▼
P 从 M0 解绑 → 绑定到空闲 M1（或新建 M1）
    │
    ▼
M1 继续调度 P 上的其他 G
    │
    ▼
M0 syscall 返回 → 尝试拿回 P，失败则 G 进入全局队列，M0 休眠
```

**为什么网络 I/O 不需要 Hand Off？**
Go 的网络 I/O 通过 `netpoller`（epoll/kqueue）实现非阻塞，G 挂起时不占用 M，M 可以继续执行其他 G。

### 5. sysmon 监控线程

sysmon 是一个**不需要 P 就能运行**的后台线程，负责：

| 职责 | 触发条件 | 动作 |
|------|----------|------|
| 抢占调度 | G 运行超过 10ms | 设置 `stackPreempt` 标志，G 在函数调用时让出 |
| Hand Off | M 陷入 syscall > 20μs | 解绑 P，移交给其他 M |
| netpoller | 有就绪的网络事件 | 唤醒对应 G |
| GC 触发 | 堆内存达到阈值 | 触发 GC |

> Go 1.14 起支持**基于信号的异步抢占**（SIGURG），解决了老版本 for 死循环无法被抢占的问题。

### 6. 生产经验：调度相关问题排查

**场景 1：goroutine 数量暴增**

```go
// 问题：未控制并发数，G 堆积
for _, req := range requests {
    go process(req) // 瞬间创建数万 G
}

// 修复：用 worker pool 控制并发
sem := make(chan struct{}, 100) // 最多 100 个并发
for _, req := range requests {
    sem <- struct{}{}
    go func(r Request) {
        defer func() { <-sem }()
        process(r)
    }(req)
}
```

**场景 2：GOMAXPROCS 设置不当（容器环境）**

容器中默认读取宿主机 CPU 数，如宿主机 96 核，但容器只分配了 4 核：

```go
import _ "go.uber.org/automaxprocs" // 自动读取 cgroup 配额设置 GOMAXPROCS
```

**场景 3：局部性优化**

G 和数据在同一 P 上处理时，CPU 缓存命中率更高：

```go
// 利用 P 本地性，避免跨 P 的数据竞争
// sync.Pool 的 Get/Put 优先操作当前 P 的本地池
```

---

## 高频追问

**Q：goroutine 和线程的区别？**

| 维度 | goroutine | OS 线程 |
|------|-----------|---------|
| 初始栈 | 2~8 KB，动态增长 | 固定 1~8 MB |
| 创建开销 | ~300ns，纯用户态 | ~10μs，需系统调用 |
| 切换开销 | 极少寄存器（PC/SP/DX） | 完整上下文 + TLB flush |
| 调度 | Go 运行时（协作+抢占） | OS 内核（完全抢占） |
| 数量 | 百万级 | 数千级（受 OS 限制） |

**Q：P 的数量设为多少合适？**

- **CPU 密集型**：`GOMAXPROCS = CPU 核数`（默认值），充分利用并行
- **I/O 密集型**：默认值已足够，I/O 阻塞的 G 不占 P
- **混合型**：通过压测确定，通常默认值是最优的

**Q：为什么要有 P 这一层，M 直接调度 G 不行吗？**

P 的核心价值：
1. **本地队列减少锁竞争**：G 放入/取出本地队列无需全局锁
2. **Work Stealing 的基础单位**：偷的是 P 的本地队列
3. **隔离 syscall 阻塞影响**：M 阻塞时 P 可以快速转移，不阻塞其他 G

**Q：Go 1.14 之前抢占有什么问题？**

1.14 之前是**协作式抢占**，只在函数调用时检查抢占标志。纯计算的死循环无法被抢占：

```go
// Go 1.13 及以前：这段代码会导致其他 goroutine 饿死
go func() {
    for {
        // 没有函数调用，sysmon 设的抢占标志永远不会被检查
        x++
    }
}()
```

1.14 通过向运行 G 的 M 发送 **SIGURG 信号**实现异步抢占，彻底解决此问题。

**Q：goroutine 泄漏如何排查？**

```go
// 通过 pprof 查看 goroutine 数量和堆栈
import _ "net/http/pprof"

// curl http://localhost:6060/debug/pprof/goroutine?debug=2
// 观察是否有大量 goroutine 阻塞在同一位置
```

常见泄漏原因：
- channel 发送/接收无人对端，永久阻塞
- `sync.WaitGroup` 计数未归零
- context 未传递导致 goroutine 无法退出

---

## 延伸阅读

- [Go runtime scheduler design](https://golang.org/s/go11sched)（Dmitry Vyukov 原始设计文档）
- [`runtime/proc.go`](https://github.com/golang/go/blob/master/src/runtime/proc.go)（调度器核心实现）
- [The Go scheduler](https://morsmachine.dk/go-scheduler)（图文讲解）
- [Go 1.14 release notes - Goroutine asynchronous preemption](https://golang.org/doc/go1.14)

---

**[← 上一篇：返回目录](../README.md)** · **[下一篇：GC 垃圾回收机制 →](./02-gc.md)**
