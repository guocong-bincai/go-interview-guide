# Go 标准库进阶：sync包、context、net/http 生产实践

> 考察频率：★★★★★  优先级：P0

## 面试官考察意图

这道题是 Go 高级工程师面试的**重灾区**。面试官不只是问"会用"，而是看你**是否踩过坑**——并发安全陷阱、goroutine 泄漏、连接池耗尽、请求取消不生效等真实问题。能把这些讲清楚的人，才是真正的资深工程师。

---

## 🔥🔥🔥🔥🔥 题目一：SingleFlight 解决缓存击穿（高频必考）

### 核心答案

`sync/singleflight.Group.Do` 的核心作用是：**对同一个 key 的并发调用，只执行一次函数，其余等待并共享结果**。它完美解决了缓存击穿的经典场景——大量并发请求同时查询不到缓存，导致全部穿透到数据库。

### 详细解析

```go
package main

import (
    "fmt"
    "sync"
    "sync/singleflight"  // Go 1.24+ 引入（实验性），或 golang.org/x/sync/singleflight
    "time"
)

var cache = make(map[string]string)
var group singleflight.Group

func getValue(key string) (string, error) {
    // Do 方法签名：Do(key string, fn func() (interface{}, error)) (interface{}, error, bool)
    // shared=true 表示这是等待者（复用了其他人的结果），shared=false 表示你是真正执行的那个
    v, err, shared := group.Do(key, func() (interface{}, error) {
        // ===== 进入到这里说明你是唯一的执行者 =====
        
        // 第一步：二次检查（有人在你之前已经写入了缓存）
        if val, ok := cache[key]; ok {
            return val, nil
        }
        
        // 第二步：模拟昂贵的操作（查数据库 / 调用下游服务）
        time.Sleep(2 * time.Second)
        result := fmt.Sprintf("result-for-%s", key)
        
        // 第三步：写入缓存
        cache[key] = result
        
        return result, nil
    })
    
    if err != nil {
        return "", err
    }
    
    result := v.(string)
    if shared {
        fmt.Printf("[%s] 命中 SingleFlight 共享结果（不是唯一执行者）\n", key)
    } else {
        fmt.Printf("[%s] 我是唯一执行者\n", key)
    }
    return result, nil
}

func main() {
    var wg sync.WaitGroup
    
    // 模拟 10 个并发请求查询同一个 key
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            val, _ := getValue("user:1001")
            fmt.Printf("Goroutine %d 拿到结果: %s\n", id, val)
        }(i)
    }
    
    wg.Wait()
    // 输出：所有 10 个 goroutine 都拿到了结果
    // 但 "result-for-user:1001" 实际上只被计算了 1 次！
}
```

### 面试话术

> "SingleFlight 在 Redis 预热、大数据聚合等场景中非常有用。它的本质是用一个 Group 实例按 key 合并重复调用，避免缓存击穿导致的雪崩效应。关键是要理解 `shared` 参数的含义——它告诉调用者自己是不是真正执行了那个耗时函数。"

### 追问扩展

**Q: SingleFlight 和 sync.Once 有什么区别？**

| 维度 | sync.Once | SingleFlight |
|------|-----------|--------------|
| 适用场景 | 全局只做一次的初始化（如单例创建） | 可重复执行的缓存/去重（每次key不同都会重新执行） |
| 参数化 | 无参数，固定函数 | 有 key 参数，不同 key 独立执行 |
| 返回值 | 无 | 返回 fn 的结果 |
| 多次调用 | 永远只执行一次 fn | 每个 key 的首次调用会执行 fn |

**Q: SingleFlight 底层是怎么实现的？**

1. 内部维护一个 map[string]*call，key → 正在执行的 call 对象
2. 如果 key 不存在于 map 中，创建新 call 并执行 fn
3. 如果 key 已存在，当前 goroutine 加入该 call 的 wait 列表并阻塞
4. fn 执行完成后，将结果写入 call.result，唤醒所有等待者
5. 之后从 map 中移除该 key（防止内存泄漏；如果需要保留结果可以设置 `Group.Set()`）

---

## 🔥🔥🔥🔥🔥 题目二：nil channel vs closed channel 语义差异（经典陷阱）

### 核心答案

Go 中对 **nil channel** 和 **closed channel** 的操作行为完全不同，混淆两者是线上 goroutine 永久阻塞的常见原因。核心规则只有一条：

- **读 nil channel → 永久阻塞**
- **写 nil channel → 永久阻塞**
- **读 closed channel → 立即返回零值**
- **写 closed channel → panic!**

