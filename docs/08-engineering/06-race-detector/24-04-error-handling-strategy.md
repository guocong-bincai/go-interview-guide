# 错误处理最佳实践：errors.Is/As/Wrap 与自定义错误

> 考察频率：★★★★★  难度：★★★☆☆
> 关键词：errors.Is、errors.As、errors.Wrap、Sentinel Error、Custom Error、错误包装链

## 面试官考察意图

错误处理是 Go 面试中的**必考项**。大多数候选人知道"return error"，但很少真正理解 `errors` 包提供的工具函数和正确用法。高级工程师需要能讲清错误类型体系、包装机制和实际应用场景。

这道题区分的是：**你是只会写 `fmt.Errorf("xxx failed")` 的人，还是理解 Go 错误哲学的设计者。**

---

## 核心答案（30 秒版）

Go 的错误处理不是 throw-catch，而是 return-forward。三个核心工具：`errors.New` 创建 sentinel errors（固定不变），`fmt.Errorf` + `%w` 包装错误（保留原始错误），`errors.Is/As` 检查错误（支持 wrapped chain）。最佳实践：用 sentinel errors 定义 API 级错误类型，用 `%w` 包装上下文信息，上层用 `errors.Is` 做判断。不要只返回字符串错误——它失去了结构化能力。

---

## Go 错误类型体系

### 三种错误处理方式

```go
// 方式 1：errors.New — Sentinel Error（固定值）
var ErrNotFound = errors.New("not found")

func findUser(id int) (*User, error) {
    if id <= 0 {
        return nil, ErrNotFound
    }
    return user, nil
}

// 方式 2：fmt.Errorf — 简单错误消息
func validateEmail(email string) error {
    if !strings.Contains(email, "@") {
        return fmt.Errorf("invalid email format: %s", email)
    }
    return nil
}

// 方式 3：errors.New + context —— 推荐模式（带上下文的 sentinel）
func findUserV2(id int) (*User, error) {
    if id <= 0 {
        return nil, fmt.Errorf("find user by id %d: %w", id, ErrNotFound)
    }
    return user, nil
}
```

### Sentinel Errors vs Dynamic Errors

| 类型 | 创建方式 | 特点 | 适用场景 |
|------|---------|------|---------|
| **Sentinel** | `errors.New("msg")` | 固定的全局变量 | API 契约级别的错误 |
| **Dynamic** | `fmt.Errorf("detail: %v", err)` | 每次运行生成新实例 | 临时性/一次性错误 |

---

## 错误包装：errors.Wrapper 接口

### `%w` vs `%v` 的区别

```go
// ❌ %v 只是拼接字符串，丢失了错误链
err := fmt.Errorf("failed to save user: %v", db.ErrRecordNotFound)
// 此时无法用 errors.Is(err, db.ErrRecordNotFound) 匹配到原错误！

// ✅ %w 包装错误，保留完整错误链
err := fmt.Errorf("failed to save user: %w", db.ErrRecordNotFound)
// 现在可以用 errors.Is 向上追溯找到 db.ErrRecordNotFound
```

### 原理

```go
// errors.Wrapper 接口
type Wrapper interface {
    Unwrap() error
}

// %w 的本质就是实现了 Unwrap() 的 customError struct
// errors.Is 会递归调用 Unwrap() 来遍历整个错误链
```

---

## errors.Is / As / Unwrap 详解

### errors.Is：判断错误是否在链中

```go
func processUser(id int) error {
    user, err := findUser(id)
    if err != nil {
        if errors.Is(err, db.ErrNotFound) {
            // 用户不存在 → 返回 404
            return &AppError{Code: 404, Msg: "user not found"}
        }
        return fmt.Errorf("failed to find user: %w", err)
    }
    
    // ... 处理逻辑
    return nil
}

// 验证
err := processUser(999)
if errors.Is(err, db.ErrNotFound) {
    // ✅ 能找到被包装的 ErrNotFound
}
```

### errors.As：将错误转为具体类型

```go
type MySpecificError struct {
    Code int
    Detail string
}

func (e *MySpecificError) Error() string {
    return fmt.Sprintf("error code %d: %s", e.Code, e.Detail)
}

func handle(err error) {
    var myErr *MySpecificError
    if errors.As(err, &myErr) {
        log.Printf("Got specific error: code=%d detail=%s", myErr.Code, myErr.Detail)
        // 可以访问具体的字段
    }
}
```

### errors.Unwrap：直接获取包装层的下一个错误

```go
// 如果你想只看一层（不递归）
raw := errors.Unwrap(err)
```

---

## 实战：设计完整的错误体系

### Layered Error Architecture

```
Application Layer         ────> AppError {Code, Message, TraceID}
Service Layer             ────> ServiceError {Op, Entity, Cause}
Repository Layer          ────> Sentinel Errors (ErrNotFound, ErrDuplicateKey)
External Dependency       ────> ThirdPartyError {HTTPStatusCode, Body}
```

### 实现示例

