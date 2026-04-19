# Go 错误处理最佳实践

> 考察频率：★★★★☆  优先级：P1

---

## 1. 面试官考察意图

Go 的错误处理是区分工程师水平的分水岭。初级工程师只会 `if err != nil { return err }`，高级工程师知道：1）如何用 errors.Is/As 做错误链判断；2）如何包装错误而不丢失原信息；3）如何设计自定义错误类型；4）panic/recover 的正确使用边界。这道题考察的是 Go 错误处理的**工程化思维**。

---

## 2. 核心答案（30秒版）

Go 错误是一个普通的 `interface{ Error() string }`，本质上是一个字符串。推荐用法：**sentinel error**（预定义常量）用于库级判断；**自定义错误类型**（struct）用于携带结构化信息；**errors.Is/As** 而不是类型断言来判断错误链；**%w** 而不是 `fmt.Errorf` 来包装错误。panic 只用于真正的不可恢复错误（如数组越界、严重的编程错误），不用于业务错误。

---

## 3. 深度展开

### 3.1 error 接口本质

```go
// error 接口只有这一个方法
type error interface {
    Error() string
}

// 很多内置类型实现了 error
// io.EOF, os.ErrNotExist, syscall.ENOENT 等

// 自定义错误极其简单
type SyntaxError struct {
    Line int
    Col  int
    Msg  string
}

func (e *SyntaxError) Error() string {
    return fmt.Sprintf("%d:%d: %s", e.Line, e.Col, e.Msg)
}
```

### 3.2 errors.Is / errors.As / errors.Unwrap

#### errors.Is — 判断错误链中是否有指定错误

```go
// ❌ 旧方式：只能判断最外层错误
if err == os.ErrPermission {
    // 不准确：err 可能是包装过的
}

// ✅ 新方式：顺着 unwrap 链找
if errors.Is(err, os.ErrPermission) {
    // 正确：会 unwrap 层层包装，找到最终原因
}
```

```go
// 示例：包装链路
err1 := os.ErrPermission               // 原始错误
err2 := fmt.Errorf("db query: %w", err1) // 包装一层
err3 := fmt.Errorf("service call: %w", err2) // 再包装一层

// errors.Is 会顺着 unwrap 链找到 os.ErrPermission
errors.Is(err3, os.ErrPermission) // true
```

#### errors.As — 从错误链中提取特定类型

```go
// ❌ 旧方式：类型断言
if perr, ok := err.(*os.PathError); ok {
    fmt.Println(perr.Path)
}

// ✅ 新方式：用 errors.As
var perr *os.PathError
if errors.As(err, &perr) {
    fmt.Println("path:", perr.Path)
}
```

#### errors.Unwrap — 解包一层

```go
// errors.Unwrap 返回被包装的下一层错误
err := fmt.Errorf("context: %w", os.ErrNotExist)
unwrapped := errors.Unwrap(err)
fmt.Println(unwrapped) // "file does not exist"
```

### 3.3 %w 包装 vs 直接 fmt.Errorf

```go
// ❌ 旧方式：信息丢失，无法用 errors.Is 判断原因
err := fmt.Errorf("db error: %s", err) // %s 只保留字符串，原错误丢失

// ✅ 正确方式：%w 保留完整错误链
err := fmt.Errorf("db error: %w", originalErr)
errors.Is(err, originalErr) // true
```

#### 最佳实践：多层包装

```go
func readConfig(path string) error {
    f, err := os.Open(path)
    if err != nil {
        // 只包装一次，保留最少的上下文
        return fmt.Errorf("open config: %w", err)
    }
    defer f.Close()

    var cfg Config
    if err := json.NewDecoder(f).Decode(&cfg); err != nil {
        return fmt.Errorf("decode config: %w", err)
    }
    return nil
}

// 调用方
err := readConfig("/etc/app/config.json")
if err != nil {
    if errors.Is(err, os.ErrNotExist) {
        // 处理文件不存在
    } else if errors.Is(err, os.ErrPermission) {
        // 处理权限问题
    } else {
        // 处理其他错误
    }
}
```

### 3.4 自定义错误类型：sentinel error vs 自定义 struct

#### sentinel error（预定义常量）

```go
// 定义在包级别
var (
    ErrNotFound      = errors.New("not found")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrAlreadyExists = errors.New("already exists")
)

// 使用
if errors.Is(err, ErrNotFound) {
    // 处理找不到的情况
}
```

适用场景：库/SDK 提供给外部使用的标准错误，调用方需要精确判断。

#### 自定义错误类型（struct）

```go
type ValidationError struct {
    Field   string
    Message string
    Cause   error // 可选的原始错误
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on '%s': %s", e.Field, e.Message)
}

func (e *ValidationError) Unwrap() error {
    return e.Cause
}

// 使用
if err := validateEmail(email); err != nil {
    var ve *ValidationError
    if errors.As(err, &ve) {
        fmt.Printf("字段 %s 校验失败: %s\n", ve.Field, ve.Message)
    }
}
```

**何时用 struct**：需要携带额外上下文信息（字段名、具体的业务参数）。

**何时用 sentinel**：只需要知道「是什么类型的错误」，不需要额外信息。

### 3.5 panic / recover 的正确使用边界

Go 的哲学是**错误是值**，panic 不是错误处理的主要手段。但 panic 在特定场景下是合理的：

