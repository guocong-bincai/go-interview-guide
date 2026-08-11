# Context Value 传递陷阱与最佳实践

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：context.WithValue、key 类型、超时误用、Context 泄露、请求链路追踪

## 🎯 面试官考察意图

面试中经常出现"使用 context 传递用户信息/认证 token"的场景题。面试官想确认：

1. 候选人是否理解 `context.WithValue` 的设计初衷——**它不是用来传业务数据的**
2. 是否知道 key 必须是可比较的且建议定义为自定义类型防止冲突
3. 是否能识别常见反模式（如用 Context 传配置、传数据库连接池）
4. 是否了解 HTTP request context 的生命周期管理

---

## ⚡ 核心答案（30秒版）

**`context.Context` 是请求范围的元数据载体，设计用于取消信号和截止时间传递，不是通用的参数传递工具。**

- ✅ **正确用途**：传递 timeout、cancel signal、request-scoped values（用户ID、traceID）
- ❌ **错误用途**：传配置字典、传数据库连接池、传需要共享修改的全局变量
- ⚠️ **关键陷阱**：WithValue 创建的 context 是一个全新的 context，修改它的父 context 不会影响子 context，反之亦然

---

## 🔬 深度展开

### 1. WithValue 的正确用法

```go
type contextKey string

const (
    userIDKey   contextKey = "userID"
    traceIDKey  contextKey = "traceID"
)

// ✅ 正确：在请求入口处设置
func authMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        userID := r.Header.Get("X-User-ID")
        
        // 创建新的 context，注入用户信息
        ctx := context.WithValue(r.Context(), userIDKey, userID)
        
        // 把新 context 传递给下游
        next.ServeHTTP(w, r.WithContext(ctx))
    }
}

// ✅ 正确使用：在 handler 中读取
func getUserProfile(w http.ResponseWriter, r *http.Request) {
    userID, ok := r.Context().Value(userIDKey).(string)
    if !ok || userID == "" {
        http.Error(w, "unauthorized", http.StatusUnauthorized)
        return
    }
    
    // 查询用户信息...
}
```

### 2. Key 的类型选择（重要！）

```go
type contextKey string

const userIDKey contextKey = "userID"

// ❌ 错误：用 int / string 字面量作为 key
ctx := context.WithValue(r.Context(), "userID", "123")
// 问题：第三方库如果也用 "userID" 做 key，会互相覆盖！

// ✅ 正确：用自定义类型，确保唯一性
type userIDKey string
const myUserIDKey userIDKey = "myapp.userID"

// ✅ 更安全的做法：用 struct 包装
type userIDKeyType struct{}
var userIDKey = userIDKeyType{}

ctx := context.WithValue(r.Context(), userIDKey, "123")
val := ctx.Value(userIDKey).(string)
```

**为什么不能用普通类型？**
context 包文档明确警告："Keys should be of a type that is not comparable to strings or other built-in types." 因为 Go map 的比较规则会导致潜在冲突。

### 3. Context 层级与继承关系

```
parentCtx
    │
    ├─ childCtx1 = WithValue(parentCtx, K1, V1)  ← 只含 K1/V1
    │       │
    │       └─ grandChild = WithCancel(childCtx1)   ← 继承 parentCtx+CANCEL + K1/V1
    │
    └─ childCtx2 = WithCancel(parentCtx)          ← 只有 CANCEL，不含 K1/V1
```

**关键事实：**
- 调用 `WithValue` 返回的是一个新的 context，**旧 context 不会受到影响**
- 子 context 可以访问所有祖先 context 的值（沿 chain 向上查找）
- 但是父 context 修改自己的值，子 context **看不到**（value 只在创建时复制一次）

```go
parent := context.Background()
child1 := context.WithValue(parent, "k", "v1")
fmt.Println(child1.Value("k")) // "v1"

parent = context.WithValue(parent, "k", "updated") // 这是新的 context！
fmt.Println(child1.Value("k")) // 仍然是 "v1"！父没变
fmt.Println(parent.Value("k")) // "updated"
```

### 4. 常见反模式与陷阱

#### 陷阱一：用 Context 传大量数据或复杂结构

```go
// ❌ 不推荐：传完整配置对象
type Config struct {
    DBHost     string
    DBPort     int
    CacheTTL   time.Duration
    // ... 几十上百个字段
}

ctx := context.WithValue(r.Context(), "config", &config)
// 问题：context 应该轻量，不应该承载业务数据

// ✅ 正确：如果需要全局配置，直接用 package-level 变量或单例
```

#### 陷阱二：用 Context 传 Database Connection Pool

```go
// ❌ 不推荐
ctx := context.WithValue(r.Context(), "dbPool", dbPool)
pool := ctx.Value("dbPool").(*sql.DB)

// ✅ 正确：db.Pool 是全局可访问的资源
pool := database.GetPool()
```

#### 陷阱三：忽略超时导致 goroutine 泄漏

```go
// ❌ 危险：没有超时控制
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

go func() {
    // 这个 goroutine 可能永远跑不完！
    doLongTask(ctx)
}()

// ✅ 必须有超时
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

go func() {
    doLongTask(ctx)
    cancel() // 完成时手动取消
}()
```

