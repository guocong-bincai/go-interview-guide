[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [📚 标准库](../README.md)

---

# Graceful Shutdown：优雅关闭 HTTP Server

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：http.Server.Shutdown、信号处理、上下文取消、连接 draining、优雅退出模式

## 🎯 面试官考察意图

服务部署（尤其 Kubernetes）中 Graceful Shutdown 是高频实战考点。面试官想确认：

1. 是否知道 `http.Server.Shutdown()` 的正确调用方式
2. 能否设计完整的优雅退出流程（接收信号 → 停止接受新请求 → 等待进行中请求完成 → 清理资源）
3. 是否了解 K8s PodTerminationGracePeriodSeconds 与 ShutDownTimeout 的配合
4. 进阶：是否能处理数据库连接池关闭时机等依赖资源的协调关闭

---

## ⚡ 核心答案（30秒版）

**优雅关闭 = 停止接受新连接 + 等待活跃请求完成 + 释放下游依赖。**

核心 API 是 `http.Server.Shutdown(ctx)`，它不会立即终止，而是先关闭监听器不再接受新连接，然后等待所有正在处理的 HTTP 请求完成（最多等到 ctx 超时）。必须在 goroutine 中接收 SIGTERM/SIGINT 信号并调用 Shutdown()，同时在 Shutdown 之前用 context.WithTimeout 确保最终兜底强制退出。

---

## 🔬 深度展开

### 1. 标准优雅关闭模式

```go
func main() {
    // 1. 创建 server
    srv := &http.Server{
        Addr:         ":8080",
        Handler:      myHandler,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // 2. 在 goroutine 中启动（避免阻塞）
    go func() {
        fmt.Println("Server starting on :8080")
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            log.Fatalf("listen failed: %v", err)
        }
    }()

    // 3. 等待退出信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit // 阻塞直到收到 SIGINT 或 SIGTERM

    fmt.Println("Shutting down server...")

    // 4. 创建带超时的 context（给进行中请求的缓冲时间）
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // 5. 优雅关闭（draining 阶段）
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("server forced to shutdown: %v", err)
    }

    fmt.Println("Server exited properly")
}
```

关键要点：
- **ListenAndServe 和 Shutdown 必须分开调用**——如果用了 `ListenAndServeTLS` 等变体，需要先拿到 Listener 再调用 `Serve(ln)`
- **`http.ErrServerClosed` 是正常退出，不应视为错误**
- **Shutdown timeout 要大于业务最长请求的处理时间**

### 2. Shutdown 的完整生命周期

```
SIGTERM 到达
    │
    ├─→ srv.Shutdown(ctx)
    │       │
    │       ├─ ① 关闭 listener（不再 accept 新连接）
    │       │   已建立的 TCP 连接保持存活
    │       │
    │       ├─ ② 等待 active requests 完成
    │       │   - 每个请求在其 context 被标记为 canceled
    │       │   - handler 应当检查 ctx.Done() 并提前返回
    │       │
    │       └─ ③ 超时后强制关闭
    │           - 剩余未完成的请求被中断
    │           - 如果有 err 则返回
    │
    └─→ 继续执行关闭后续资源（DB 连接池、cache 连接等）
```

### 3. 常见陷阱

#### 陷阱一：handler 不感知 context 取消

```go
// ❌ handler 永远不会因为 server 关闭而结束
func badHandler(w http.ResponseWriter, r *http.Request) {
    result := longRunningQuery()  // 没有检查 ctx！
    json.NewEncoder(w).Encode(result)
}

// ✅ handler 应该随时检查 ctx
func goodHandler(w http.ResponseWriter, r *http.Request) {
    // r.Context() 会被 Shutdown 自动取消
    select {
    case result := <-longRunningQuery(r.Context()):
        json.NewEncoder(w).Encode(result)
    case <-r.Context().Done():
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(map[string]string{"error": "request cancelled"})
    }
}
```

#### 陷阱二：Shutdown timeout 设置过短

```go
// ❌ 太短，大量请求会被强制中断
ctx, _ := context.WithTimeout(context.Background(), 1*time.Second)
srv.Shutdown(ctx)

// ✅ 设置为业务最长请求时间的 1.5~2 倍
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

### 4. 多服务依赖的关闭顺序

实际系统中 HTTP Server 通常依赖 DB、Cache、消息队列等。正确的关闭顺序：

```go
func gracefulShutdown(srv *http.Server, dbPool *sql.DB, cache *redis.Client) error {
    // Step 1: 先关闭 HTTP（拒绝新请求）
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("HTTP shutdown error: %v", err)
    }

    // Step 2: 再关闭下游依赖（DB → Cache）
    dbPool.Close()    // 断开数据库连接
    cache.Close()     // 断开 Redis 连接
    
    return nil
}
```

关闭原则：**先关入口（HTTP），再关上游依赖（DB/Cache），最后关自身**。反过来开服务时则是相反的顺序。

---

## 🗣️ 面试话术

- **初级**："用 `signal.Notify` 接收 SIGTERM，然后调用 `http.Server.Shutdown(ctx)` 优雅退出。"
- **中级**："Shutdown 会先停止监听，再等待活跃请求完成（context 自动取消）。关键是 handler 里要检查 ctx.Done()，timeout 要设得合理。"
- **高级**："K8s 环境中 PodTerminationGracePeriodSeconds 默认 30s，所以 Shutdown timeout 应略小于这个值。多服务场景要先关 HTTP 入口再关 DB 连接池。对于长连接 WebSocket，需要在 client 侧配合做重连逻辑。"

---

## 🔗 关联阅读

- [http.Server 底层结构](./22-01-net-http.md)
- [context 原理](../02-concurrency/42-05-context.md)
- [Web Framework 对比](./28-10-web-framework-comparison.md)
