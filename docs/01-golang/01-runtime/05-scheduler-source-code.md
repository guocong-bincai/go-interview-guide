[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 运行时原理](../README.md)

---

# Go 调度器源码走读

## 面试官考察意图

这道题区分度极高。
初级候选人能讲 GMP 三要素，高级候选人能**对照源码讲清楚一次调度循环的完整链路**，包括：P 的本地队列如何窃取、全局队列如何避免饥饿、M 阻塞时 P 如何转移、网络 I/O 的协程如何被 epoll 唤醒。
如果还能讲清 `schedule()` 和 `findRunnable()` 的具体实现，说明候选人有源码阅读习惯，对 Go 运行时的理解已到工程级别。

---

## 核心答案（30 秒版）

Go 调度器的核心循环是：

```
M 执行 schedule()
  → findRunnable() 依次尝试：
      1. 本地队列 pop (runqget)
      2. 全局队列窃取 (globrunqget)
      3. 网络 I/O 唤醒 (netpoll)
      4. 跨 P 窃取 (runqsteal)
  → execute() 将 G 绑定到 M 的 g0 栈执行
  → G 主动让出 / 阻塞 / 时间片耗尽 → 重新 schedule()
```

所有调度行为集中在 `src/runtime/proc.go`，核心函数约 3000 行，是理解 Go 运行时最关键的文件。

---

## 深度展开

### 1. 源码环境说明

本文以 Go 1.22 源码为准，路径 `src/runtime/proc.go`。
可通过以下方式查看：

```bash
# 在线浏览（推荐）
https://github.com/golang/go/blob/master/src/runtime/proc.go

# 本地查看（需先 clone 完整 Go 源码）
ls $GOROOT/src/runtime/proc.go
```

关键概念提前说明：

| 变量/函数 | 含义 |
|-----------|------|
| `getg()` | 获取当前 goroutine 的 g 结构体指针 |
| `m0` | 启动进程的第一个 M（主线程） |
| `g0` | 每个 M 的管理用 g0 栈（8KB，比普通 G 栈大） |
| `allm` | 所有 M 的链表 |
| `allp` | 所有 P 的数组 |
| `sched` | 全局调度器状态（全局队列、idle M 列表等） |

### 2. g0 栈：调度器运行的"安全地带"

普通 G 的栈初始只有 2KB，调度器代码运行在 G 的栈上不安全（栈可能溢出）。
每个 M 有一个 `g0`，专用于运行调度器代码和 CGO：

```go
// src/runtime/proc.go
type m struct {
    g0      *g         // goroutine with scheduling stack
    morebuf gobuf      // gobuf arg to morestack
    // ...
}
```

`g0` 的栈在 `runtime/proc.go` 的 `mstart` → `mstart1` → `scheduler` 初始化时绑定，
大小比普通 G 大（通常 8KB），且**不会动态增长**，因为调度器代码本身不会递归太深。

### 3. G 的状态机

```go
// src/runtime/runtime2.go
const (
    _Gidle    = iota  // 0: 刚分配，未初始化
    _Grunnable        // 1: 在队列中，等待被调度
    _Grunning         // 2: 正在 M 上运行
    _Gsyscall         // 3: 正在执行系统调用
    _Gwaiting         // 4: 被阻塞（channel/select/sleep/锁）
    _Gdead            // 5: 已结束，栈可复用
    _Gcopystack       // 6: 正在复制栈（栈增长中）
)
```

状态转换图：

```
  _Gidle
    │ go 创建
    ▼
  _Grunnable ────────────────────────────────────────┐
    │ P 调度 execute()                                 │
    ▼                                                  │
  _Grunning ──┬────────────────────────────────────────┤
    │          │ G 主动让出 (Gosched)                   │
    │          ▼                                       │
    │     _Grunnable ←────────────────────────────────┤
    │                                                │
    │          │ channel/sleep/锁 等                  │
    │          ▼                                       │
    │     _Gwaiting ── 条件满足 ──► _Grunnable ────────┤
    │                                                │
    │          │ 系统调用                             │
    │          ▼                                       │
    │     _Gsyscall ── syscall 返回 ──► _Grunnable ───┤
    │                                                │
    │          │ 执行完毕                             │
    │          ▼                                       │
    └────► _Gdead ◄───────────────────────────────────┘
                │ gtmpel 回收或重用
                └────────────────────────────────────► _Gidle
```

