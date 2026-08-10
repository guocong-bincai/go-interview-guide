[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 并发编程](../README.md)

---

# 死锁：原理与排查

## 面试官考察意图

考察候选人对并发编程中**最危险缺陷**的理解深度。
初级只能说出"加锁后忘了解锁"，高级要能讲清楚**死锁的四个必要条件、Go 里的典型场景、运行时检测机制、以及生产环境排查工具**。

---

## 核心答案（30 秒版）

死锁产生的四个必要条件（Coin 条件）：**互斥、占有并等待、非抢占、循环等待**。
Go 中最常见的死锁场景：**channel 发送/接收不匹配、sync.Mutex 重入、WaitGroup 计数错误**。运行时通过 `sysmon` 检测所有 goroutine 阻塞且无系统调用来发现死锁。

---

## 深度展开

### 1. 死锁的四个必要条件（Coin 条件）

| 条件 | 含义 | 破坏方式 |
|------|------|----------|
| **互斥** | 资源只能被一个 goroutine 持有 | 无法破坏（锁的本质） |
| **占有并等待** | 持有资源的同时请求其他资源 | 同时获取所有锁，或一口气 Unlock |
| **非抢占** | 资源无法被强制夺走 | 使用 `tryLock`（不阻塞） |
| **循环等待** | goroutine 间形成环形依赖 | 按固定顺序加锁，或一次性获取 |

### 2. Go 里最常见的死锁场景

#### 场景 1：channel 发送接收不匹配

```go
// 死锁：没人接收
ch := make(chan int)
ch <- 1  // 永久阻塞

// 死锁：没人发送
ch := make(chan int)
<-ch    // 永久阻塞

// 死锁：环形等待
ch1 := make(chan int)
ch2 := make(chan int)
go func() { ch1 <- 1; <-ch2 }()
go func() { ch2 <- 1; <-ch1 }()  // 互相等待

// ✅ 修复：确保有接收方或使用缓冲 channel
ch := make(chan int, 1)  // 缓冲 channel
ch <- 1  // 不阻塞
```

#### 场景 2：Mutex 重入（不可重入）

```go
var mu sync.Mutex

func A() {
    mu.Lock()
    defer mu.Unlock()
    B()  // ❌ B() 再次 Lock() → 永久阻塞
}

func B() {
    mu.Lock()  // 已持有锁，再次加锁 → 死锁
    defer mu.Unlock()
    // ...
}

// ✅ 修复：拆分为锁内/锁外两层，不重入
func A() {
    mu.Lock()
    // 只做必要的操作
    mu.Unlock()
    B()  // 调用时不持有锁
}
```

#### 场景 3：WaitGroup 计数错误

```go
// 死锁：Done() 少调用一次
var wg sync.WaitGroup
wg.Add(1)
go func() {
    // 任务（但忘了 defer wg.Done()）
    // wg.Wait() 永远阻塞
}()

// ✅ 修复：务必 defer
go func() {
    defer wg.Done()
    // 任务
}()

// 死锁：Add 次数不够
wg.Add(3)
go task1()
go task2()
// 漏掉了 task3，Wait 永远不等
```

#### 场景 4：select 永久阻塞

```go
// 死锁：select 所有分支都阻塞，且无 default
select {
case <-ch1:  // 没有人发送
    // ...
case <-ch2:  // 没有人发送
    // ...
}  // 永久阻塞

// ✅ 修复：加 default 或 timer
select {
case <-ch1:
case <-ch2:
case <-time.After(time.Second):
    // 超时退出
default:
    // 无可用水
}
```

### 3. Go 运行时死锁检测机制

Go 的 `sysmon`（系统监控线程）在启动时创建，它会检测**所有 goroutine 都处于 waiting 状态且没有可运行的 M**的情况，一旦发现立即 panic：

```
fatal error: all goroutines are asleep - deadlock!
```

**为什么无法检测有 golang 运行中的死锁？** 因为如果还有 goroutine 在运行（比如自旋等待），sysmon 无法判断这是"活跃工作"还是"死锁"。这也是为什么"有 goroutine 在忙等锁"的场景下，死锁不会被立即检测到。

### 4. 工具排查

#### 4.1 `go run -race`（竞态与部分死锁检测）

```bash
go run -race main.go
# 如果有 data race 导致 goroutine 阻塞等待，会报告
```

#### 4.2 `go tool trace`

```go
import "runtime/trace"

func main() {
    f, _ := os.Create("trace.out")
    defer f.Close()
    trace.Start(f)
    defer trace.Stop()
    
    // 程序逻辑
}
```

```bash
go tool trace trace.out
# 在浏览器中查看 goroutine 状态时间线
# 找出长期处于 waiting 状态的 goroutine
```

#### 4.3 `pprof`（ goroutine 分析）

```go
import _ "net/http/pprof"

func main() {
    go http.ListenAndServe(":6060", nil)
    // 程序逻辑
}
```

