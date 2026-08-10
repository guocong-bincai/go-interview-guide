[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 并发编程](../README.md)

---

# context 原理：取消传播、超时控制与层级树

## 面试官考察意图

考察候选人对 Go context 包的深层理解。
初级只能说出"用来取消 goroutine"，高级要能讲清楚 **context 的树形取消传播机制、WithCancel/WithTimeout/WithDeadline 底层实现、以及在 HTTP 框架（gRPC/Gin）和数据库查询中的实际应用**，并能分析 context 值传递的安全性。

---

## 核心答案（30 秒版）

Go context 是**树形取消信号传播**机制：

| 函数 | 作用 | 适用场景 |
|------|------|----------|
| `context.Background()` | 根节点，不可取消 | HTTP Handler 入口 |
| `WithCancel(parent)` | 手动取消 | 主动取消一个操作树 |
| `WithTimeout(parent, d)` | 超时取消 | HTTP 请求超时、RPC 调用超时 |
| `WithDeadline(parent, t)` | 绝对时间截止 | DB 查询截止时间 |
| `WithValue(parent, k, v)` | 携带键值对 | 埋点 ID、用户信息传递 |

**核心语义：** 父 context 取消 → 所有子 context 全部取消（级联传播）。

---

## 深度展开

### 1. context 核心结构

```go
// src/context/context.go
type Context interface {
    Deadline() (deadline time.Time, ok bool)  // 截止时间
    Done() <-chan struct{}                    // 取消信号 channel
    Err() error                              // 取消原因
    Value(key any) any                       // 获取值
}
```

### 2. context 树形结构

```
context.Background()  ←─────────────── 根节点，永不取消
    │
    ├── WithCancel(ctx)  →  cancelCtx
    │       │
    │       └── WithTimeout(ctx, 3s) → timerCtx
    │               │
    │               └── WithValue(ctx, "traceID", "abc") → valueCtx
    │
    └── WithDeadline(ctx, time.Now().Add(5s))  → timerCtx
```

### 3. 四种 Context 实现

#### 3.1 emptyCtx：根节点

```go
type emptyCtx int

func (emptyCtx) Deadline() (time.Time, bool) { return time.Time{}, false }
func (emptyCtx) Done() <-chan struct{}       { return nil }
func (emptyCtx) Err() error                   { return nil }
func (emptyCtx) Value(key any) any            { return nil }

var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

// context.Background() = emptyCtx，永不取消
// context.TODO() = emptyCtx，用在未确定根节点时
```

#### 3.2 cancelCtx：可取消 context

```go
type cancelCtx struct {
    Context
    mu       sync.Mutex
    done     atomic.Pointer[chan struct{}]  // 延迟初始化
    children map[cancelCtx]struct{}          // 子 context
    err      error                          // 取消原因
}
```

**Done() 的延迟初始化（重要考点）：**

```go
func (c *cancelCtx) Done() <-chan struct{} {
    d := c.done.Load()
    if d != nil {
        return *d
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    d = c.done.Load()
    if d == nil {
        ch := make(chan struct{})
        c.done.Store(&ch)
        d = &ch
    }
    return *d
}
// 为什么不直接初始化？—— 避免创建永远不取消的 channel（节省资源）
```

**取消传播的树形删除：**

```go
func cancel(ctx Context, removeFromParent bool, err error) {
    c, ok := ctx.(*cancelCtx)
    if !ok {
        return
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // 已经取消过
    }
    c.err = err
    d, _ := c.done.Load()
    if d == nil {
        ch := make(chan struct{})
        c.done.Store(&ch)
        d = &ch
    }
    close(*d) // 广播取消信号

    // 递归取消所有子 context
    for child := range c.children {
        cancel(child, true, err)
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        // 从父节点删除自己
        if parent, ok := parentDesc{c.parent}; ok {
            parent.cancelCtx.mu.Lock()
            delete(parent.cancelCtx.children, c)
            parent.cancelCtx.mu.Unlock()
        }
    }
}
```

#### 3.3 timerCtx：超时 context

```go
type timerCtx struct {
    cancelCtx
    timer    *time.Timer
    deadline time.Time
}

func (c *timerCtx) Deadline() (time.Time, bool) {
    return c.deadline, true
}
```

`WithTimeout` 本质：`WithDeadline(parent, time.Now().Add(timeout))`

```go
// 源码
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```

**timerCtx 的双重取消：**

```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err)
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
    if removeFromParent {
        // 从父节点删除
    }
}
// timerCtx 取消时会：① 关闭 cancelCtx 的 Done channel，② 停止定时器
```

#### 3.4 valueCtx：键值传递

```go
type valueCtx struct {
    Context
    key, val any
}

func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    // 沿着 context 链向上查找
    return c.Context.Value(key)
}
```

**WithValue 的"不可覆盖"语义：**

```go
ctx := context.Background()
ctx = context.WithValue(ctx, "key", "original")
ctx = context.WithValue(ctx, "key", "override") // 不是覆盖！创建了新的 valueCtx

// ctx.Value("key") → "override" ← 这是覆盖的假象
// 实际上：子节点的 Value() 查到自己的 key 就返回，不会继续向上找
// 所以语义上是"遮蔽"，不是"覆盖"

func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val  // 找到就返回，不会继续问 parent
    }
    return c.Context.Value(key)  // 找不到才继续
}
```

### 4. 实际应用场景

#### 场景 1：gRPC 超时取消