### 4. 调度核心：schedule() 函数

```go
// src/runtime/proc.go:4230
func schedule() {
    _g_ := getg()  // 当前 g0

    // GC 或栈扫描时禁用调度
    if _g_.m.lockedm != 0 {
        // 锁定的 M 继续运行当前 G
    }

top:
    // 禁用 preemption（GC CG 等场景）
    pp := _g_.m.p.ptr()
    if pp == nil {
        // 无 P，只能执行 g0 上的特殊任务
    }

    // 如果在 GC mark 阶段且需要触发 GC_WORK 设置了 g 的优先级
    if sched.gcwaiting != 0 {
        goto top
    }

    // 从当前 P 本地队列、全局队列、netpoller 找 G
    var gp *g
    if trace.enabled {
        // tracing 逻辑
    }

    gp, inheritTime = runqget(_g_.m.p.ptr())
    // 如果本地队列有，直接返回 G

    // 本地队列没有，尝试全局队列
    gp, inheritTime = globrunqget(pp, 0)

    // 全局也没有，尝试 netpoll（异步网络 I/O）
    if gp == nil {
        gp, inheritTime = netpoll(0)
    }

    // 偷取其他 P 的 G
    if gp == nil {
        gp, inheritTime = stealWork(pp)
    }

    // 仍然没有，进入 idle 逻辑（等待网络事件或创建新 M）
    if gp == nil {
        goto top
    }

    // 执行 G
    execute(gp, inheritTime)
}
```

关键洞察：**每调用一次 schedule() 只调度一个 G**，而不是遍历整个队列。
这是 Go 调度器"先到先得"低延迟设计的一部分。

### 5. 窃取来源：findRunnable()（实际入口）

Go 源码中 `schedule()` 的真正入口通常通过 `findRunnable()` 封装，
逻辑顺序如下（对应 `proc.go` 中的 `findRunnable` 函数）：

```go
// src/runtime/proc.go:4360（简化版）
func findRunnable() (gp *g, inheritTime bool) {
    // ① 本地队列
    if gp = runqget(_p_); gp != nil {
        return
    }

    // ② 全局队列（每 61 次强制检查一次，避免饥饿）
    if _p_.schedtick%61 == 0 && gp = globrunqget(_p_, 0); gp != nil {
        return
    }

    // ③ 网络 I/O 事件（epoll/kqueue 返回的已就绪 G）
    if gp = netpoll(0); gp != nil {
        return
    }

    // ④ Work Stealing：随机窃取其他 P 的本地队列
    //    窃取一半，防止自己队列后半段饥饿
    for i := 0; i < 4; i++ {
        if gp = runqsteal(_p_, allp[rand], false); gp != nil {
            return
        }
    }

    return nil  // 确实没有 G，调用者会让 M 进入休眠
}
```

**为什么全局队列每 61 次才检查一次？**
因为全局队列操作需要 `sched.lock` 锁，如果每次调度都先抢锁会导致严重竞争。
61 是一个质数选择，降低与调度周期的共振频率。

### 6. G 的启动：execute()

```go
// src/runtime/proc.go:4450
func execute(gp *g, inheritTime bool) {
    _g_ := getg()

    // 更新 G 状态
    casgstatus(gp, _Grunnable, _Grunning)

    // 将 G 绑定到当前 M
    gp.m = _g_.m

    // 调度 tick +1
    gp.schedtick++

    // 初始化或恢复 G 的调度上下文
    // （把之前 save 的 PC/SP 等恢复到 CPU 寄存器）
    gostartcallfn(&gp.startpc, gp)
}
```

`execute()` 之后，G 就正式在 M 上执行了。
下一次调度发生时，当前 G 会通过 `goexit()` → `mcall(goexit0)` 回到 `schedule()`。

### 7. M 的管理：startm() / stopm() / wakep()

**启动一个 M（休眠唤醒或新建）：**

