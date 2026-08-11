[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [💬 语言机制](../README.md)

---

# nil Channel vs Closed Channel：发送和接收的差异

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：nil channel、close(channel)、零值阻塞、ok 判断、select default、goroutine 泄漏防护

## 🎯 面试官考察意图

Channel 是 Go 并发的核心，但 nil channel 和已关闭 channel 的行为区别是高频面试陷阱。面试官想确认：

1. 对 nil channel 收发行为的理解（永远阻塞）
2. 对已关闭 channel 收发行为的理解（读返回零值+false，写 panic）
3. 能否用 `_, ok := <-ch` 正确判断 channel 是否关闭
4. 能否写出安全的 goroutine 生命周期管理代码

---

## ⚡ 核心答案（30秒版）

**nil channel 的发送和接收永远阻塞；已关闭 channel 发送会 panic、读取返回零值和 false。**

```go
var ch chan int      // nil channel ← 读写都永久阻塞
closedCh := make(chan int); close(closedCh) ← 读 → 0, false; 写 → panic
```

生产中最常用的模式：**发送方负责关闭 channel**，接收方用 `for range ch` 或 `_, ok := <-ch` 感知关闭并退出循环。

---

## 🔬 深度展开

### 1. 四种组合行为总表

| 操作 | nil channel | 未关闭 channel | 已关闭 channel |
|------|-------------|----------------|----------------|
| `ch <- v`（发送） | 永久阻塞 💤 | 正常发送 | **panic: send on closed channel** 💥 |
| `<-ch`（接收） | 永久阻塞 💤 | 正常接收 | 立即返回 `(零值, false)` ✅ |
| `range ch` | 无意义（永远不会执行） | 依次遍历直到关闭 | 立即结束（不会 panic）✅ |
| `select { case <-ch }`（无 default） | 永久阻塞 💤 | 等数据 | 继续等（读到零值）→ 可能逻辑错误 ⚠️ |
| `select { case <-ch: ...; default }` | 走到 default ✅ | 有数据走case，没走default | 走到 default ✅ |

### 2. 典型场景与最佳实践

#### 场景一：生产者-消费者模型

```go
func producer(data []int, ch chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for _, v := range data {
        ch <- v // 发送完毕后...
    }
    close(ch) // ✅ 发送方负责关闭 channel
}

func consumer(ch <-chan int) {
    for v := range ch { // ✅ range 自动在 channel 关闭后退出
        process(v)
    }
}
```

#### 场景二：安全检测 channel 是否关闭

```go
// ❌ 常见错误：直接检查变量是否为 nil（语义不清晰）
if ch == nil { ... }

// ✅ 推荐：用 select + default 模式检测
select {
case v, ok := <-ch:
    if !ok {
        fmt.Println("channel closed")
    } else {
        fmt.Printf("got value: %d\n", v)
    }
default:
    fmt.Println("no data available (but might not be closed)")
}
```

关键理解：**select 里的 default 分支表示"如果此时没数据可读就走这里"**，它不关心 channel 是否关闭，只关心当前是否有数据可取。

#### 场景三：goroutine 泄漏防护

```go
// ❌ 危险：消费者收到关闭信号但没有及时处理，导致上游持续写入 panic
func worker(ctx context.Context, ch <-chan int, results chan<- Result) {
    go func() {
        for v := range ch {
            res := compute(v)
            results <- res // 💥 如果 results 满了且没人消费，goroutine 永久阻塞！
        }
    }()
    
    <-ctx.Done() // 等待取消
}

// ✅ 正确：所有输出都用 context 感知
func workerSafe(ctx context.Context, ch <-chan int, results chan<- Result) {
    go func() {
        defer close(results) // 确保最终关闭
        for {
            select {
            case v, ok := <-ch:
                if !ok {
                    return // channel 关闭，退出
                }
                res := compute(v)
                select {
                case results <- res: // 带上下文感知的发送
                case <-ctx.Done():
                    return
                }
            case <-ctx.Done():
                return
            }
        }
    }()
}
```

### 3. nil channel 的实际用途

nil channel 不是 BUG，有时是有意的用法：

```go
// 用途1：惰性初始化（配合 sync.Once 更安全）
type Connection struct {
    ch     chan<- Command
    once   sync.Once
}

func (c *Connection) Send(cmd Command) {
    c.once.Do(func() {
        c.ch = make(chan Command, 100)
        go c.worker()
    })
    if c.ch != nil {
        c.ch <- cmd
    }
    // 另一种写法：用 select default
    select {
    case c.ch <- cmd:
    default:
        log.Printf("buffer full, dropping command: %v", cmd)
    }
}

// 用途2：条件性禁用某个 channel
func multiSourceMerge(srcs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    
    for _, src := range srcs {
        wg.Add(1)
        go func(s <-chan int) {
            defer wg.Done()
            for v := range s {
                out <- v
            }
        }(src)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

// 传入 nil source 会被跳过（range nil channel 什么都不做）
// 所以可以动态过滤掉空的 source
```

⚠️ 注意：**`range nil channel` 永远不会进入循环体，因为它永远不会收到任何值**。这是一个常见的误用点。

### 4. close 的安全调用规则

```go
// ✅ 唯一安全的 close 时机：发送方确定不会再有任何数据写入时
close(ch)

// ❌ 绝对禁止：向已关闭的 channel 发送数据
func safeSend(ch chan<- int, v int) bool {
    defer func() {
        recover() // 吞掉可能的 panic
    }()
    ch <- v
    return true
}

// ❌ 绝对禁止：并发 close（多个 goroutine 同时 close 同一 channel → panic）
// ✅ 解决方案：用一个单独的 goroutine 负责 close
go func() {
    wg.Wait()
    close(ch)
}()
```

### 5. 实战题目（面试常考）

```go
func main() {
    ch := make(chan int, 1)
    ch <- 1
    
    select {
    case v := <-ch:
        fmt.Print(v) // 打印 1
    
    default:
        fmt.Print("default") // 不会走这里，因为 channel 里有数据
    }
}
// 输出：1

// ======

func main() {
    ch := make(chan int, 1)
    ch <- 1
    close(ch) // 先关再读
    
    select {
    case v, ok := <-ch:
        fmt.Printf("%d,%t\n", v, ok) // 1,true —— 还能读出最后一个值
    default:
        fmt.Println("empty")
    }
}
// 输出：1,true

// ======

func main() {
    var ch chan int // nil!
    select {
    case <-ch:
        fmt.Println("received") // 永远不会到这里
    case <-time.After(time.Second):
        fmt.Println("timeout")  // 一秒后超时
    }
}
// 输出：timeout
```

---

## 🗣️ 面试话术

- **初级**："nil channel 读写都阻塞，关闭的 channel 读返回零值和false，写会 panic。发送方负责关闭 channel。"
- **中级**："生产环境常用 `for range ch` 或 `_, ok := <-ch` 模式来处理 channel 关闭。nil channel 适合做惰性初始化和条件性禁用。多个 goroutine 不能并发 close 同一个 channel。"
- **高级**："nil channel 在 select 中会一直不被选中，可以利用这个特性实现条件分支。goroutine 泄漏的核心排查手段之一就是看是否有人在等待 nil channel。`range nil channel` 等价于死循环但不迭代——编译器会优化掉。"

---

## 🔗 关联阅读

- [Channel 底层原理](../02-concurrency/38-01-channel.md)
- [Goroutine 生命周期](../01-runtime/11-11-goroutine-lifecycle.md)
- [Select 深度解析](../02-concurrency/43-07-select.md)