### 详细解析

```go
package main

import (
    "fmt"
)

func main() {
    // ===== 场景对比 =====
    
    var nilCh chan int  // nil channel（未初始化的 channel）
    ch := make(chan int)
    close(ch)           // closed channel
    
    // 情况1: 读 nil channel —— 永久阻塞！
    // fmt.Println(<-nilCh) // 这里会永远等下去...
    
    // 情况2: 读 closed channel —— 返回零值
    val, ok := <-ch
    fmt.Printf("closed channel read: val=%d, ok=%v\n", val, ok)
    // 输出: closed channel read: val=0, ok=false
    
    // 情况3: 第二次读 closed channel —— 依然返回零值（不会panic）
    val, ok = <-ch
    fmt.Printf("second read: val=%d, ok=%v\n", val, ok)
    // 输出: second read: val=0, ok=false
    
    // 情况4: 写 closed channel —— panic!
    // ch <- 42 // fatal error: send on closed channel
    
    // ===== 实用模式：用关闭 channel 通知退出 =====
    done := make(chan struct{})
    go func() {
        doWork()
        close(done) // 通知消费者：工作已完成
    }()
    
    select {
    case <-done:
        fmt.Println("收到完成信号")
    case <-time.After(time.Second):
        fmt.Println("超时")
    }
}
```

### 实战：优雅停止 worker pool

```go
// 利用 channel 的关闭来优雅停止 goroutine
type WorkerPool struct {
    jobs   chan string
    done   chan struct{}
    cancel context.CancelFunc
}

func NewWorkerPool(ctx context.Context) *WorkerPool {
    _, cancel := context.WithCancel(ctx)
    return &WorkerPool{
        jobs:   make(chan string, 100),
        done:   make(chan struct{}),
        cancel: cancel,
    }
}

func (wp *WorkerPool) Start(n int) {
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case job, ok := <-wp.jobs:
                    if !ok {
                        // ✅ 正确：channel 关闭后，range 会正常退出
                        fmt.Println("worker 退出")
                        return
                    }
                    process(job)
                case <-wp.done:
                    // ✅ 双重保护：也监听 done channel
                    fmt.Println("worker 收到 cancel")
                    return
                }
            }
        }()
    }
    
    // 主流程：停止时先关闭 jobs channel，再关闭 done channel
    wp.Stop = func() {
        close(wp.jobs)  // 先关 jobs，触发 range 退出
        close(wp.done)  // 再关 done，触发 select 退出
        wg.Wait()       // 等所有 worker 结束
    }
}
```

### 面试话术

> "nil channel 和 closed channel 是最基础的 Go 并发常识，但恰恰是这种基础最容易在线上出问题。nil channel 读写都永久阻塞，closed channel 读取返回零值和 false、写入直接 panic。实践中常用 close(channel) 来发送终止信号，配合 range 或 select 实现优雅退出。"

### 追问：select 中 nil channel 的行为？

```go
func testSelect() {
    var nilCh <-chan int
    ch := make(chan int, 1)
    ch <- 1
    close(ch)
    
    select {
    case <-nilCh:
        fmt.Println("from nilCh")  // 永远不会到这里
    case val, ok := <-ch:
        fmt.Printf("from ch: val=%d, ok=%v\n", val, ok)  // 会打印
    default:
        fmt.Println("default")  // 不会执行，因为 ch 是有数据的
    }
}
// 输出: from ch: val=1, ok=true

// 如果 ch 也是 nil 或 closed 且无数据，就会走到 default
```

---

## 🔥🔥🔥🔥🔥 题目三：defer + named return 执行顺序（细节决定成败）

### 核心答案

defer 的执行时机和变量的作用域常常被人忽视。当 return 语句使用了**命名返回值**时，return 分为两步：**先赋值给命名返回值变量，再执行 defer，最后才从函数体返回**。

### 详细解析