```go
// src/runtime/proc.go:4600
func startm(_p_ *p, spinning bool) {
    lock(&sched.lock)

    // 尝试从空闲 M 列表获取
    mp := sched.midle.pop()
    unlock(&sched.lock)

    if mp == nil {
        // 没有空闲 M，创建新的 OS 线程
        if spinning {
            // spinning M 不计入 running M 数量
        }
        newm(nil, _p_)
        return
    }

    // 唤醒已有 M 来绑定 P
    mp.spinning = spinning
    mp.nextp = _p_
    notewakeup(&mp.park)  // 唤醒休眠中的 M
}
```

**M 进入休眠（找不到 G）：**

```go
// src/runtime/proc.go:4670
func stopm() {
    // 解绑 P
    _p_ := releasep()
    noteclear(& Notesleep)
    // ...
    lock(&sched.lock)
    sched.midle.push(mp)  // 放回空闲列表
    unlock(&sched.lock)
    notesleep(&mp.park)   // 真正休眠
}
```

**唤醒 idle M 触发点（wakep）：**

```go
// src/runtime/proc.go:4550
func wakep() {
    // 如果有空闲 P，但没有空闲 M，唤醒或新建 M
    for pp := 0; pp < len(allp); pp++ {
        if p := allp[pp]; p.status == _Pidle {
            startm(p, true)  // spinning=true，不计入 running M 数
            break
        }
    }
}
```

### 8. 系统调用处理：entersyscall() / exitsyscall()

**进入阻塞 syscall：**

```go
// src/runtime/proc.go:4870
func entersyscall() {
    // 标记 G 正在执行系统调用
    casgstatus(gp, _Grunning, _Gsyscall)

    // 解绑 P（Hand Off 核心）
    releasep()

    // 更新 m.syscalltick（防 IPC 延迟）
    mp.syscalltick++
}
```

P 被释放后，sysmon 线程（或 `startm`）会把这个 P 分给其他空闲 M，
让其他 G 继续执行。**这就是 Hand Off 机制的实现**。

**从阻塞 syscall 返回：**

```go
// src/runtime/proc.go:4920
func exitsyscall() {
    // 尝试重新获取之前的 P
    if exitsyscallfast() {
        casgstatus(gp, _Gsyscall, _Grunning)
        return
    }

    // 获取失败，放到全局队列
    gp.schedflags |= _Grunnable
    goready(gp, 0)

    // 当前 M 进入休眠
    notesleep(&mp.park)
}
```

### 9. netpoller：Go 网络 I/O 的秘密武器

Go 不直接用阻塞的系统调用处理网络 I/O，而是用 epoll/kqueue 对所有 fd 进行管理：

```go
// src/runtime/netpoll.go
// epoll/ kqueue 封装
// 伪代码结构：
func netpoll(block bool) *g {
    // 调用 epoll_wait 或 kevent
    n := epoll_wait(epfd, events, nevents, nextTimeout)

    for i := 0; i < n; i++ {
        ev := events[i]
        gd := getgFromPollDesc(ev.data)

        // 把 G 标记为可运行，放入全局队列
        casgstatus(gd, _Gwaiting, _Grunnable)
        globrunqput(gd)
    }
    return nil
}
```

**G 在等待网络 I/O 时不占 M**：调用 `netpoll` 后 G 进入 `_Gwaiting`，
M 可以继续调度其他 G，**彻底解决了 C10K 问题**。

### 10. 生产经验：调度相关踩坑

**坑 1：for 死循环导致调度器饥饿（1.14 之前）**

Go 1.13 及之前是协作式抢占，如果 G 在一个无函数调用的循环中，`sysmon` 设置的抢占标志永远不会被检查：

```go
// Go 1.13：其他 goroutine 饿死，无法被调度
go func() {
    var x int64
    for {
        x++ // 没有函数调用，永远不让出 M
    }
}()

// 1.14+ 修复：SIGURG 强制抢占
// 建议：避免长计算循环，用 runtime.Gosched() 或显式 yield
```

**坑 2：CPU 密集型 G 长期占用 P（"小人国问题"）**

Go 的抢占调度基于信号，在某些极端情况下可能不够及时：

```go
// 长时间 CPU 计算，每 10ms 会触发一次抢占
// 但如果计算非常密集，P99 延迟可能波动较大
// 解决：用 GOMAXPROCS + 任务分片，或在计算中插入 runtime.Gosched()
```