```go
package apperror

import (
    "errors"
    "fmt"
)

// === 1. Repository Level: Sentinels ===

var (
    ErrNotFound     = errors.New("record not found")
    ErrDuplicateKey = errors.New("duplicate key violation")
    ErrLockConflict = errors.New("optimistic lock conflict")
)

// === 2. Service Level: Custom Error Type ===

type ServiceError struct {
    Op      string // "order.create"
    Entity  string // "Order"
    Cause   error
    Retryable bool
}

func (e *ServiceError) Error() string {
    return fmt.Sprintf("%s.%s failed: %v", e.Op, e.Entity, e.Cause)
}

func (e *ServiceError) Unwrap() error {
    return e.Cause
}

func NewServiceError(op, entity string, cause error) *ServiceError {
    return &ServiceError{Op: op, Entity: entity, Cause: cause}
}

// Helper functions
func IsNotFoundError(err error) bool {
    return errors.Is(err, ErrNotFound)
}

func IsRetryable(err error) bool {
    var se *ServiceError
    if errors.As(err, &se) && se.Retryable {
        return true
    }
    return false
}

// === 3. Application Level ===

type AppError struct {
    HTTPCode int    `json:"-"`
    Code     string `json:"code"`
    Message  string `json:"message"`
    TraceID  string `json:"trace_id,omitempty"`
}

func (e *AppError) Error() string {
    return e.Message
}

func NotFound(msg string) *AppError {
    return &AppError{HTTPCode: 404, Code: "NOT_FOUND", Message: msg}
}

func Internal(serverMsg string) *AppError {
    return &AppError{HTTPCode: 500, Code: "INTERNAL_ERROR", Message: serverMsg}
}

// === 4. Handler Layer: Translate between layers ===

func HandleServiceError(err error) error {
    if err == nil {
        return nil
    }
    
    if IsNotFoundError(err) {
        return NotFound("requested resource not found")
    }
    
    if IsRetryable(err) {
        return Internal("service temporarily unavailable, please retry")
    }
    
    // Fallback
    return Internal("unexpected error occurred")
}
```

### 使用示例

```go
func CreateOrder(ctx context.Context, req *CreateOrderRequest) (*Order, error) {
    // 1. Validate
    if err := validate(req); err != nil {
        return nil, fmt.Errorf("validate order: %w", err)
    }
    
    // 2. Check inventory
    if ok, err := checkInventory(req.Items); err != nil {
        return nil, fmt.Errorf("check inventory: %w", err)
    } else if !ok {
        return nil, fmt.Errorf("check inventory: %w", ErrInsufficientStock)
    }
    
    // 3. Create order
    order, err := repo.CreateOrder(ctx, req.ToDomain())
    if err != nil {
        return nil, fmt.Errorf("order.repo.create: %w", 
            &ServiceError{
                Op:      "order.create",
                Entity:  "Order",
                Cause:   err,
                Retryable: isTransient(err),
            })
    }
    
    // 4. Send notification
    go func() {
        if err := notifyCustomer(order.UserID, order.ID); err != nil {
            log.Printf("notification failed (non-fatal): %v", err)
            // 非关键操作失败不影响主流程
        }
    }()
    
    return order, nil
}
```

---

## 常见错误处理模式

### 模式 1：非致命错误（记录但不阻断）

```go
// 发送通知失败不影响主流程
go func() {
    if err := sendNotification(user, msg); err != nil {
        logger.Warn("send notification failed", zap.Error(err))
        // 不回传错误给调用方
    }
}()
```

### 模式 2：降级策略

```go
func getCacheOrFallback(key string) (string, error) {
    val, err := redis.Get(key)
    if err != nil {
        // Redis 不可用时降级到内存缓存
        logger.Warn("redis unavailable, falling back to local cache", zap.Error(err))
        return memcache.Get(key)
    }
    return val, nil
}
```

### 模式 3：错误聚合

```go
// 批量操作中某个失败时，收集所有错误而非立即返回
type AggregateError struct {
    Errors []error
}

func (e *AggregateError) Error() string {
    msgs := make([]string, len(e.Errors))
    for i, err := range e.Errors {
        msgs[i] = err.Error()
    }
    return fmt.Sprintf("aggregate error (%d failures): %s", len(e.Errors), strings.Join(msgs, "; "))
}

func (e *AggregateError) Unwrap() []error {
    return e.Errors
}

// Go 1.20+ 支持 slice unwrap
func aggregateErrors(errs []error) error {
    if len(errs) == 0 {
        return nil
    }
    var lastErr error
    for _, err := range errs {
        if err != nil {
            lastErr = fmt.Errorf("%w; %w", lastErr, err)
        }
    }
    return lastErr
}
```

---

## 高频追问

**Q：什么时候用 sentinel error，什么时候用动态 error？**
A：API 契约级别的错误用 sentinel（如 `ErrNotFound`），运行时产生的临时错误用 `fmt.Errorf`。Sentinel 用于 `errors.Is` 判断，动态 error 用于提供上下文细节。

**Q：`%w` 和 `%v` 的区别是什么？为什么这么设计？**
A：`%w` 创建了 `unwrappedError` struct，保留了错误链，允许 `errors.Is/As` 回溯。`%v` 只是文本拼接，丢失了结构信息。这个设计是为了让错误处理保持灵活性——既可以当字符串用，也可以结构化分析。

**Q：如何判断一个 error 是否可重试？**
A：通过自定义错误类型 + `errors.As` 来提取具体类型，然后检查它的字段或方法。不要硬编码比较错误消息字符串。

**Q：Go 1.20 后 errors join 功能怎么用？**
A：`errors.Join(errs...)` 接受多个 error 返回一个复合 error，可以用 `errors.Is` 和 `errors.As` 对每个子错误做检查，非常适合批量操作的错误聚合。

---

## 延伸阅读

- [Go Blog: Go 1.20 — Multiple Error Handling](https://go.dev/blog/go1.20-errors-join)
- [Uber Go Style Guide: Error Handling](https://github.com/uber-go/guide/blob/master/style.md#error-handling)
- [Google Java Error Handling Guide (类比)](https://research.google/pubs/pub40165/)