```bash
# 查看所有 goroutine 的堆栈
curl http://localhost:6060/debug/pprof/goroutine?debug=1

# 查看阻塞的 goroutine
curl http://localhost:6060/debug/pprof/block?debug=1
```

#### 4.4 Delve 调试器

```bash
dlv debug main.go
(dlv) goroutines
  Goroutine 1 - User:
// 列出所有 goroutine 及其状态
(dlv) bt  // 查看当前堆栈
(dlv) break *0x12345  // 在可疑地址设断点
```

### 5. 预防死锁的工程实践

#### 策略 1：按固定顺序获取锁

```go
// ❌ 危险：不同顺序可能导致死锁
func (a *A) Exchange(b *B) {
    muA.Lock()
    muB.Lock()  // 如果 B.Exchange(A) 同时调用，可能死锁
    // 交换逻辑
    muB.Unlock()
    muA.Unlock()
}

// ✅ 安全：始终按地址排序
func (a *A) Exchange(b *B) {
    // 按固定顺序加锁
    if reflect.ValueOf(a).Pointer() < reflect.ValueOf(b).Pointer() {
        muA.Lock()
        defer muB.Lock()
    } else {
        muB.Lock()
        defer muA.Lock()
    }
    // 或使用一个全局锁
}
```

#### 策略 2：使用 `TryLock` 避免阻塞

```go
// 检查锁是否能获取，不阻塞
if atomic.CompareAndSwapInt32(&mu.state, 0, 1) {
    // 获取到锁
} else {
    // 退避重试或降级
}
```

#### 策略 3：超时 + 重试

```go
func TryLock(m *sync.Mutex, timeout time.Duration) bool {
    deadline := time.Now().Add(timeout)
    for {
        if m.TryLock() {
            return true
        }
        if time.Now().After(deadline) {
            return false
        }
        time.Sleep(10 * time.Millisecond)
    }
}
```

#### 策略 4：channel 超时设计

```go
select {
case ch <- value:
    // 成功
case <-time.After(5 * time.Second):
    log.Warn("send timeout")
    return ErrTimeout
}
```

### 6. 生产案例分析

**案例：用户服务频繁卡死**

表现：用户服务响应时间 p99 突然飙升，goroutine 数量从 50 涨到 5000+。

排查步骤：

```bash
# 1. 查看 goroutine 堆栈
curl http://localhost:6060/debug/pprof/goroutine?debug=1 > goroutine.txt

# 2. 分析堆栈，发现大量 goroutine 停在同一个 channel 发送操作
# ch := make(chan *Request, 100)
# ch <- req  <- 大量在此阻塞

# 3. 查看 channel 接收方，发现接收方因连接池耗尽在等待 DB
# 结论：DB 慢 → channel 满 → 发送阻塞 → goroutine 积压 → 服务雪崩

# 修复：
# 1. 增加 channel 缓冲
# 2. 增加 DB 超时
# 3. 添加背压机制（channel 满时拒绝请求）
```

```go
// 背压机制：channel 满时快速失败
select {
case ch <- req:
    // 正常处理
default:
    metrics.Inc("back_pressure_total")
    http.Error(w, "service overloaded", http.StatusServiceUnavailable)
}
```

---

## 高频追问

**Q：Go 的 sync.Mutex 支持重入吗？**

不支持。重入会导致死锁，因为 Go 的 Mutex 依赖 state 字段判断锁是否被持有，没有记录持有锁的 goroutine 信息（不像 Java 的 ReentrantLock 记录 owner）。

**Q：死锁和活锁的区别？**

- **死锁**：goroutine 相互等待，线程/ goroutine 完全停止运行
- **活锁**：goroutine 持续运行但无法推进（如两个 goroutine 互相让路，但时序总是冲突）

**Q：如何检测业务代码中的潜在死锁？**

1. 代码审查：检查加锁顺序是否一致，是否有重入风险
2. `go vet -deadlock`：静态分析工具
3. 单元测试 + `go run -race`：在测试环境触发并发场景
4. 生产监控：goroutine 数量告警（正常应该稳定）

**Q：channel 的有缓冲 vs 无缓冲会影响死锁吗？**

会。无缓冲 channel 要求同时有发送和接收，一旦只有一方就会死锁。有缓冲 channel 在缓冲区满之前不会阻塞发送方，提供了额外的灵活性。

---

## 延伸阅读

- [Go 官方文档：Deadlocks](https://go.dev/blog deadlock)
- [Go 运行时源码：semaphore.go](https://github.com/golang/go/blob/master/src/runtime/sema.go)（信号量实现）
- [uber/goleak: goroutine 泄漏检测库](https://github.com/uber-go/goleak)
- [The Go Memory Model](https://go.dev/ref/mem)（happens-before 与死锁的关系）

---

**[← 上一篇：Race Condition](./race-condition.md)** · **[返回目录](../README.md)**
