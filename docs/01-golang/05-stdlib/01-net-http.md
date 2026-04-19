# Go net/http 深度解析

> 考察频率：★★★★☆  优先级：P1

---

## 1. http.Server 底层结构

### 面试官考察意图

考察候选人对 Go HTTP 服务器底层机制的理解深度。初级工程师只会用 `ListenAndServe`，高级工程师知道连接是怎么管理的、goroutine 是怎么分配的、什么时候会阻塞。这道题是考察是否真正理解 Go 并发模型的重要切入点。

### 核心答案（30秒版）

Go 的 HTTP Server 用的是**goroutine-per-connection 模型**：每个 TCP 连接分配一个 goroutine 处理（`serve()` → `serveConn()`），而不是传统的 epoll 单线程循环。这意味着 Go 的 HTTP 处理是天然并发的，瓶颈在 goroutine 调度而不是 IO 多路复用。

### 深度展开

#### 核心结构

```go
type Server struct {
    Addr         string
    Handler      Handler  // 请求处理器
    ReadTimeout  time.Duration  // 读请求超时
    WriteTimeout time.Duration  // 写响应超时
    IdleTimeout  time.Duration  // 空闲连接超时
    MaxHeaderBytes int

    // 内部结构
    mu     sync.Mutex
    listeners map[*net.Listener]struct{}
    doneChan  chan struct{}
}
```

#### 连接处理流程

```
TCP连接建立
    │
    ▼
net.Listen("tcp", ":8080")
    │
    ▼
for {
    conn, err := listener.Accept()  // 阻塞等待
    │
    ▼
    go c.serve(conn)  // 每个连接一个goroutine（goroutine-per-connection）
}
```

#### goroutine-per-connection vs epoll

| 模型 | 代表技术 | 适用场景 |
|------|---------|---------|
| **goroutine-per-connection** | Go HTTP Server | 中低并发（万级连接） |
| **单线程 epoll 循环** | Nginx / Java NIO | 超高并发（十万级连接） |
| **多线程 epoll** | Redis | 高并发+计算密集 |

Go 的优势：每个连接一个 goroutine，代码写起来像同步，但实际是并发的。不需要开发者关心 epoll 细节。

#### 真实踩坑：连接数上限

```go
// 问题：默认情况下，单机 Go HTTP 服务器能处理多少并发连接？
// 答案：受限于：1）系统文件描述符上限；2）内存

// 查看系统限制
$ ulimit -n  // 通常是1024

// 在Go中设置
func main() {
    srv := &http.Server{Addr: ":8080", Handler: mux}

    // 方式1：启动前设置
    // $ ulimit -n 65535

    // 方式2：Go代码中无法直接改系统限制，但可以做连接数监控
    go func() {
        for {
            fmt.Println("当前连接数:", runtime.NumGoroutine())
            time.Sleep(5 * time.Second)
        }
    }()

    srv.ListenAndServe()
}
```

---

## 2. Transport 连接池

### 面试官考察意图

考察候选人对 HTTP 客户端资源管理的理解。`http.Client` 背后有一个连接池，如果不了解它的工作机制，容易写出连接泄漏或资源耗尽的代码。高级工程师知道什么时候配置连接池参数，以及为什么默认配置不一定适合生产。

### 核心答案（30秒版）

`http.Transport` 维护了一个**连接池**：MaxIdleConns 控制空闲连接上限，MaxIdleConnsPerHost 控制每个 host 的空闲连接数。默认配置下，连接会被**复用**而不是每次请求新建。生产环境要根据 QPS 和目标服务能力调参。

### 深度展开

#### Transport 核心配置

```go
transport := &http.Transport{
    // 空闲连接池大小
    MaxIdleConns:        100,   // 全局最大空闲连接数
    MaxIdleConnsPerHost: 10,    // 每个Host的最大空闲连接数（关键！）

    // 连接建立
    DialContext: (&net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
    }).DialContext,

    // 超时配置
    IdleConnTimeout:       90 * time.Second,  // 空闲连接多久关闭
    ResponseHeaderTimeout: 5 * time.Second,    // 读取响应头超时
}
client := &http.Client{Transport: transport}
```