```go
package main

import "fmt"

// ===== 场景1：return x; 没有命名返回值 =====
func simpleReturn() int {
    x := 0
    defer func() { x++ }()  // defer 修改的是局部变量 x
    return x                 // 直接返回 x 的值（此时 x=0）
}
// 返回值: 0

// ===== 场景2：return; 使用命名返回值 =====
func namedReturn() (r int) {
    r = 0
    defer func() { r++ }()  // defer 修改的是命名返回值 r
    return                  // 先执行 return r (r=0)，然后 defer r++, 最终返回 1
}
// 返回值: 1

// ===== 场景3：带标签的 return 语句 =====
func taggedReturn() (x int) {
    defer func() { fmt.Printf("defer 时 x=%d\n", x) }()
    goto tag               // 跳转到 label
tag:
    x = 5
    return x               // 等价于: x = 5; return （但实际是 return x，即 return 5）
    // defer 执行时 x=5，最终返回 5
}
// 输出: defer 时 x=5
// 返回值: 5

// ===== 最经典的面试坑 =====
func tricky() (v int) {
    defer func() { v++ }()
    return 0
}
func main() {
    fmt.Println(tricky())  // 输出: 1
}
```

**执行流程图：**

```
func f() (x int) {
    x = 10
    
    // 遇到 return x
    step1: 赋值 x = x          ← x 已经是 10
    step2: 执行 defers         ← 此时 x=10
    step3: 返回 x              ← 返回 10
}

vs

func g() (x int) {
    x = 10
    defer func() { x++ }()     ← defer 捕获的是 x 引用
    
    return x                    // step1→2→3: 
    // step1: x = x (10)
    // step2: defer 执行, x++ → x 变成 11
    // step3: 返回 x → 返回 11！
}
```

### 面试话术

> "defer 是在 return 之后才执行的，但 return 如果是 return expr 形式，expr 会先求值存入命名返回值变量，然后再执行 defer，最后函数返回。所以 defer 里可以直接修改命名返回值来实现'兜底'效果。如果不放心，可以用匿名函数返回，避免 deferred 修改影响返回值。"

### 最佳实践

```go
// ❌ 风险：defer 意外修改了返回值
func parseValue(data []byte) (value int, err error) {
    value = parseInt(data)
    defer func() {
        if err != nil {
            value = 0  // 出了错就归零 —— 虽然意图好，但不明确
        }
    }()
    return
}

// ✅ 推荐：defer 只用于资源清理，不影响返回值
func parseValue(data []byte) (int, error) {
    buf := allocateBuffer()
    defer release(buf)  // 只负责清理
    
    value, err := parseInt(data)
    if err != nil {
        return 0, err  // 明确的 error return
    }
    return value, nil
}
```

---

## 🔥🔥🔥🔥 题目四：context.Context 传播与超时控制（微服务必备）

### 核心答案

Context 在 Go 中的核心职责是**跨 API 边界传递上下文信息**（超时、取消、请求追踪），而不是用来传递业务参数。它的传播链必须完整：入口层 → 业务层 → 数据访问层，任何一环断裂都可能造成 goroutine 泄漏或级联超时。

### 详细解析

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// ===== 1. 基本传播链 =====
func handleRequest(ctx context.Context) error {
    // 传给子函数
    if err := processQuery(ctx); err != nil {
        return fmt.Errorf("处理请求失败: %w", err)
    }
    return nil
}

func processQuery(ctx context.Context) error {
    // 继续传给 DB 层
    return queryDB(ctx)
}

func queryDB(ctx context.Context) error {
    // context 传给 db.QueryRowContext
    row := db.QueryRowContext(ctx, "SELECT ...")
    var val string
    if err := row.Scan(&val); err != nil {
        return err
    }
    return nil
}

// ===== 2. Context 树构建（入口点）=====
func httpHandler(w http.ResponseWriter, r *http.Request) {
    // 创建根 context（通常带 deadline）
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()  // ⚠️ 重要！没有这个，超时 timer 不会释放
    
    // 附加请求级别的信息
    ctx = context.WithValue(ctx, "requestID", generateReqID())
    ctx = context.WithValue(ctx, "userID", extractUserID(r))
    
    err := handleRequest(ctx)
    if err != nil {
        // 区分取消和超时
        if errors.Is(err, context.Canceled) {
            // 客户端断开连接
        }
        if errors.Is(err, context.DeadlineExceeded) {
            // 超时，可能重试
        }
    }
}

// ===== 3. WithValue 的正确用法 =====
// ✅ 推荐：只传 request-scoped 数据
ctx := context.WithValue(parentCtx, "requestID", reqID)

// ❌ 禁止：不要用它传配置或数据库连接
// ctx := context.WithValue(ctx, "db", dbClient)  // 破坏封装
// ctx := context.WithValue(ctx, "config", config)  // 应该用依赖注入

// ===== 4. 避免频繁创建 Context =====
func badApproach(items []string) {
    for _, item := range items {
        // ❌ 每次都创建新的 WithCancel，产生大量 goroutine/timer
        ctx, cancel := context.WithCancel(context.Background())
        doSomething(ctx)
        cancel()
    }
}

