# Panic / Recover：异常处理的正确姿势

> 考察频率：★★★★★  难度：★★★☆☆
> 关键词：panic vs error、recover 时机、defer + recover、错误传播边界、Go哲学

## 🎯 面试官考察意图

Go 没有 try-catch，但有 panic/recover。面试官想确认候选人是否理解：

1. **Panic 不是 Exception**——它不是错误处理机制，而是"程序出事了，停掉！"的信号
2. **Recover 的正确使用时机**——在 goroutine 入口或 HTTP handler 顶层做兜底
3. **能否区分"recoverable"和"unrecoverable"错误**
4. **是否了解 panic 对 GC、内存、性能的影响**

---

## ⚡ 核心答案（30秒版）

**Go 的 panic/recover 不是用来处理业务错误的，而是用来处理"不应该发生"的情况。**

- `panic`：表示不可恢复的严重错误（bug、基础设施损坏），会沿着调用栈向上回溯并触发 defer
- `recover`：只能在 defer 函数中捕获 panic，捕获后可以恢复程序继续运行
- **最佳实践**：在 goroutine 入口处用 recover 做兜底，防止 goroutine 崩溃导致整个进程退出；HTTP handler 层用 recover 记录日志后返回 500

记住：**能返回 error 的地方绝不用 panic**。

---

## 🔬 深度展开

### 1. Panic 的生命周期

```
panic(x)
    ↓
runtime.gopanic()  // 进入 panic 流程
    ↓
沿当前 goroutine 的调用栈从内向外执行 defer 函数
    ↓
如果某处的 defer 中调用了 recover() → panic 被捕获，控制流回到 recover 调用处
    ↓
如果没有 recover() → 打印 panic message 和堆栈，goroutine 退出（多个 goroutine 都没 recover 时整个程序退出）
```

关键事实：
- **panic 不会立即终止程序**，它会先执行所有 deferred 函数
- **只有 defer 中的 recover() 才能捕获 panic**，普通函数不行
- **recover() 只能捕获同一 goroutine 的 panic**，跨 goroutine 不行

### 2. 正确的 Recover 模式

#### 模式一：HTTP Handler 兜底（最常见）

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/api/data", func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("PANIC in %s: %v", r.URL.Path, err)
                w.Header().Set("Content-Type", "application/json")
                w.WriteHeader(http.StatusInternalServerError)
                json.NewEncoder(w).Encode(map[string]string{
                    "error": "internal server error",
                })
            }
        }()
        
        // 业务逻辑...
        result := processRequest(r)
        json.NewEncoder(w).Encode(result)
    })
    
    http.ListenAndServe(":8080", mux)
}
```

**为什么这是推荐做法？**
- 保证单个请求崩溃不会影响其他请求
- 优雅地返回 500 + 日志，方便排查
- 不泄露内部实现细节给客户端

#### 模式二：Goroutine 入口兜底

```go
go func() {
    defer func() {
        if err := recover(); err != nil {
            log.Printf("Recovered from panic: %v\n%s", err, debug.Stack())
            // 可选：发送告警通知
        }
    }()
    
    longRunningTask()
}()
```

**为什么必须这样做？**
- 没有 recover 的 goroutine panic 会导致整个进程退出
- 对于后台任务（定时任务、消息消费者等），一个任务的 crash 不能拖垮整个服务

#### 模式三：可恢复的业务场景

```go
func parseAndProcess(data []byte) error {
    defer func() {
        if r := recover(); r != nil {
            // 解析过程中发生了意料之外的错误，转为 error 返回
            log.Printf("Parse recovered: %v", r)
        }
    }()
    
    // 可能 panic 的操作
    val := mustUnmarshal(string(data))
    return doSomething(val)
}
```

**注意**：这种情况要谨慎。通常更好的做法是让函数返回 error 而不是 panic。

### 3. 什么时候该用 Panic？

✅ **适合 panic 的场景：**
- 构造函数参数校验失败（无法返回有效对象）
- 初始化阶段发现无法修复的问题（配置加载失败）
- 数组/切片越界访问（这是 bug，应该修代码）
- 断言失败（assertion failed）
- 不可恢复的状态损坏

❌ **不该用 panic 的场景：**
- 业务逻辑错误（用户输入非法、数据库查询为空、文件不存在）
- 网络超时、连接失败
- 资源不足（但应尝试 graceful degradation）
- 任何其他可以通过返回值传递的错误

### 4. Panic 对性能的影响

虽然正常路径（不触发 panic）的性能开销极低，但仍然需要注意：

| 指标 | 开销 | 说明 |
|------|-----|------|
| 编译器优化 | 轻微增加 | Go 编译器在处理含 defer/panic 的函数时内联决策更保守 |
| 运行时 | 接近零 | 未触发 panic 时几乎零开销 |
| 触发 panic | 显著 | 需要构建 stack trace，涉及 runtime 复杂操作 |

**关键数据**：根据 Go 官方 Benchmark，在正常路径上不触发 panic 的情况下，使用 recover defer 函数的额外开销通常在 **纳秒级别**（< 10ns），远低于网络 I/O 或磁盘 I/O 的开销。

### 5. Panic 与 GC 的关系

当 panic 发生时：
1. 正在执行的 goroutine 的所有局部变量进入 "unreachable" 状态
2. GC 会在下一个扫描周期回收这些内存
3. 但是 **已分配的 mspan/mcache 不会被立即释放**，只是标记为 free

这意味着频繁的 panic 会导致一定的内存碎片化，但这通常是极端场景才需要考虑的问题。

### 6. 常见陷阱

#### 陷阱一：忘记 recover 导致进程崩溃

```go
func handleTask() {
    go func() {
        // ❌ 没有 recover，如果这里 panic，整个程序退出
        criticalOperation()
    }()
}
```

#### 陷阱二：在 defer 外面调用 recover

```go
func wrong() {
    r := recover() // ❌ recover 必须在 defer 中调用，否则会返回 nil
    // ...
}

