# 结构化日志工程化：log/slog 生产落地

> 考察频率：★★★★☆  优先级：P1
> 关键词：log/slog、Handler、TraceID 透传、日志采样、脱敏、Zap/Zerolog 对比、每请求 Logger

## 面试官考察意图

"你们日志怎么打的？为什么用 Zap 不直接用标准库？"——看似基础，实则考察**工程素养**。Go 1.21 引入标准库 `log/slog` 后，这道题的答案变了：候选人是否知道**标准库现在也能高性能结构化日志**，以及生产落地的关键细节：

1. `slog` 的 **Handler / Logger / Record / Attr** 模型；
2. **TraceID 全链路透传**（从 HTTP middleware 到 DB 查询）；
3. **日志采样**（高频日志降噪）与**脱敏**（手机号/身份证不能进日志）；
4. 与 **Zap/Zerolog 的选型**（性能、生态、API 稳定性）。

**核心立场：日志是给人排障用的，不是给机器看的流水——结构化 + 带上下文 + 可分级采样，才是生产级日志。**

---

## 核心答案（30 秒版）

**`log/slog`（Go 1.21+）** 提供**结构化日志**：键值对（`Attr`）而非拼字符串，输出 JSON / text 由 `Handler` 决定。

```go
slog.InfoContext(ctx, "order paid",
    "order_id", orderID,
    "amount", amount,
    "user_id", uid,
)
// {"level":"INFO","msg":"order paid","order_id":"...","amount":123,"user_id":"..."}
```

**四个核心对象**：
- `Logger`：入口，提供 `Info/Debug/Error/With/Log`；
- `Handler`：真正决定**怎么写出**（JSON / Text / 自定义）；
- `Record`：一条日志（时间、级别、消息、attrs）；
- `Attr`：`slog.String("k", v)` 等键值对。

**生产落地四件事**：
1. **请求级 Logger 注入 context**：`ctx = context.WithValue(ctx, logKey, reqLog)`，业务层 `slog.InfoContext` 自动带上 trace_id / request_id；
2. **`slog.SetDefault` 全局配置**：级别、JSON handler、输出源；
3. **采样降噪**：高频日志（如每请求一条 info）做比例采样或只在 error 时输出；
4. **脱敏**：手机号、身份证、密码、token 用 `ReplaceAttr` 或自定义类型屏蔽。

**一句话**：**日志的价值 = 结构化（可检索）+ 上下文（能串联）+ 分级采样（不淹没用信息）+ 脱敏（不泄漏）。**

---

## 深度展开

### 一、slog 最小可用配置（生产模板）

```go
func setupLogger(level slog.Level, jsonOutput bool) *slog.Logger {
    opts := &slog.HandlerOptions{
        Level: level,
        // 统一添加公共字段
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            // 👇 脱敏：手机号等敏感字段
            if a.Key == "phone" || a.Key == "id_card" {
                return slog.String(a.Key, "***")
            }
            // 统一时间格式（RFC3339）
            if a.Key == slog.TimeKey && len(groups) == 0 {
                return slog.String(a.Key, a.Value.Time().Format(time.RFC3339))
            }
            return a
        },
    }
    var h slog.Handler
    if jsonOutput {
        h = slog.NewJSONHandler(os.Stdout, opts) // 生产用 JSON，方便采集
    } else {
        h = slog.NewTextHandler(os.Stdout, opts) // 本地开发可读性好
    }
    return slog.New(h)
}
```

### 二、TraceID 全链路透传（最高频考点）

**错误示范**：每层手动传 logger，链路过长后代码全是 `logger.Info`。

**正确做法**：把 logger 放进 `context.Context`，业务层用 `InfoContext`。

```go
type ctxKey struct{}

func WithLogger(ctx context.Context, l *slog.Logger) context.Context {
    return context.WithValue(ctx, ctxKey{}, l)
}

func LoggerFrom(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(ctxKey{}).(*slog.Logger); ok {
        return l
    }
    return slog.Default()
}

// HTTP 中间件：注入 trace_id / request_id
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        traceID := r.Header.Get("X-Trace-ID")
        if traceID == "" {
            traceID = uuid.NewString()
        }
        w.Header().Set("X-Trace-ID", traceID)

        reqLog := slog.Default().With(
            slog.String("trace_id", traceID),
            slog.String("method", r.Method),
            slog.String("path", r.URL.Path),
        )
        ctx := WithLogger(r.Context(), reqLog)
        reqLog.Info("request start")
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// 下游任意层：无需显式传 logger
func (s *OrderService) Pay(ctx context.Context, id string) error {
    log := LoggerFrom(ctx)
    log.InfoContext(ctx, "start pay", "order_id", id)
    return nil
}
```