func goodApproach(items []string) {
    // ✅ 复用同一个 parent context，只在需要时加一层
    ctx := context.Background()
    for _, item := range items {
        if shouldTimeOut(item) {
            ctxWithTimeout, cancel := context.WithTimeout(ctx, time.Second)
            doSomething(ctxWithTimeout)
            cancel()
        } else {
            doSomething(ctx)
        }
    }
}
```

### 面试话术

> "Context 的传播就像接力赛——每一环都必须把上一环的 context 传给自己创建的 child context。最关键的三个点是：一是入口处用 WithTimeout 设 deadline，二是每一个 WithCancel/WithTimeout 都要配对一个 cancel，三是 Value 只能传 request-scoped 元数据不能放重型对象。最常见的线上故障就是 cancel 缺失导致 goroutine 无限等待。"

### 追问：WithCancel 和 WithTimeout 的区别？

| 特性 | WithCancel | WithTimeout | WithDeadline |
|------|-----------|-------------|-------------|
| 取消方式 | 手动调用 cancel() | 超时自动取消 | 指定绝对截止时间 |
| 底层实现 | Done channel 由用户关闭 | 基于 Timer 自动调 cancel | 基于 Timer 自动调 cancel |
| 适用场景 | 知道何时该停的场景 | 已知需要等多久 | 知道具体截止时间 |
| 资源释放 | cancel() 后 Timer 立即回收 | Timer 到期自动回收 | Timer 到期自动回收 |

---

## 🔥🔥🔥🔥 题目五：errors.Is / As / Unwrap — Go 错误处理的最佳实践

### 核心答案

Go 1.13+ 引入了标准的错误包装机制：`fmt.Errorf("%w", err)` 包装错误、`errors.Is()` 判断错误类型、`errors.As()` 提取错误详情、`Unwrap()` 遍历错误链。这取代了之前的"魔数比较"和 `if err.Error() == "...")` 反模式。

### 详细解析

```go
package main

import (
    "errors"
    "fmt"
    "os"
)

// ===== 1. 定义业务错误 =====
var (
    ErrUserNotFound = errors.New("user not found")
    ErrPaymentFailed = errors.New("payment processing failed")
)

// 带详情的业务错误
type ValidationErr struct {
    Field string
    Msg   string
}

func (e *ValidationErr) Error() string {
    return fmt.Sprintf("validation error on field '%s': %s", e.Field, e.Msg)
}

// 实现 Is 方法，支持 errors.Is 匹配
func (e *ValidationErr) Is(target error) bool {
    targetV, ok := target.(*ValidationErr)
    return ok && e.Field == targetV.Field
}

// ===== 2. 错误包装（错误链）=====
func createUser(name string) error {
    if name == "" {
        return fmt.Errorf("create user: %w", 
            &ValidationErr{Field: "name", Msg: "cannot be empty"})
    }
    return nil
}

// ===== 3. 判断错误类型 =====
func handleCreateUser(name string) {
    err := createUser(name)
    
    // ✅ 推荐：用 errors.Is 判断（支持错误链遍历）
    if errors.Is(err, ErrUserNotFound) {
        // 找到特定错误
    }
    
    // ✅ 推荐：用 errors.As 提取详细信息
    var valErr *ValidationErr
    if errors.As(err, &valErr) {
        fmt.Printf("字段 %s: %s\n", valErr.Field, valErr.Msg)
        // 输出: 字段 name: cannot be empty
    }
    
    // ❌ 反模式：不要用这个
    // if err.Error() == "validation error on field..."  // fragile!
    // if err == ErrXXX  // 只有未包装的错误才能这样比较
}

// ===== 4. 实际场景：分层错误包装 =====
func OrderService_CreateOrder(ctx context.Context, order *Order) (*Order, error) {
    // 第1层：业务校验
    if err := validateOrder(order); err != nil {
        return nil, fmt.Errorf("order service: validate: %w", err)
    }
    
    // 第2层：调用库存服务
    inventory, err := InventoryService_CheckStock(ctx, order.Items)
    if err != nil {
        return nil, fmt.Errorf("order service: check stock: %w", err)
    }
    
    // 第3层：扣减库存
    if err := InventoryService_Deduct(ctx, inventory.ID); err != nil {
        return nil, fmt.Errorf("order service: deduct stock: %w", err)
    }
    
    return order, nil
}