#### 连接复用原理

```
请求1: GET /api/users
        │
        ├── 第一次连接 → Dial → Handshake → TLS → 创建连接
        │
        └── 完成 → 放入空闲池（MaxIdleConnsPerHost上限）

请求2: GET /api/orders（同一Host）
        │
        └── 从空闲池取到连接 → 复用！无需新建TCP
```

#### 常见连接泄漏问题

```go
// ❌ 连接不读取body就关闭（连接无法复用）
resp, _ := client.Get("http://example.com/api")
body := resp.Body.Close() // 只关闭body，但底层连接被浪费

// ✅ 正确做法：完全读取body后再关闭
resp, _ := client.Get("http://example.com/api")
defer resp.Body.Close()
io.Copy(ioutil.Discard, resp.Body) // 把body读干净

// 或者使用 io.ReadAll
data, _ := io.ReadAll(resp.Body)
```

#### 连接池耗尽问题

```go
// 问题场景：高QPS + 慢响应 + 连接池太小 = 连接池耗尽
// 表现：后续请求超时

// 场景：1000 QPS，每个请求耗时1秒，MaxIdleConns=100
// → 100个连接在用，100个空闲，800个请求在等 → 超时

// ✅ 解决方案：调大MaxIdleConns + MaxIdleConnsPerHost
transport := &http.Transport{
    MaxIdleConns:        1000,
    MaxIdleConnsPerHost: 100, // 关键：每个host至少100个空闲连接
}
```

---

## 3. Handler 接口与 ServeMux 路由

### 面试官考察意图

考察候选人对 Go HTTP 框架本质的理解。`http.Handler` 接口只有 `ServeHTTP(ResponseWriter, *Request)` 一个方法，极其简洁。面试官想看你能不能说清楚：路由匹配规则、handler 链如何实现、以及为什么标准库没有提供路由功能。

### 深度展开

#### Handler 接口

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// 任何函数只要签名匹配，都是Handler（适配器模式）
type HandlerFunc func(w ResponseWriter, r *Request)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r) // 调用自身
}
```

#### DefaultServeMux 路由规则

```go
// 注册到默认路由
func handleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}

// 路由匹配规则（注意：不是前缀匹配，是完整路径匹配）
http.HandleFunc("/api/users", handler)
// /api/users    ✓ 匹配
// /api/users/1  ✗ 不匹配（这是另一个路径）
// /api          ✗ 不匹配
```

#### 路由匹配示例

```go
// 标准库 ServeMux 的匹配规则：
// 1. 完全匹配优先
// 2. 静态路径不会匹配有尾随斜线的

// 示例：
mux := http.NewServeMux()
mux.HandleFunc("/api/", handler1)       // /api/ 开头
mux.HandleFunc("/api/users", handler2) // 精确匹配

// 请求 /api/users → handler2
// 请求 /api/profile → handler1
```

#### 中间件链式调用

```go
// 中间件就是一个返回 Handler 的函数
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        fmt.Printf("%s %s %v\n", r.Method, r.URL.Path, time.Since(start))
    })
}

// 链式组合
handler := myHandler
handler = loggingMiddleware(handler)
handler = authMiddleware(handler)
handler = tracingMiddleware(handler)

