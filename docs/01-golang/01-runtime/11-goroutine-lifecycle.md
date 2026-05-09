# Goroutine 全生命周期：创建 / 运行 / 阻塞 / 退出与泄漏排查

> 考察频率：★★★★☆  优先级：P0（必考知识点）

---

## 面试官考察意图

这道题考察候选人对 goroutine **完整生命周期**的理解深度。
初级只知道"go func() 创建，调度器自动执行"，高级要能讲清楚 **G 的状态机流转、P 的调度循环、阻塞的分类（channel/Mutex/syscall/GC）以及泄漏的根因定位**，并能用 pprof 和 trace 工具有效排查生产问题。

---

## 核心答案（30 秒版）

Goroutine 生命周期：

```
新建（Created） → 就绪（Runnable） → 运行（Running） → 阻塞（Blocked） → 退出（Dead）
     │                    │                    │
     └────────────────────┴────────────────────┘
                  调度循环（schedule）
```

| 状态 | 含义 | 触发 |
|------|------|------|
| **Created** | goroutine 被创建，但未进入调度 | `go func()` |
| **Runnable** | 等待调度，可被 P 取出执行 | 编译器和调度器设置 |
| **Running** | 正在 P 上执行 | 被 M 绑定执行 |
| **Blocked** | 因 channel/Mutex/syscall/GC 等原因挂起 | 等待事件 |
| **Dead** | 执行完毕，栈被回收 | 函数返回或 panic |

Goroutine 泄漏的三大根因：**channel 死锁、select 永远阻塞、http.Client 未设置 timeout**。

---

## 深度展开

### 1. Goroutine 状态机

#### 1.1 状态定义（src/runtime/runnable2.go / racewalk.go）

Go runtime 用 `G` 结构体描述 goroutine，状态字段为 `atomicstatus`：

```go
// src/runtime/runtime2.go（简化版）
type g struct {
    stack       stack         // 栈：[lo, hi)
    stackguard0 uintptr       // 栈顶guard，用于栈增长检查
    m            *m           // 当前绑定的 M
    sched        gobuf        // 调度上下文（PC/SP/DL/BP）
    atomicstatus uint32       // Goroutine 状态
    waitreason   string       // 阻塞原因（调试用）
    goid         uint64      // 全局唯一 ID
    ...
}
```

**状态常量：**

```go
// src/runtime/runtime2.go
const (
    _Gidle    = iota  // 0: 刚分配，尚未初始化
    _Grunnable        // 1: 在队列中，等待被调度
    _Grunning         // 2: 正在 P 上运行（只能由调度器设置）
    _Gsyscall         // 3: 正在执行系统调用（不占 P）
    _Gwaiting         // 4: 因 channel/mutex/syscall/GC 阻塞
    _Gdead            // 5: 已退出，栈已释放
    _Gcopystack       // 6: 正在复制栈（栈增长时）
)
```

#### 1.2 生命周期流转图

```
                    ┌─────────────────────────────────┐
                    │           go func()             │
                    │        → newcreatefn()           │
                    └───────────────┬─────────────────┘
                                    │
                                    ▼
                              status = _Grunnable
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │     P 的本地队列 / 全局队列    │
                    │   findrunnable() 取出 G        │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                          ┌─────────────────┐
                          │ status = _Running│ ← M 绑定 P 执行
                          │  执行用户代码    │
                          └────────┬────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
            ▼                      ▼                      ▼
    ┌──────────────┐     ┌────────────────┐     ┌──────────────┐
    │syscall阻塞   │     │  channel阻塞   │     │  函数执行完  │
    │status=_Gsyscall│  │status=_Gwaiting│     │  status=_Gdead│
    └───────┬───────┘     └───────┬────────┘     └──────────────┘
            │                     │
            │                     ▼
            │            ┌──────────────────┐
            │            │ recvq.sendq收到  │ → 唤醒后重新进入
            │            │ status=_Grunnable │   调度循环
            │            └──────────────────┘
            │
            ▼
    ┌──────────────────┐
    │ sysmon 检测到    │ → 重新进入 findrunnable()
    │ syscall 结束     │
    └──────────────────┘
```

### 2. 创建：go func() 的完整流程

#### 2.1 编译期 + 运行时

```go
// 用户代码
go func() {
    doSomething()
}()
```

**编译期：** 编译器将 `go func()` 转换成 `runtime.newproc()` 调用。

**运行时 newproc：**

