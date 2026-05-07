# sync.Cond：条件变量与 Go 1.26 Breaking Change

> 考察频率：★★★☆☆  优先级：P1
> 关键词：Cond、Wait、Signal、Broadcast、Go 1.26 breaking change

## 面试官考察意图

考察候选人对 `sync.Cond` 条件变量的理解深度。
初级只知道 `Cond.Wait()` 会阻塞，`Signal()` 会唤醒，高级要能讲清楚**虚假唤醒（spurious wakeup）原理、Wait 的底层实现、以及 Go 1.26 对 Cond 行为的重要变更**，并能结合生产场景（连接池通知、任务队列唤醒）说明使用方式。

---

## 核心答案（30 秒版）

`sync.Cond` 是 Go 提供的**条件变量**，用于一组 goroutine 等待某个条件变为 true 后再继续：

| 操作 | 说明 |
|------|------|
| `Cond.Wait()` | 原子地释放锁并进入阻塞（必须持有锁调用）|
| `Cond.Signal()` | 唤醒**等待时间最长**的那个 goroutine |
| `Cond.Broadcast()` | 唤醒**所有**等待中的 goroutine |
| `NewCond(L locker)` | 用一个锁初始化条件变量 |

**Go 1.26 Breaking Change：** `Broadcast()` 不再能唤醒在调用 `Broadcast()` 之后才调用 `Wait()` 的 goroutine（之前是虚假唤醒最多种情况之一）。

---

## 深度展开

### 1. 什么是条件变量

条件变量（Condition Variable）是操作系统层面的并发原语，用于**等待某个谓词（predicate）变为 true**。

核心语义：
```
// 错误的做法：忙等待（浪费 CPU）
for !condition {
    time.Sleep(time.Millisecond) // ❌ 浪费 CPU
}

// 正确的做法：条件变量（阻塞等待）
mu.Lock()
for !condition {
    Cond.Wait()  // ✅ 释放锁、阻塞，等条件满足后自动醒来
}
mu.Unlock()
```

**为什么必须在 `for` 循环里 wait，而不是 `if`？**

→ **虚假唤醒（Spurious Wakeup）**：POSIX 标准允许在没有任何 Signal 的情况下唤醒 Wait，以减少无效的上下文切换。Go 的 `sync.Cond` 也遵循这一语义。因此必须在循环中重新检查条件：

```go
// 标准用法：for 循环内 Wait
mu.Lock()
defer mu.Unlock()
for !condition {
    cond.Wait()  // 醒来后要重新检查条件
}
// 此时 condition == true，继续执行
```

### 2. Cond 底层实现

```go
// src/sync/cond.go
type Cond struct {
    noCopy noCopy  // 禁止复制
    L      Locker  // 用户传入的锁，Wait 会原子释放它
    cq     *waiterChain  // 等待者链表（双向循环链表）
    sg     *waiterChain  // signal 队列（单次唤醒链）
}
```

**Wait 的原子性保证（核心）：**

```go
func (c *Cond) Wait() {
    // 1. 记录当前检查点（用于判断唤醒来源）
    c.checker.pack()  // 保存当前 waiter 链表快照

    // 2. 原子地：释放锁 + 阻塞 goroutine
    // 这两步是原子的，不会漏掉中间到达的 Signal
    runtime_notifyListWait(&c.notify.list)
    c.L.Lock()
}
```

**Signal 实现：**

```go
func (c *Cond) Signal() {
    // 从等待队列头部取出一个 waiter，唤醒它
    runtime_notifyListNotifyOne(&c.notify.list)
}
```

**Broadcast 实现：**

```go
func (c *Cond) Broadcast() {
    // 唤醒等待队列中的所有 waiter
    runtime_notifyListNotifyAll(&c.notify.list)
}
```

### 3. Go 1.26 Breaking Change

**变更内容：** `Broadcast()` 调用之前加入等待队列的 waiter 才能被唤醒，在 `Broadcast()` 之后才加入队列的 waiter **不会被唤醒**。

**变更原因：** 消除虚假唤醒的一种极端情况——"惊群效应"（Thundering Herd）。在旧版实现中，`Broadcast()` 可能唤醒在调用之后才执行 `Wait()` 的 goroutine（因为底层链表操作时序问题），导致这些 goroutine 的 Wait 调用实际上是"无效的"。Go 1.26 修复了这个 bug。

**旧版（Go ≤ 1.25）的坑：**

```go
// goroutine A（等待者）
go func() {
    mu.Lock()
    cond.Wait()  // 在某个时刻加入等待队列
    println("A 被唤醒")  // 可能被后续的 Broadcast 错误唤醒
    mu.Unlock()
}()

// goroutine B（广播者）
go func() {
    mu.Lock()
    cond.Broadcast()  // 唤醒所有当前等待者
    mu.Unlock()
}()
```

**新版（Go 1.26+）的行为：**