// ===== 5. 调用方解析错误链 =====
func handleError(err error) {
    // 遍历整个错误链
    rootCause := unwrapToRoot(err)
    fmt.Printf("根因: %v\n", rootCause)
}

func unwrapToRoot(err error) error {
    for err != nil {
        unwrapped := errors.Unwrap(err)
        if unwrapped == nil {
            return err
        }
        err = unwrapped
    }
    return err
}
```

### 面试话术

> "Go 1.13 的错误处理进化是重大改进。核心原则是：上游用 `%w` 包装错误形成链，下游用 `errors.Is` 做类型匹配、用 `errors.As` 提取详情。多层服务之间的错误通过 `fmt.Errorf("service_x: %w", upstreamErr)" 向上包装，每层保留自己的上下文。判断错误永远用 `Is`，需要拿详情用 `As`，不要在中间层判断太细粒度的错误——让上层来决定如何处理。"

---

## 🔥🔥🔥🔥 题目六：strings.Builder 替代字符串拼接（性能题）

### 核心答案

`strings.Builder` 是 Go 1.10+ 引入的高效字符串构建器。其核心优势在于**按需扩容、支持零拷贝输出**，避免了 `+=` 和 `fmt.Sprintf` 在循环中的大量临时对象分配。对于 100+ 次的字符串拼接，性能提升可达 10-50 倍。

### 详细解析

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // ===== 方案对比 =====
    
    // ❌ 反模式： += 在循环中
    result := ""
    for i := 0; i < 1000; i++ {
        result += fmt.Sprintf("item%d ", i)
        // 每次 += 都创建新的 []byte + 复制 + GC
        // O(n²) 复杂度！
    }
    
    // ✅ 推荐：strings.Builder
    var sb strings.Builder
    sb.Grow(10000)  // 预分配容量，避免反复扩容
    
    for i := 0; i < 1000; i++ {
        sb.WriteString(fmt.Sprintf("item%d ", i))
        // O(1) 追加，总复杂度 O(n)
    }
    result = sb.String()
    
    // 🔥 高级技巧：StringBuilder + Reset 复用
    var reusedBuilder strings.Builder
    reuseLoop := func(count int) {
        reusedBuilder.Reset()  // 清零复用
        for i := 0; i < count; i++ {
            reusedBuilder.WriteString("data ")
        }
        _ = reusedBuilder.String()
    }
    for iter := 0; iter < 100; iter++ {
        reuseLoop(iter * 100)  // 复用同一个 builder
    }
}

// ===== Benchmark 对比 =====
// strings.Join([]string{"a","b","c"}, "-") → 最快（元素数量确定时）
// strings.Builder + String()          → 适合动态追加
// "+" 运算符                          → O(n²)，小量可用，大量禁用
```

### 面试话术

> "字符串拼接的性能陷阱几乎每个 Go 开发者都踩过。核心要点是：循环拼接用 strings.Builder 并预分配 Grow、元素确定的数组用 strings.Join、少量拼接（<=5次）直接用 +。关键是知道 '为什么'——+ 运算符每次创建新切片是 O(n²) 复杂度，而 Builder 是摊销 O(1)。"

### 追问：strings.Builder.String() 是否会 copy？

- **`sb.String()`**：会做一次 memcpy，返回新的 string
- **`sb.String()` + 立即丢弃**：Go GC 能优化掉
- **如果你不想做任何 copy**：用 `bytes.Buffer` 的 `Bytes()` + string conversion（但要注意生命周期）
- **Go 1.22+**：`sb.AppendString()` 等方法进一步减少了中间对象的生成

---

## 🔥🔥🔥 题目七：http.Client 连接池管理与超时配置（线上故障高频）

### 核心答案

默认的 `http.DefaultClient` **不推荐在生产环境直接使用**——它的超时默认是无限的（`Timeout=0`），连接池也没有合理的上限配置。正确的做法是每个客户端显式配置 `Transport`（连接池参数）、`Timeout`（总超时）和 `IdleConnTimeout`（空闲连接回收）。

### 详细解析

```go
package main

import (
    "crypto/tls"
    "net"
    "net/http"
    "time"
)