```go
// src/runtime/proc.go
func newproc(siz int32, fn *funcval) {
    // 1. 从当前 goroutine 的栈获取参数（go func() 的参数）
    argp := add(unsafe.Pointer(&fn), sys.PtrSize)
    gp := getg()                 // 当前 goroutine
    pc := getcallerpc()          // 调用 newproc 的返回地址

    // 2. 获取 P 的 gFree 栈缓存（无锁，快速分配）
    pp := getp()
    newg := gfget(pp)            // 优先从 gFree 列表获取

    // 3. 初始化 goroutine
    schedgof(newg).goid = cas64(&nextgoid, atomic.Add64(&boundid, 1))
    newg.startpc = fn.fn         // 入口函数

    // 4. 设置初始状态为 runnable
    casgstatus(newg, _Gdead, _Grunnable)

    // 5. 加入 P 的本地运行队列
    runqput(pp, newg, true)     // true = 优先加入队头（抢占式）
}
```

#### 2.2 Goroutine 栈的初始大小

| Go 版本 | 初始栈大小 | 增长方式 |
|---------|-----------|---------|
| Go 1.0~1.9 | 8KB | 分段栈（stack segment） |
| Go 1.10~1.16 | 2KB | 连续栈（contiguous stack） |
| Go 1.17+ | 2KB（amd64/arm64） | 连续栈，编译器预热 |

```go
// 初始栈分配（src/runtime/stack.go）
func newstack() {
    // 检查栈指针是否超过 stackguard0（栈增长阈值）
    if sp < moffset {
        // 触发栈增长：分配 2x 大小的新栈
        // 复制旧栈内容到新栈
        // 更新 SP 寄存器
    }
}
```

### 3. 阻塞：Goroutine 阻塞的分类

#### 3.1 Channel 阻塞

```go
// channel 发送阻塞（src/runtime/chan.go）
func chansend(c *hchan, elem unsafe.Pointer, block bool) {
    // ... 
    // 1. 环形队列有空间：直接入队
    if c.qcount < c.dataqsiz {
        gp := getg()
        // 将数据写入 buffer...
        sendq.enqueue(gp)    // G 加入 sendq
        casgstatus(gp, _Grunning, _Gwaiting)  // 状态变为 waiting
        // 当前 M 释放 P，开始调度下一个 G
        goSched()
    }
}
```

**G 被挂起时：**
- M 不阻塞，继续执行其他 G（高并发保证）
- G 加入 `hchan.sendq` 链表，等待被唤醒

#### 3.2 Mutex 阻塞

```go
// Mutex.Lock() 互斥锁阻塞（src/runtime_mutex.go）
func lock2(s *mutex) {
    if atomic.Cas(&s.key, 0, locked) {
        return  // 快速获得锁
    }
    // 慢路径：自旋 + 阻塞
    semacquire(&s.n, 1)   // 实际调用 futex 系统调用
    // 阻塞在此，直到锁被释放
}
```

#### 3.3 系统调用阻塞

```
G 执行 read() → 进入内核 → M 阻塞 → P 被"让出"（hand off）
                                    ↓
                           sysmon 检测到 M 长期阻塞
                                    ↓
                           将 P 分配给其他 M
                                    ↓
                           被阻塞的 G/syscall 结束后，重新加入全局队列
```

### 4. 退出：Goroutine 死亡流程

```go
// goroutine 退出（src/runtime/proc.go）
func goexit() {
    // 1. 调用 defer（如果存在）
    // 2. 状态变为 _Gdead
    casgstatus(gp, _Grunning, _Gdead)
    // 3. 将 gFree 归还到 P 的 gFree 列表（栈被释放）
    gfput(pp, gp)        // 栈被回收，可复用
    // 4. 触发新一轮调度
    goSched()
}
```

**关键点：** 死亡不等于内存释放。栈归还到 `gFree` 列表，供后续 `newproc` 复用，无需 malloc。

### 5. Goroutine 泄漏：根因与排查

#### 5.1 泄漏的分类

| 类型 | 根因 | 典型场景 |
|------|------|----------|
| **channel 泄漏** | 发送者未关闭 channel，接收者永远阻塞 | 请求超时未 close、select 缺少 default |
| **select 泄漏** | select 没有任何 case 能执行，且无 default | 遗漏 default |
| **HTTP Client 泄漏** | 未设置 Timeout，长连接未释放 | http.Get 无 context |
| **Timer 泄漏** | time.Timer 未 Stop，GC 无法回收 | 循环创建 Timer 未清理 |
| **Context 泄漏** | 父 context 已取消，子 goroutine 继续执行 | 未监听 ctx.Done() |

