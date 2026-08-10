[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 并发编程](../README.md)

---

# Goroutine vs OS 线程：核心差异与调度开销

## 面试官考察意图

考察候选人对 Go 并发模型的理解深度，以及是否能从**调度开销、内存占用、上下文切换成本**等维度讲清楚为什么 goroutine 如此轻量。
初级只能说出"goroutine 比线程轻量"，高级要能给出**可量化的数据、底层原因、以及在生产中的实际意义**。

---

## 核心答案（30 秒版）

| 对比维度 | OS 线程 | Goroutine |
|---------|---------|-----------|
| **创建成本** | ~1MB 栈，调用内核 | ~2KB 初始栈，可动态增长 |
| **切换成本** | 约 1~2μs（内核态）| 约 0.2~0.5μs（用户态） |
| **调度方式** | 内核抢占式（时间片）| 协作+抢占混合（GMP模型）|
| **内存占用** | 固定 1MB（不可动态调整）| 2KB~1GB 动态增长 |
| **创建速度** | 慢（需系统调用）| 快（纯用户态）|

goroutine 轻量的根本原因：**用户态调度 + 动态栈 + 复用线程（M）**。一次线程创建代价约 1~10ms，而 goroutine 仅 ~0.1ms。

---

## 深度展开

### 1. 内存占用对比

**OS 线程**的栈大小是固定的：

```
Java Thread:     默认 1MB（可调整，但浪费严重）
pthread_create:  8~16MB（取决于实现）
Linux 内核:      8KB（最小）+ 映射开销
```

固定栈的问题：如果线程只需要 10KB 栈，仍然占用 1MB 内存，**1000 个线程 = 1GB 内存**。

**Goroutine**的栈是**动态增长**的：

```go
// src/runtime/runtime2.go
type g struct {
    stack   stack        // 栈：[stack.lo, stack.hi)
    stackguard0 uintptr  // 栈溢出检测阈值
    stackguard1 uintptr
    ...
}

// 初始栈大小：2KB（Go 1.4+）
// 最大栈大小：1GB（可通过 debug.SetMaxStack 调整）
```

动态栈增长通过**连续栈（Contiguous Stack）**实现：

```go
// 栈空间不足时，分配 2 倍大的新栈，复制旧栈内容
// Go 1.14 之前用分段栈，Go 1.14+ 用连续栈（避免栈分裂的开销）
func newstack() {
    oldSize := thisg.m.curg.stack.hi - thisg.m.curg.stack.lo
    newSize := oldSize * 2
    // 复制旧栈到新栈，调整指针
}
```

| 场景 | 1000 线程内存 | 1000 goroutine 内存 |
|------|--------------|-------------------|
| 空闲 | ~1GB | ~2MB |
| 峰值（各 100KB 栈）| ~100MB | ~100MB |

### 2. 上下文切换成本

**线程切换**需要内核介入（上下文切换）：

```
用户态 → 内核态（syscall/软中断）
保存寄存器：PC、SP、通用寄存器、浮点寄存器（约 16~32 个）
切换页表（TLB 刷新）
切换内核态栈
→ 约 1~2μs（CPU 时钟周期 ~3000 个）
```

**Goroutine 切换**在用户态完成，无需进入内核：

```go
// gosave/saveig：保存 goroutine 上下文到 g.sched
// goyield：主动让出（协作调度）
// noteswitch：触发调度器切换
```

```
用户态寄存器保存（g.sched）
选择下一个可运行的 G
恢复 G 的寄存器
→ 约 0.2~0.5μs
```

> 注意：goroutine 切换**可能**也会触发线程切换（如果当前 G 阻塞），那时就是完整的线程切换成本。

### 3. 调度模型对比

**OS 线程调度**（内核级）：

```
内核调度器（Completely Fair Scheduler in Linux）
    ↓
时间片轮转（通常 4~100ms）
    ↓
所有线程公平竞争 CPU
```

**GMP 调度**（用户级 + 协作/抢占混合）：

```
G（goroutine） → P（逻辑处理器） → M（内核线程）
    |                    |
    └── 本地 G 队列       └── 最多 GOMAXPROCS 个 P
                            |
                            └── 全局队列 + Work Stealing
```

GMP 的优势：
1. **减少内核调用**：P 绑定 M 后，无需每次都进内核
2. **Work Stealing**：本地队列空时从其他 P 偷一半 G，减少空闲
3. **Hand Off**：G 调用阻塞 syscall 时，M 释放 P 给其他 M用，CPU 不空转
4. **抢占式调度**：Go 1.14+ 基于信号的抢占（sysmon），避免长任务霸占 P