```go
// 客户端调用
func callGRPC(ctx context.Context, req *pb.Request) (*pb.Response, error) {
    // ctx 携带超时，gRPC 自动在该截止时间取消
    return grpcClient.Method(ctx, req)
}

// 服务端
func (s *Server) Method(srv interface{}, ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err() // deadline exceeded 或 canceled
    default:
    }
    // 继续处理...
}
```

#### 场景 2：数据库查询超时

```go
func queryDB(ctx context.Context, sql string) ([]Row, error) {
    // 创建一个有超时限制的 context 给 DB
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    rows, err := db.QueryContext(ctx, sql)
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            // 查询超时，记录慢查询
        }
        return nil, err
    }
    return rows, nil
}
```

#### 场景 3：并发任务树取消

```go
func scrapeURLs(ctx context.Context, urls []string) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    errCh := make(chan error, len(urls))
    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            // 只要 ctx 取消（任意一个失败，或超时），所有 goroutine 都停止
            err := fetchURL(ctx, u)
            if err != nil {
                cancel() // 一个失败，取消所有
                errCh <- err
            }
        }(url)
    }

    wg.Wait()
    close(errCh)

    var errs []error
    for err := range errCh {
        errs = append(errs, err)
    }
    if len(errs) > 0 {
        return fmt.Errorf("%d errors: %v", len(errs), errs)
    }
    return nil
}
```

#### 场景 4：传递请求级别数据

```go
// 定义 context key（防止冲突
type contextKey string
const traceIDKey = contextKey("traceID")
const userIDKey = contextKey("userID")

// 服务端入口注入
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        traceID := r.Header.Get("X-Trace-ID")
        ctx := context.WithValue(r.Context(), traceIDKey, traceID)
        ctx = context.WithValue(ctx, userIDKey, getUserID(r))
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// 业务代码读取
func handler(w http.ResponseWriter, r *http.Request) {
    traceID := r.Context().Value(traceIDKey)
    logger.Info("request", "traceID", traceID)
}
```

### 5. 常见坑

#### 坑 1：context 泄漏

```go
// ❌ 错误：context 没有被消费，timer 永远不会被 GC
func bad() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Hour)
    defer cancel() // ✅ 有 defer，但是...

    // 如果提前 return，ctx 被 defer cancel
    // 但是！如果有 goroutine 持有 ctx 引用（比如传递给后台任务）
    // 外部的 ctx 被 cancel 了，那个后台任务仍然持有引用，但 timerCtx
    // 的 timer 可能不会被及时清理
}

// ✅ 正确：确保所有使用 ctx 的路径都能感知取消
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
go backgroundTask(ctx) // 明确传递 context
```

#### 坑 2：context 作为函数第一个参数的习惯

```go
// ✅ 标准做法：context 总是第一个参数
func QueryUser(ctx context.Context, userID int64) (*User, error) {
    return db.QueryContext(ctx, "SELECT ...")
}

// ❌ 不要这样做
func QueryUser(userID int64, ctx context.Context) (*User, error)

// ✅ 不需要 context 时，显式传 context.Background()
result := QueryUser(context.Background(), 123)
```

#### 坑 3：context.Value 的键类型冲突

```go
// ❌ 用 string 作为 key，容易冲突
ctx := context.WithValue(ctx, "userID", 123)
val := ctx.Value("userID") // 任何包可能都用 "userID"，冲突！

// ✅ 用 unexported 类型作为 key
type userIDKey struct{}
ctx := context.WithValue(ctx, userIDKey{}, 123)
val := ctx.Value(userIDKey{}) // 只有知道类型的代码才能访问
```

---

## 高频追问

**Q：context 的 Done() 返回 nil 意味着什么？**

对于 `context.Background()` 和 `context.TODO()`，`Done()` 返回 nil，因为它们永远不会被取消。对于其他 context，如果还没调用 cancel/超时/截止，`Done()` 第一次被调用时才创建 channel（延迟初始化）。

**Q：context 取消是同步还是异步的？**

取消操作（`cancel()`）本身是同步的：它立即关闭 `done` channel，所有阻塞在 `<-ctx.Done()` 上的 goroutine 会同时被唤醒（不需要等待）。这是通过关闭 channel 实现的，关闭 channel 会立即唤醒所有等待者。

**Q：context 和 go.uber.org/zap 的日志框架有什么关系？**

`zap` 支持从 context 提取字段：

```go
logger.Info("request",
    zap.Stringer("traceID", traceIDFromContext(ctx)),
    zap.Stringer("userID", userIDFromContext(ctx)),
)
```

**Q：WithCancel 和 WithTimeout 的 cancelFunc 有什么区别？**

两者返回的 `CancelFunc` 本质相同（都是 `func()`），区别是：
- `WithCancel`：手动取消，必须调用一次（幂等，多次调用是 no-op）
- `WithTimeout`：手动取消 **或** 自动超时取消（只需要调用一次，不要重复调用）

**Q：goroutine 使用已取消的 context 会怎样？**

已经创建的 goroutine 不会自动停止，只是它们持有的 context 的 `Done()` channel 会被关闭。goroutine 需要自己在适当的地方检查 `ctx.Done()` 或 `ctx.Err()` 来决定是否退出。

---

## 延伸阅读

- [Go context 官方博客：Go Concurrency Patterns: Context](https://go.dev/blog/context)
- [context 源码](https://github.com/golang/go/blob/master/src/context/context.go)（官方实现）
- [Best Practices for Context Values in Go](https://go.dev/blog/context-value-function)
- [gRPC Cancellation and Timeouts](https://grpc.io/docs/guides/cancellation/)

---

**[← 上一篇：并发模式](./04-patterns.md)**