// ===== 生产级 http.Client 配置 =====
func newHTTPClient() *http.Client {
    transport := &http.Transport{
        // 连接池配置（关键！）
        MaxIdleConns:        100,      // 最大空闲连接数
        MaxIdleConnsPerHost: 100,      // 每个 host 的最大空闲连接数
        IdleConnTimeout:     90 * time.Second,  // 空闲连接存活时间
        
        // TCP 连接配置
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,   // TCP 握手超时
            KeepAlive: 30 * time.Second,  // TCP KeepAlive 间隔
        }).DialContext,
        
        // TLS 配置
        TLSHandshakeTimeout: 5 * time.Second,
        
        // 请求头等待响应超时（不包括 body 读取）
        ExpectContinueTimeout: 1 * time.Second,
    }
    
    return &http.Client{
        Transport: transport,
        Timeout:   10 * time.Second,    // ⚠️ 必须有！否则会永久挂起
    }
}

// ===== 常见线上故障模式 =====
/*
故障1: DefaultClient 不配置超时 → 某个下游挂掉，所有关联请求永久挂住
  现象: goroutine 泄漏、CPU 飙升、连接耗尽
  
故障2: MaxIdleConnsPerHost 过小（默认10）→ 频繁建立新 TCP 连接
  现象: P99 延迟高、TLS 握手开销大
  
故障3: IdleConnTimeout 为0（无限期）→ 即使下游重启，旧连接持续报错
  现象: 偶发 connection reset by peer
*/

// ===== 配套中间件：请求日志 & 超时统计 =====
func withTimeoutMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // 如果有父 context，用它的 deadline
        deadline, ok := r.Context().Deadline()
        if !ok {
            // 设置默认超时
            ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
            defer cancel()
            r = r.WithContext(ctx)
        }
        
        wrapped := &timeoutResponseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        next.ServeHTTP(wrapped, r)
        
        duration := time.Since(start)
        if duration > 5*time.Second {
            log.Warn("slow request", "method", r.Method, "path", r.URL.Path, 
                     "duration_ms", duration.Milliseconds(), 
                     "status", wrapped.statusCode)
        }
    })
}

type timeoutResponseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (tw *timeoutResponseWriter) WriteHeader(code int) {
    tw.statusCode = code
    tw.ResponseWriter.WriteHeader(code)
}
```

### 面试话术

> "DefaultClient 是个定时炸弹，因为它没设超时。生产环境的 Client 至少要配：Transport 的连接池大小和空闲回收、TCP 握手超时、整体请求超时。常见的线上事故都是 Client 超时缺失引发的——单个慢请求拖垮整个线程池。配置不当还有另一类问题：MaxIdleConnsPerHost 太小会导致频繁 TLS 握手，太大则浪费文件描述符资源。"

---

## 高频追问汇总

**Q1: sync.Pool 里的对象什么时候被 GC？**

GC 周期内随时可能被回收！所以不能把持久化数据放进 Pool。典型误用：把一个 long-lived 的 slice 放入 Pool，下次取出时发现内容被清除了。正确用法：只存可重新创建的临时 buffer。

**Q2: os.Open 返回的文件实现了哪些接口？**

`os.File` 实现了 `io.Reader`、`io.Writer`、`io.Closer`、`io.Seeker`、`io.ReaderAt`、`io.WriterAt`、`io.ByteScanner`、`io.RuneScanner`、`io.WriterTo`（零拷贝优化）。其中 `WriteTo` 是关键，标准库 http.Server 在发送文件时会优先调用它走 sendfile 零拷贝路径。

**Q3: context.WithValue 和结构体字段有什么取舍？**

WithValue 适合传 request-scoped 元数据（traceID、用户ID、请求开始时间），不应传业务对象或配置。如果多个地方都需要某些数据，更推荐使用结构化设计（传入 struct/context）而非塞到 Value map 里，这样更安全、可发现性更好。

**Q4: 如何 debug 死锁的 goroutine？**

1. `runtime.Stack(true, &buf)` 获取所有 goroutine 堆栈
2. pprof `go tool pprof -http=:8080 http://localhost:6060/debug/pprof/goroutine?debug=1`
3. 看是否有 goroutine 在 channel recv/send 处 hang 住
4. 最常见原因：单向 channel 方向搞反、close 了还在写、goroutine 启动后无人消费

---

## 延伸阅读

- [sync.singleflight 文档](https://pkg.go.dev/sync/singleflight)
- [context 文档](https://pkg.go.dev/context)
- [error 最佳实践 (Rob Pike)](https://go.dev/blog/go1.13-errors)
- [http.Transport 配置指南](https://www.baeldung.com/golang-http-client-config)
- [strings.Builder 源码解读](https://github.com/golang/go/blob/master/src/strings/builder.go)

---

**[← 上一篇：io 包深度解析](./02-02-io-reader-writer-deep.md)** · **[返回目录](../README.md)**
