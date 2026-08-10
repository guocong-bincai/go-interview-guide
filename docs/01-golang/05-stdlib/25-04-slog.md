# Go log/slog 结构化日志详解

> 考察频率：★★★★☆  优先级：P1（高频加分项）

---

## 1. 面试官考察意图

考察候选人对 Go 1.21+ 标准库新特性的掌握，以及工程化日志实践的理解。
初级只知道 `log.Printf`，高级能讲清 **slog 的设计理念、与 zap/logrus 的对比、日志级别/Handler/Context 链路的工程价值**。这道题同时也在考察工程化思维——日志是线上问题的第一手信息来源，结构化日志能力直接反映候选人的生产经验。

---

## 2. 核心答案（30 秒版）

`slog` 是 Go 1.21 引入的标准库结构化日志，核心是 **key-value 日志** + **Handler 分离**。
- **默认 logger**：输出类似 `log`，但带级别 `INFO/WARN/ERROR`
- **TextHandler**：输出 `key=value` 格式，便于 grep/解析
- **JSONHandler**：输出 JSON 行，方便日志收集系统（ELK/Loki）接入
- **自定义 Handler**：可实现日志路由、过滤、脱敏
- **Logger.With**：构建带固定字段的子 logger（如 traceID、userID）

slog 的优势是**零依赖**（不需引入 zap/logrus），与 `log` 包兼容（`log.SetOutput` 可指向 slog）。

---

## 3. 深度展开

### 3.1 为什么需要结构化日志

传统 `log.Printf("user %s login at %s", name, time)` 的问题：
- 无法按字段搜索（如 `level=ERROR AND service=order`）
- 不同服务格式不一致，下游解析困难
- JSON 输出需要引入第三方库

结构化日志：
```
time=2026-04-23T15:46:00Z level=INFO msg="user login" user_id=12345 service=order
```
可被任意日志系统解析，支持索引、过滤、聚合。

### 3.2 slog 三种 Handler

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    // 1. 默认 Handler（类似 log.Printf，追加 level/msg）
    slog.Info("hello, world")

    // 2. TextHandler（key=value 格式，适合人类阅读）
    logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
    logger.Info("order placed", "order_id", 12345, "amount", 99.9)

    // 输出：time=2026-04-23T15:46:00Z level=INFO msg="order placed" order_id=12345 amount=99.9

    // 3. JSONHandler（机器可读，适合 ELK/Loki/Graylog）
    jsonLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    jsonLogger.Error("database connection failed",
        "host", "db.prod.internal",
        "port", 5432,
        "error", "connection timeout",
    )
    // 输出：{"time":"2026-04-23T15:46:00Z","level":"ERROR","msg":"database connection failed","host":"db.prod.internal","port":5432,"error":"connection timeout"}
}
```

### 3.3 自定义 Handler 示例（生产级）

```go
// 生产级需求：脱敏 + 添加 traceID + 按级别路由
type ProdHandler struct {
    next   slog.Handler
    traceID func() string // 从 context 提取
}

func (h *ProdHandler) Handle(ctx context.Context, r slog.Record) error {
    // 自动注入 traceID
    r.AddAttrs(slog.String("trace_id", h.traceID()))

    // 脱敏：敏感字段打码
    r.Message = desensitize(r.Message)

    return h.next.Handle(ctx, r)
}

func desensitize(msg string) string {
    // 简单示例：实际生产用正则
    return msg // 业务逻辑
}

func NewProdHandler(next slog.Handler, traceID func() string) *ProdHandler {
    return &ProdHandler{next: next, traceID: traceID}
}

// 使用
handler := NewProdHandler(slog.NewJSONHandler(os.Stdout, nil), func() string {
    return "abc123" // 从 context 提取
})
logger := slog.New(handler)
logger.Info("payment callback", "amount", 1000)
```

### 3.4 Logger.With —— 构建上下文 Logger

```go
// 典型用法：每个请求初始化一个带 traceID 的 logger
func handleRequest(ctx context.Context) {
    traceID := ctx.Value("trace_id").(string)
    logger := slog.Default().With("trace_id", traceID, "user_id", getUserID(ctx))

    logger.Info("request started")  // 自动带上 trace_id + user_id
    // ...
    logger.Error("request failed", "error", err)
}
```

### 3.5 与 zap/logrus 的对比

| 特性 | slog（标准库） | zap | logrus |
|------|--------------|-----|--------|
| 依赖 | **零依赖** | 需引入 | 需引入 |
| 性能 | 中等（JSON 编码非并行）| 最高（零分配）| 较低 |
| 上手 | 最简单 | 中等 | 简单 |
| 生态 | 逐渐流行（大厂迁移中）| 成熟 | 成熟 |
| 适用场景 | 中小型服务、微服务 | 超高 QPS 服务 | 需要丰富 hook |

**面试加分回答**：Uber 已将 zap 的设计理念贡献给 slog，未来标准库会越来越强。建议新项目直接用 slog，老项目可逐步迁移。

### 3.6 slog 与 log 包兼容

```go
import "log"

// slog.SetDefault 会同时设置 log 包的输出
slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stderr, nil)))

// 后续 log.Printf 也会走 JSONHandler
log.Printf("legacy code using log")  // 实际走 slog JSON 输出
```

---

## 4. 高频追问

### Q1：slog 的日志级别如何配置运行时切换？

```go
// HandlerOptions 可设置 MinLevel
opts := &slog.HandlerOptions{
    Level: slog.LevelDebug, // 开发环境全开
}
logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))

// 生产环境通过环境变量或配置中心动态调整
```

### Q2：slog 能否异步输出？

slog 本身不提供异步 Handler，但可结合 `channel` 或 `io.Writer` 封装：

```go
type AsyncHandler struct {
    ch  chan slog.Record
    fn  func([]slog.Record)
}

func (h *AsyncHandler) Handle(ctx context.Context, r slog.Record) error {
    select {
    case h.ch <- r:
    default:
        // 队列满，丢日志（不要阻塞）
    }
    return nil
}
```

### Q3：slog 的 JSON 输出性能如何？

实测低于 zap（无 zero-allocation 优化），但对于大多数服务（QPS < 10万）足够。如果极端追求性能，建议用 `slog.JSONHandler` + `zap.Sync()` 混合方案，或继续使用 zap。

---

## 5. 延伸阅读

- [Go slog 官方文档](https://pkg.go.dev/log/slog)
- [Structured Logging with slog - Go Blog](https://go.dev/blog/slog)
- [Uber zap 设计原理](https://github.com/uber-go/zap)

---

*版本：v2.42 → v2.43 | 新增 log/slog 结构化日志专题*