### 4. 为什么 goroutine 可以创建数十万个？

关键在于**复用 M（线程）**：

```go
// 不需要为每个 goroutine 创建一个线程
// 10万个 goroutine 只需要 ~N 个 M（N = GOMAXPROCS 或稍多）
runtime.GOMAXPROCS(8)  // 默认 = CPU 核数，通常只需要 8~16 个 M
```

对比 Java：

```go
// Java: 每个线程一个 OS 线程
// 10万线程 = 10万内核线程 → 内存爆炸 + 调度龟速
Thread t = new Thread(() -> {...});
t.start();  // 每个线程 1MB 栈
```

### 5. 生产经验：goroutine 泄漏的危害

虽然 goroutine 非常轻量，但**泄漏**仍然会拖垮服务：

```go
// 泄漏场景 1：channel 忘记 close，接收方永远阻塞
func leak() {
    ch := make(chan int)
    go func() {
        // 忘记 close(ch)
        for i := range ch {
            fmt.Println(i)
        }
    }()
    // 主 goroutine 退出，子 goroutine 泄漏
}

// 泄漏场景 2：goroutine 内的 goroutine 泄漏
func leak2() {
    for i := 0; i < 10; i++ {
        go func() {
            subCh := make(chan int)
            go func() {
                // 内层 goroutine 泄漏
                <-subCh
            }()
            // 如果这里阻塞，外层 goroutine 也泄漏
        }()
    }
}
```

goroutine 泄漏排查：

```bash
# 1. pprof goroutine profile（对比前后）
go tool pprof -http=:8080 http://localhost:8080/debug/pprof/goroutine?debug=1

# 2. 查看 goroutine 数量监控（Prometheus）
# gauge go_goroutines 异常增长 = 泄漏信号

# 3. 查看泄漏 goroutine 的堆栈
curl http://localhost:8080/debug/pprof/goroutine?debug=2 > goroutine.txt
```

---

## 高频追问

### Q1：goroutine 是协作式调度还是抢占式调度？

Go 1.14 之前是**协作式调度**（非抢占），goroutine 必须主动让出（如 channel 操作、sleep、mutex）才能切换。问题：**一个无限循环的 goroutine 会永久占用 P，导致其他 G 饿死**。

Go 1.14+ 引入**基于信号的抢占式调度**（`sysmon` + `SIGURG`）：
- GC 触发 STW 时，向所有 M 发送信号强制抢占
- 禁止 for 循环的 goroutine 永久占用 P
- 栈扫描期间，如果 G 正在执行且超过 `stackPreempt` 阈值，强制让出

### Q2：一个 M（线程）可以同时运行多个 G 吗？

**不能**，同一时刻一个 M 只能执行一个 G。但 GMP 模型中：
- 多个 G 可能在**同一个 P 的本地队列**中等待
- M 在它们之间**快速切换**（每条指令或调度点）
- 切换频率极高（毫秒级），看起来像"并发"

### Q3：GOMAXPROCS 默认值是多少？可以调大吗？

```go
// 默认 = CPU 逻辑核心数
fmt.Println(runtime.GOMAXPROCS(0))  // 查询当前值

// 可以调大（在 IO 密集型场景可能有收益）
runtime.GOMAXPROCS(32)  // 但通常不建议超过 CPU 核数
```

> **注意**：GOMAXPROCS 只影响可以同时运行的 P 的数量，而不是 goroutine 总数。数十万个 goroutine 完全可以创建，但同时运行的上限是 GOMAXPROCS。

### Q4：channel 阻塞会切换到其他 goroutine 吗？

**会**。当 goroutine 在 `sendq` 或 `recvq` 中等待时：
- G 状态变为 `_Gwaiting`
- P 选择下一个 G 执行
- 唤醒时根据 channel 的 `buf` 状态决定继续 send 还是 recv

```go
// 无缓冲 channel send 阻塞的全流程
ch <- 1
// 1. 获取锁
// 2. 检查 buf 是否满
// 3. 满 → gopark（切换 G，状态 = _Gwaiting）
// 4. P 调度下一个 G
```

---

## 延伸阅读

- [Go 调度器源码：schedule()/findRunnable()](https://github.com/golang/go/blob/master/src/runtime/proc.go)
- [Go 动态栈实现：continuous stack](https://github.com/golang/go/blob/master/src/runtime/stack.go)
- [深度解析 Go GMP 调度模型](https://www.cnblogs.com/yangykaifa/p/archive/2026/01/18)
- [Go 1.14 抢占式调度实现](https://github.com/golang/go/issues/24503)