> ⚠️ **坑：不要把 logger 直接塞进 struct 字段跨请求复用**。带 trace_id 的 logger 是**请求级**的，塞进单例 service 会导致 trace_id 串台。

### 三、日志采样：高频信息降噪

高频日志（如每条成功的请求都打 info）会淹没有效信息、推高日志账单。两种策略：

```go
// 策略：用 slog 的 Level + 采样。标准库没有内置采样 API，
// 但可以用 LevelHandler 或第三方 handler 实现"每 N 条记 1 条"。
type samplingHandler struct {
    slog.Handler
    every int64
    n     atomic.Int64
}

func (h *samplingHandler) Handle(ctx context.Context, r slog.Record) error {
    // 只对 Debug/Info 采样，Warn/Error 全量保留
    if r.Level >= slog.LevelWarn {
        return h.Handler.Handle(ctx, r)
    }
    if h.n.Add(1)%h.every != 0 {
        return nil // 丢弃
    }
    return h.Handler.Handle(ctx, r)
}
```

**原则**：**越高的级别越不该被采样——Error 必须全量**。Info 采样比例可配（如 1/100），Debug 默认关闭。

### 四、选型：slog vs Zap vs Zerolog

| 维度 | log/slog | uber-go/zap | rs/zerolog |
|---|---|---|---|
| 引入 | 标准库（Go 1.21+） | 第三方 | 第三方 |
| 性能 | 够用（有 alloc，可优化） | **极快**（zero-alloc 路径） | 极快 |
| API 稳定性 | 官方长期维护 | 稳定，生态成熟 | 稳定 |
| 生态 | 快速增长，很多库开始支持 | 最广 | 广 |
| 迁移/混用 | Handler 可桥接 Zap（`slog-zap`） | 可导出 slog | 可导出 slog |
| 推荐 | 新项目、追求依赖极简 | 极致性能/已有生态 | 极致性能/轻量 |

**结论**：**新项目默认用 `slog` 就够了**——标准库、生态在收敛、能通过自定义 Handler 或 `slog-zap` 适配后端。只有在**日志吞吐是瓶颈**（比如每秒几十万条）时才优先 Zap/Zerolog。

### 五、日志规范（工程约束）

- **禁止拼接字符串**：`log.Info("user " + uid + " paid")` ❌ → 用结构化字段，便于检索与聚合；
- **禁止打整个结构体**：可能带上密码字段、体积巨大 → 只打必要字段；
- **`Error` 必须带上下文**：错误对象 + 关键业务 id；
- **统一字段命名**：`trace_id`/`user_id`/`order_id` 全项目一致，否则 Kibana 检索会分裂；
- **日志级别纪律**：Error=需要人介入、Warn=可能有问题、Info=关键业务节点、Debug=排障专用（默认关）。

---

## 面试话术

> "我们用 Go 1.21 之后标准库的 `log/slog` 做结构化日志。核心是两点：一是**请求级 logger 注入 context**，在 HTTP 中间件里用 `With` 挂上 trace_id、request_id，业务层统一走 `InfoContext`，这样一条链路的所有日志能自动串起来，也不用一层层手动传 logger；二是配了 `ReplaceAttr` 做敏感字段脱敏，比如手机号、身份证。输出生产用 JSON 方便采集，本地用 Text 方便看。对于高频日志我们会做采样降噪——Warn/Error 全量保留，Info 按比例采样，避免日志淹没有效信息。选型上标准库够用，只有在日志吞吐是瓶颈时才会考虑 Zap。"

---

## 高频追问

- **Q：slog 和 Zap 性能差多少？** A：Zap 的 hot path 能做到接近零分配，比默认 slog 快数倍；但 slog 可通过自定义 Handler 优化，多数业务量级不是瓶颈。
- **Q：怎么保证 trace_id 不串台？** A：logger 必须请求级创建，通过 context 传递；绝不能把带 trace_id 的 logger 存进单例。
- **Q：日志能不能打整个请求体？** A：不能，体积大且可能含敏感信息；只打关键字段，大 body 走单独的采样/trace 存储。
- **Q：日志和 Trace 的分工？** A：Trace 定位"哪个环节慢/错"，日志看"具体上下文"，Metric 看"趋势和告警"，三者用 trace_id 串联。