// 启动服务
http.ListenAndServe(":8080", handler)
```

---

## 4. 超时配置

### 面试官考察意图

考察候选人对 HTTP 超时体系的理解。超时不只是「请求不能太慢」，更重要的是**防止资源泄漏**。如果服务处理请求时调用了阻塞的外部资源但没设超时，连接会一直占用直到客户端断开，导致连接池耗尽。

### 核心答案（30秒版）

Go HTTP Server 有三层超时：**ReadTimeout**（读取请求的时间）、**WriteTimeout**（写响应的时间）、**IdleTimeout**（空闲连接保留时间）。配置原则是：读超时设短、写超时设长、外部调用必须用 context 超时。

### 深度展开

#### 三层超时

```go
srv := &http.Server{
    Addr: ":8080",
    Handler: mux,

    // 读取请求头+body的总时间（建议5~10秒）
    ReadTimeout: 10 * time.Second,

    // 从请求读完到响应写完的时间（建议30~60秒）
    // 如果处理逻辑耗时长（DB查询、文件IO），这个值要够大
    WriteTimeout: 30 * time.Second,

    // 空闲连接存活时间（建议90秒）
    // 连接处理完一个请求后，如果这个时间内没新请求，就关闭
    IdleTimeout: 90 * time.Second,
}
```

#### 实际踩坑案例

```go
// 问题：WriteTimeout 设了 5 秒，但业务逻辑中有 30 秒的 DB 查询
// 结果：请求处理到一半被强制断开，客户端收到 connection reset

// ✅ 正确做法
srv := &http.Server{
    WriteTimeout: 60 * time.Second, // 要容纳最长业务逻辑
}
```

#### HTTP 客户端超时

```go
client := &http.Client{
    Timeout: 30 * time.Second, // 整体请求超时（包含连接+读+写）

    // 或分组件配置
    Transport: &http.Transport{
        DialContext: (&net.Dialer{
            Timeout: 5 * time.Second, // 建连超时
        }).DialContext,

        ResponseHeaderTimeout: 10 * time.Second, // 读响应头超时
    },
}

// 外部调用要额外设 context 超时
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
req = req.WithContext(ctx)
resp, err := client.Do(req)
```

---

## 5. 优雅关闭：Shutdown vs Close

### 面试官考察意图

考察候选人对服务稳定发布流程的理解。`Close` 会立即关闭所有连接（强制断开），`Shutdown` 会等待正在处理的请求完成。K8s 滚动更新依赖 Shutdown，候选人必须知道两者的区别和正确的使用方式。

### 核心答案（30秒版）

`Close()` 立即关闭监听端口和所有活跃连接，会导致正在处理的请求直接失败；`Shutdown()` 先停止接收新连接，等待现有请求处理完再退出。生产环境发布必须用 `Shutdown`，配合 `WaitGroup` 确保请求处理完。

### 深度展开

```go
// ❌ Close：强制关闭，正在处理的请求会 connection reset
func badShutdown() {
    srv.Close() // 立即断开所有连接
}

// ✅ Shutdown：优雅关闭
func goodShutdown() {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // 1. 停止接收新连接
    // 2. 等待现有连接处理完（在超时前）
    // 3. 退出
    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("shutdown error: %v", err)
    }
}
```

---

## 高频追问

**Q：如何实现一个简单的 HTTP 框架中间件？**

```go
// 中间件 = 函数，返回 http.Handler
// 核心：包装 next.ServeHTTP(w, r)

func RateLimitMiddleware(limiter *rate.Limiter, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !limiter.Allow() {
            http.Error(w, "rate limit", 429)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// Gin/Echo 风格
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v", err)
                http.Error(w, "internal error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

**Q：HTTP/2 和 HTTP/1.1 在 Go 中的性能差异？**

> HTTP/2 多了多路复用（一个 TCP 连接上多个请求并发）和头部压缩（HPACK）。在 Go 中启用 HTTP/2 只需要服务器和客户端都支持即可（Go 默认优先 HTTP/2）。对于高并发小请求场景，HTTP/2 可以减少 TCP 握手开销，提升性能。

---

## 延伸阅读

- [Go net/http 标准库源码](https://github.com/golang/go/blob/master/src/net/http/server.go)
- [Go http.Transport 详解 - Alex Ellis](https://alexstocks.github.io/blog/golang_http/)
- [HTTP/2 in Go](https://go.dev/blog/http2)
- [Go HTTP 服务平滑重启 - 深度解析](https://grisha.org/blog/2014/06/03/graceful-restart-in-golang/)