#### 陷阱四：在循环中复用同一个 context

```go
// ❌ 错误：for 循环中反复 reuse ctx
for _, task := range tasks {
    go func() {
        doWork(ctx) // 所有人共享同一个 ctx！一个 done 全死
    }()
}

// ✅ 正确：每次创建独立的 context
for _, task := range tasks {
    taskCtx := context.WithValue(ctx, "taskID", task.ID)
    go func() {
        doWork(taskCtx)
    }()
}
```

### 5. Context 生命周期管理

```go
// 标准模式：handler → service → repository
func Handler(w http.ResponseWriter, r *http.Request) {
    // Step 1: 从 HTTP request 获取 context
    ctx := r.Context()
    
    // Step 2: 添加请求级别的数据
    ctx = context.WithValue(ctx, userIDKey, getUserID(r))
    ctx = context.WithValue(ctx, traceIDKey, generateTraceID())
    
    // Step 3: 传入服务层
    svc := NewService()
    result, err := svc.Process(ctx)
    if err != nil {
        log.Printf("error: %v", err)
        http.Error(w, "failed", http.StatusInternalServerError)
        return
    }
    
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(result)
}

func (s *Service) Process(ctx context.Context) (*Result, error) {
    // 使用 context 中的 traceID 进行日志记录
    traceID := ctx.Value(traceIDKey).(string)
    slog.InfoContext(ctx, "processing", "traceID", traceID)
    
    // 传给 repository
    user, err := s.repo.FindByID(ctx, getUserID(ctx))
    if err != nil {
        return nil, fmt.Errorf("find user: %w", err)
    }
    
    return &Result{User: user}, nil
}

func (r *Repo) FindByID(ctx context.Context, id string) (*User, error) {
    // 检查 ctx 是否已取消
    select {
    case <-ctx.Done():
        return nil, ctx.Err() // 返回 context.Canceled 或 context.DeadlineExceeded
    default:
    }
    
    return r.db.QueryRowContext(ctx, "SELECT * FROM users WHERE id = $1", id)
}
```

### 6. HTTP Request Context 的特殊性

```go
// 每个 HTTP request 自带一个 context
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // 这就是默认的 request context
    
    // 当客户端断开连接时，r.Context().Done() channel 会关闭
    // 这意味着你可以用 ctx 来检测客户端是否取消了请求
    select {
    case <-ctx.Done():
        log.Println("Client disconnected!")
        return
    case result := <-doExpensiveComputation():
        w.Write(result)
    }
}
```

**注意**：Go 1.22+ 中 `r.Context()` 返回的是 `context.Context` 而不是原来的 `*http.Request.Context()`。语义不变，但 API 更简洁。

---

## 💻 完整示例：中间件链式注入 Context

```go
package middleware

import (
    "context"
    "net/http"
    "strings"
)

type contextKey string

// 预定义的 key 类型，避免冲突
type userIDKeyType struct{}

var UserIDKey contextKey = "middleware.userID"
var TraceIDKey contextKey = "middleware.traceID"

// AuthMiddleware: 注入用户身份
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "missing auth header", http.StatusUnauthorized)
            return
        }
        
        // 简化示例：实际应验证 JWT 等
        userID := strings.TrimPrefix(authHeader, "Bearer ")
        ctx := context.WithValue(r.Context(), UserIDKey, userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// TraceMiddleware: 注入 traceID
func TraceMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        traceID := r.Header.Get("X-Trace-ID")
        if traceID == "" {
            traceID = generateUUID()
        }
        ctx := context.WithValue(r.Context(), TraceIDKey, traceID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// ExtractValue 安全的 context value 提取函数
func ExtractUserID(ctx context.Context) string {
    v := ctx.Value(UserIDKey)
    if v == nil {
        return ""
    }
    s, ok := v.(string)
    if !ok {
        return ""
    }
    return s
}

func extractTraceID(ctx context.Context) string {
    v := ctx.Value(TraceIDKey)
    if v == nil {
        return ""
    }
    s, ok := v.(string)
    if !ok {
        return ""
    }
    return s
}
```

---

## 🗣️ 面试话术

> "我对 context 的理解是：它是 'request scope' 的元数据通道，主要用途是传取消信号、deadline 和 request-scoped 的信息如用户ID、traceID。但它不是万能的 parameter passing 机制。我的原则是：能用函数参数传的，就用函数参数；需要跨 goroutine 传播且带生命周期的，才用 context。另外 key 一定要用自定义不可比类型，防止和第三方库冲突。"

> "记住一句话：Context 是 for passing request-scoped values and cancellation signals, not for optional parameters."

---

## ✅ 速查表

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 超时控制 | `WithTimeout` / `WithDeadline` | Context 的核心使命之一 |
| 取消信号 | `WithCancel` | 通知下游停止工作 |
| 用户ID/TraceID | `WithValue` | request-scoped 的元数据 |
| 全局配置 | package 变量 / singleton | 不需要 request scope |
| 数据库连接池 | package 变量 | 全局共享资源 |
| 可选函数参数 | 函数选项模式 / variadic | Context 不是 parameter 容器 |

<div align="right">
<i>最后更新：2026-08-12 ｜ 模块：Go 语言深度 · 并发编程</i>
</div>