#### 5.2 实战：HTTP Client 超时泄漏

```go
// 错误写法：泄漏 goroutine
resp, err := http.Get("http://slow-server/api")
// 如果 slow-server 不响应，goroutine 永远阻塞在 read()

// 正确写法
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
req, _ := http.NewRequestWithContext(ctx, "GET", "http://slow-server/api", nil)
resp, err := http.DefaultClient.Do(req)
```

#### 5.3 生产排查 SOP

**Step 1：pprof goroutine profile（推荐）**

```bash
# 1. 采集 goroutine profile（30 秒）
curl http://localhost:6060/debug/pprof/goroutine?debug=1

# 2. 用 go tool pprof 分析
go tool pprof http://localhost:6060/debug/pprof goroutine
(pprof) top 10
(pprof) traces

# 3. 关注 goroutine 数量（正常 < 1000）
# 持续增长 = 泄漏
```

**Step 2：runtime/trace（精确追踪）**

```go
// 在代码中加 trace
import "runtime/trace"
func doWork() {
    ctx, tr := trace.NewContext(context.Background())
    defer tr.End()
    // ...
}
```

```bash
# 启动 trace
curl http://localhost:6060/debug/pprof/trace?seconds=30
go tool trace trace.out
# 查看 goroutine 状态随时间的变化
```

**Step 3：泄漏告警**

```go
// Prometheus 采集 goroutine 数
var goroutineNum = prometheus.NewGaugeFunc(prometheus.GaugeOpts{
    Name: "go_goroutines",
    Help: "Current number of goroutines.",
}, func() float64 {
    var stats runtime.MemStats
    runtime.ReadMemStats(&stats)
    return float64(stats.NumGoroutine)
})
```

```yaml
# Prometheus 告警规则
- alert: GoroutineLeak
  expr: go_goroutines > 5000
  for: 5m
  annotations:
    summary: "Goroutine 数量持续超过 5000，可能存在泄漏"
```

### 6. Go 1.27 leak Profile（推荐）

Go 1.27 默认启用的 `goroutineleak` profile，通过 GC 追踪不可达的 goroutine，是发现泄漏的利器：

```bash
# Go 1.27+ 默认开启
# 直接在 pprof 页面查看
go tool pprof http://localhost:6060/debug/pprof/goroutine

# 或使用 trace
curl -s http://localhost:6060/debug/pprof/goroutine?seconds=30 > leak.prof
go tool pprof -http=:9090 leak.prof
```

---

## 高频追问

### Q1：Goroutine 和 OS 线程的区别？

| 对比维度 | Goroutine | OS 线程 |
|---------|-----------|---------|
| 创建成本 | ~2KB 栈，0 成本 | ~1MB 栈，系统调用 |
| 调度 | 用户态调度（Go runtime） | 内核调度（CPU 时间片） |
| 切换开销 | ~ns 级（寄存器保存） | ~μs 级（特权级切换） |
| 数量限制 | 可轻松创建百万级 | 通常数千 |
| 阻塞影响 | Channel 阻塞仅阻塞 G，M 不阻塞 | 系统调用阻塞，M 阻塞 |

### Q2：Goroutine 栈为什么是 2KB 而不是 1MB？

这是 Go 团队的工程选择：初始栈设小（2KB）可以支持创建百万 goroutine；栈增长是按需的（翻倍增长），运行时开销可控。相比之下，OS 线程的 1MB 是内核预留的固定值。

### Q3：Goroutine 泄漏和内存泄漏有什么区别？

Goroutine 泄漏：G 卡在 waiting 状态，不会被 GC 回收（M + 栈），导致持续消耗内存和调度资源。

内存泄漏：内存分配后失去引用，GC 可以回收，但持续增长会耗尽内存。

**区别：** Goroutine 泄漏必然导致内存泄漏（goroutine 栈不回收），但内存泄漏不一定有 goroutine 泄漏。

### Q4：如何防止 channel 泄漏？

1. 确保每个 channel 有对应的接收方
2. 使用 `close(c)` 或 `defer cancel()` 及时关闭
3. `select` 始终加 `default` case 防止永远阻塞
4. 用 `chan struct{}` 而非 `chan value` 减少内存占用

---

## 延伸阅读

- [Goroutine Leak - go.dev](https://go.dev/blog/hangdetect)
- [Go scheduler - bit.dp.ua](https://tip.golang.org/doc/codewalk/sharemem)
- [Contiguous stacks - golang.org](https://golang.org/doc/go1.3#stack)
- 《Go 调度器源码分析》—— 柴杰，调度循环章节