```go
// goroutine B（广播者）
go func() {
    mu.Lock()
    cond.Broadcast()  // 只唤醒调用 Broadcast 之前已加入队列的 waiter
    mu.Unlock()
}()

// goroutine A（等待者）
go func() {
    mu.Lock()
    cond.Wait()  // 在 Broadcast 之后才加入 → 不会被唤醒，必须等下一次
    mu.Unlock()
}()
```

**实际影响：** 这个变更影响的是"在 `Broadcast` 调用的同时，有 goroutine 刚调用 `Wait()` 还未加入队列"这种竞争情况。在生产代码中，如果你的等待逻辑本身是正确的（条件检查在循环内），基本不受影响。

**面试加分话术：**

> Go 1.26 修复了 Cond 的"惊群"问题，消除了虚假唤醒的最后一种边缘情况。理解这个变更可以帮助我们更准确地使用条件变量——关键是始终在 for 循环内 Wait，而不是 if，因为任何条件变量的语义都允许虚假唤醒。

### 4. 生产使用场景

**场景一：连接池可用连接通知**

```go
type Pool struct {
    mu    sync.Mutex
        cond  sync.Cond
    conns []*Conn
    max   int
}

func (p *Pool) Get(timeout time.Duration) (*Conn, error) {
    deadline := time.Now().Add(timeout)
    p.mu.Lock()
    defer p.mu.Unlock()

    for len(p.conns) == 0 {
        if time.Now().After(deadline) {
            return nil, errors.New("timeout waiting for connection")
        }
        p.cond.Wait()  // 等待有连接归还
    }

    conn := p.conns[0]
    p.conns = p.conns[1:]
    return conn, nil
}

func (p *Pool) Put(conn *Conn) {
    p.mu.Lock()
    p.conns = append(p.conns, conn)
    p.cond.Signal()  // 唤醒一个等待者
    p.mu.Unlock()
}
```

**场景二：任务队列消费者唤醒**

```go
type TaskQueue struct {
    mu    sync.Mutex
    cond  sync.Cond
    tasks []func()
    closed bool
}

func (q *TaskQueue) Add(task func()) {
    q.mu.Lock()
    defer q.mu.Unlock()
    q.tasks = append(q.tasks, task)
    q.cond.Signal()  // 唤醒一个消费者
}

func (q *TaskQueue) Get() (func(), bool) {
    q.mu.Lock()
    defer q.mu.Unlock()

    for len(q.tasks) == 0 && !q.closed {
        q.cond.Wait()  // 等待任务
    }

    if q.closed {
        return nil, false
    }

    task := q.tasks[0]
    q.tasks = q.tasks[1:]
    return task, true
}

func (q *TaskQueue) Close() {
    q.mu.Lock()
    defer q.mu.Unlock()
    q.closed = true
    q.cond.Broadcast()  // 唤醒所有等待者（让它们看到 closed=true）
}
```

### 5. vs channel 选型

| 场景 | 推荐 | 原因 |
|------|------|------|
| 简单事件通知（一个人等）| `chan` | 简单直接，无锁开销 |
| 广播通知（多人等同一事件）| `sync.Cond.Broadcast` | chan 需要 N 次读取才能通知 N 个人 |
| 等待某个状态变化 | `sync.Cond` | 原子释放锁+阻塞，避免忙等 |
| 跨 goroutine 传递数据 | `chan` | Cond 只传递信号，不传数据 |

---

## 高频追问

**Q：为什么 Cond.Wait() 必须传入一个锁？**

→ `Wait()` 的语义是"原子的释放锁+阻塞"，这两步必须原子完成，否则中间有时间窗口让 Signal 丢失。传入锁让 runtime 保证原子性。

**Q：Signal 和 Broadcast 有什么区别？**

→ `Signal` 唤醒等待队列中的**一个** goroutine（通常是最早等待的），`Broadcast` 唤醒**所有**等待者。Broadcast 的典型用法是"关闭"信号，通知所有等待者退出。

**Q：Cond 和 channel 的 `for range ch` 比，优劣在哪？**

→ Cond 更灵活：可以同时等待多个条件（通过在 Wait 返回后检查不同谓词），可以 Broadcast 一对多。Channel 更简单：适合单一消费者场景，不需要手动管理锁。

---

## 延伸阅读

- [Go sync.Cond 源码](https://github.com/golang/go/blob/master/src/sync/cond.go)
- [Go 1.26 Release Notes - sync](https://go.dev/doc/go1.26#sync)
- [Spurious Wakeup (Wikipedia)](https://en.wikipedia.org/wiki/Spurious_wakeup)
- [Avoiding锁竞争的条件变量模式](https://pkg.go.dev/sync#Cond)

---

**[← 上一篇：sync 原语](./02-sync.md)** · **[下一篇：atomic 与无锁 →](./03-atomic.md)**