**坑 3：sysmon 检测延迟导致 Hand Off 延迟**

sysmon 每 20μs 检测一次 P 是否被 M 阻塞 syscall，
如果 sysmon 本身被其他 G 阻塞（如 GC STW），Hand Off 可能延迟：

```go
// 生产问题：GC STW 期间所有 M 停住，P 无法及时 Hand Off
// 表现：高峰期 GC 后接口 P99 抖动
// 解决：调小 GOGC，或在重要服务中限制 GC 并发度
// GOGC=100（默认），GOGC=200 减少 GC 频率但增加内存
```

**坑 4：P 的数量与容器 CPU 配额不匹配**

```go
// 错误：宿主机 64 核，容器只有 2 核配额，但 GOMAXPROCS=64
// 导致容器被分配远超过自己 CPU 配额的调度时间片

// 正确做法：使用 automaxprocs 自动读取 cgroup
import "go.uber.org/automaxprocs"

func init() {
    // 自动设置为容器 CPU 配额
}
```

**坑 5：goroutine 泄漏导致 M 堆积**

goroutine 永久阻塞（无缓冲 channel 无人接收、WaitGroup 未 Done）→ G 不退出 → M 无法释放 → 最终耗尽 `MaxThreads`：

```
M 数量 = min(sched.maxmcount, 所有 G 都在阻塞 + P 数)
```

---

## 高频追问

**Q：P 的数量什么时候会变化？**

正常情况下 GOMAXPROCS 在程序启动后固定不变。但在 Go 1.5+ 内部 GC 时 P 数量会临时减少（从 `gcpercentage` 推导出的 `gcBgMarkWorkerCount`），以便给 GC mark worker 腾出 P。

**Q：调度延迟如何量化？**

`go scheduler latency` 是关键指标：
- 正常情况下，调度延迟 < 1μs（M 本地队列 pop）
- 从空闲到唤醒：~10~50μs（取决于系统调度器）
- 网络 I/O 唤醒：~50~200μs（取决于 epoll_wait 间隔）

**Q：g0 栈为什么比普通 G 栈大？**

因为 `g0` 要运行完整的调度器逻辑（`schedule()`、`execute()`、`newstack()` 等），这些函数加起来超过 2KB。
`g0` 栈是**静态分配**，固定 8KB，不会动态增长。

**Q：M 的数量上限是多少？**

默认 `Max Threads = 10000`（`runtime/debug.SetMaxThreads`）。
可以通过 `pprof` 查看 `goroutine` 数量，如果接近 M 上限，程序会 panic：

```
throw: all goroutines are asleep - deadlock!

// 或者
runtime: program exceeds 10000-thread limit
```

**Q：如何验证自己的调度理解是否正确？**

```go
// 方法1：用 pprof 观察 goroutine 堆栈
import _ "net/http/pprof"
// curl http://localhost:6060/debug/pprof/goroutine?debug=1

// 方法2：用 runtime.GOMAXPROCS + trace 观察
// go test -trace=/tmp/trace.out .
// 然后 go tool trace /tmp/trace.out

// 方法3：打印当前 P/M 状态
import "runtime"
fmt.Printf("GOMAXPROCS=%d, NumCPU=%d\n", runtime.GOMAXPROCS(0), runtime.NumCPU())
```

---

## 延伸阅读

- [Go runtime 源码：proc.go（推荐在线阅读）](https://github.com/golang/go/blob/master/src/runtime/proc.go)
- [Go runtime 源码：runtime2.go（G/P/M 结构体定义）](https://github.com/golang/go/blob/master/src/runtime/runtime2.go)
- [Go runtime 源码：netpoll.go（网络轮询器）](https://github.com/golang/go/blob/master/src/runtime/netpoll.go)
- [Scheduling In Go: Part I - Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)（ArdanLab 系列，图文并茂）
- [Scheduling In Go: Part II - The Go Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)
- [Scheduling In Go: Part III - Concurrency](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part3.html)
- [Go 1.14 异步抢占实现：signal_nonbsd.go](https://github.com/golang/go/blob/master/src/runtime/signal_nonbsd.go)

---

**[← 上一篇：goroutine 栈机制 →](./04-stack.md)** · **[下一篇：返回目录](../README.md)**