func right() {
    defer func() {
        if r := recover(); r != nil {
            // ✅ 正确
        }
    }()
    // ...
}
```

#### 陷阱三：误吞 panic

```go
func careless() {
    defer func() {
        recover() // ❌ 什么都不做就 recover，等于吞掉了 panic
    }()
    // ...
}

func careful() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("Recovered panic: %v", r)  // ✅ 至少记录日志
        }
    }()
    // ...
}
```

#### 陷阱四：Recover 后错误传播

```go
func handler() error {
    defer func() {
        if r := recover(); r != nil {
            // ❌ 如果在 recover 中设置返回值，行为不确定
            // recover 返回后会继续执行后续代码
        }
    }()
    
    doThing()
    return nil // 如果 panic 被 recover，这行还会执行吗？
}

// 正确做法：用一个命名返回值
func safeHandler() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()
    
    doThing()
    return nil
}
```

---

## 💻 完整示例：生产级错误处理中间件

```go
package middleware

import (
    "context"
    "fmt"
    "log/slog"
    "net/http"
    "runtime/debug"
    "time"
)

// RecoverMiddleware 生产级 panic 恢复中间件
func RecoverMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // 使用自定义 response writer 捕获 HTTP 响应
        rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        
        defer func() {
            if err := recover(); err != nil {
                // 记录详细日志
                slog.ErrorContext(r.Context(), "Panicked",
                    "path", r.URL.Path,
                    "method", r.Method,
                    "error", err,
                    "stack", string(debug.Stack()),
                    "duration", time.Since(start),
                )
                
                // 返回通用错误响应
                w.Header().Set("Content-Type", "application/json")
                w.WriteHeader(http.StatusInternalServerError)
                fmt.Fprintf(w, `{"error":"internal server error"}`)
            }
        }()
        
        next.ServeHTTP(rw, r)
    })
}

// GoroutineRecoverer 用于启动新 goroutine 时的 recover 装饰器
type GoroutineRecoverer struct {
    logger *slog.Logger
}

func NewGoroutineRecoverer(logger *slog.Logger) *GoroutineRecoverer {
    return &GoroutineRecoverer{logger: logger}
}

func (g *GoroutineRecoverer) Run(ctx context.Context, fn func(ctx context.Context)) {
    go func() {
        defer func() {
            if err := recover(); err != nil {
                g.logger.ErrorContext(ctx, "Goroutine panicked",
                    "error", err,
                    "stack", string(debug.Stack()),
                )
            }
        }()
        
        fn(ctx)
    }()
}
```

---

## 🗣️ 面试话术

> "我对 panic/recover 的原则是：panic 是给'程序员错了'用的，不是给'业务错了'用的。业务错误永远走 error 返回值。panic 的典型使用场景是构造函数校验失败、初始化阶段发现致命问题、或者不可恢复的系统级错误。recover 的合理用法是在系统边界处做兜底——HTTP handler 顶层和 goroutine 入口处各放一个 recover，这样既能防止进程崩溃，又能记录有用的上下文信息用于排查。"

> "一句话总结：能用 error 解决的，绝不 panic。panic/recover 是安全网，不是错误处理机制。"

---

## ✅ 速查表

| 场景 | 方案 | 理由 |
|------|------|------|
| 用户输入非法 | 返回 error | 正常的业务流程 |
| 文件不存在 | 返回 error | 可预见的情况 |
| 数据库连接失败 | 返回 error + retry | 可重试 |
| 构造函数参数无效 | panic | 不可能创建合法对象 |
| HTTP handler 崩溃 | recover + 500 | 保护服务整体可用性 |
| 后台 goroutine 崩溃 | recover + 告警 | 防止单任务拖垮进程 |

<div align="right">
<i>最后更新：2026-08-12 ｜ 模块：Go 语言深度 · 并发编程</i>
</div>