```go
// ✅ 合理的 panic 场景：

// 1. 真正的编程错误（不可恢复）
func GetElement(slice []int, idx int) int {
    if idx < 0 || idx >= len(slice) {
        panic(fmt.Sprintf("index out of range: %d", idx))
        // 这是编程错误，不是业务错误，应该立即暴露
    }
    return slice[idx]
}

// 2. 启动时配置错误（服务无法启动）
func main() {
    cfg, err := loadConfig()
    if err != nil {
        log.Fatalf("配置加载失败，服务无法启动: %v", err)
        // 这里用 log.Fatalf 而不是 panic，但在容器环境中可以 panic 让进程退出
    }
}

// 3. 不可恢复的严重错误（如数据库连接彻底失败、依赖服务完全不可用）
if db == nil {
    panic("database connection is nil")
}
```

```go
// ❌ 不应该用 panic 的场景：

// 业务错误（如用户名不存在、余额不足）
func Withdraw(acc *Account, amount int) error {
    if acc.Balance < amount {
        return fmt.Errorf("余额不足") // ✅ 返回 error
        // panic("余额不足") ❌ 这是业务错误，应该正常返回
    }
    acc.Balance -= amount
    return nil
}

// 可预见的错误（如文件不存在、网络超时）
if os.IsNotExist(err) {
    return err // ✅ 正常返回
}
```

#### recover 的正确用法

```go
// recover 只在 defer 中生效
func safeCall(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recovered: %v", r)
            // 记录日志，上报监控
            log.Printf("panic in safeCall: %v\n%s", r, debug.Stack())
        }
    }()
    fn()
    return nil
}

// HTTP Server 中的 recover 中间件
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v", err)
                http.Error(w, "internal server error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### 3.6 生产最佳实践

#### 分层错误处理

```
┌─────────────────────────────────────────────────────────┐
│  Go 错误处理分层模型                                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  底层库（db, redis, http client）                        │
│    → 只返回原始错误，不打印日志，不包装（让调用方决定）        │
│    → 示例：if err != nil { return err }                  │
│                                                         │
│  中间层（service, business logic）                       │
│    → 包装错误，添加业务上下文                             │
│    → 示例：return fmt.Errorf("create order: %w", err)    │
│                                                         │
│  最上层（handler, main）                                │
│    → 判断错误类型，做最终处理（返回响应/告警/重试）          │
│    → 示例：if errors.Is(err, ErrNotFound) { 404 }        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 错误与日志的正确关系

```go
// ❌ 错误被打印两次
func handler() {
    err := doSomething()
    if err != nil {
        log.Printf("error: %v", err)  // 第一次：handler 打印
        return err                    // 第二次：调用方也打印
    }
}

// ✅ 只在最上层打印
func handler() error {
    return doSomething() // 只返回，不打印
}

func main() {
    if err := handler(); err != nil {
        log.Printf("handler failed: %v", err) // 只在这里打印
        // 或：发送告警、记录metrics
    }
}
```

#### Wrap 层数控制

```go
// 原则：只包装一次，不要过度包装
// 每一层只加自己这一层的上下文

func A() error {
    err := B()
    if err != nil {
        return fmt.Errorf("A failed: %w", err) // ✅ 只包装一次
        // return fmt.Errorf("A failed: %w: %v", err, C()) ❌ 不要多层包装
    }
    return nil
}
```

---

## 4. 高频追问

### Q1：如何在不改原始错误的情况下附加上下文信息？

> 用 `fmt.Errorf` 和 `%w`。`%w` 是唯一正确的错误包装方式，它返回一个同时实现了 `error` 接口和 `Unwrap() error` 接口的新错误。原始错误通过 `errors.Unwrap()` 可以逐层解开。不改原始错误意味着调用方仍然可以用 `errors.Is(err, originalErr)` 判断。

### Q2：errors.Is 和 errors.As 内部是怎么实现的？

> `errors.Is` 内部会沿着 `Unwrap()` 链逐层往上找，直到找到匹配的错误或链结束。`errors.As` 类似，但是用类型断言找第一个匹配的类型。
>
> ```go
> func Is(err, target error) bool {
>     // 遍历 unwrap 链
>     for {
>         if err == target {
>             return true
>         }
>         if err = Unwrap(err); err == nil {
>             return false
>         }
>     }
> }
> ```

### Q3：第三方库的错误怎么正确处理？

> 1）判断是否第三方错误：用 `errors.Is/As`，不用 `err == xxx`。2）如果第三方返回的错误需要传给下游，用 `%w` 继续包装。3）如果第三方错误需要做特殊处理（如网络超时要重试），用 `errors.As` 提取结构化信息。

### Q4：error 和异常（panic）的边界是什么？

> 核心原则：**可预见的错误（业务错误）用 error，不可预见的严重错误用 panic**。网络超时、文件不存在、权限不足都是可预见的，应该返回 error。数组越界、空指针、解码失败（编程错误）用 panic，因为说明代码有 bug。简单记忆：**如果是调用方的责任，用 error；如果是编写者自己的责任，用 panic**。

---

## 5. 延伸阅读

- [Go 错误处理官方博客](https://go.dev/blog/error-handling-and-go)
- [Go 1.13 errors 包改进](https://go.dev/blog/go1.13-errors)
- [Effective Go - Error 章节](https://go.dev/doc/effective_go#errors)
- [Practical Go Lessons - Error Handling](https://dmitri.shuralyov.com/idiomatic-go/)
