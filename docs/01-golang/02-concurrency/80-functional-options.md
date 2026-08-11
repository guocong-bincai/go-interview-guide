# 函数选项模式（Functional Options Pattern）

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：Functional Options、Option 接口、可变参数设计、Go API 优雅设计、Builder 模式对比

## 🎯 面试官考察意图

函数选项模式是 Go 生态中最常用的 API 设计模式之一。面试官想确认：

1. 候选人是否能理解为什么 Go 没有默认参数，以及如何优雅地解决
2. 是否见过生产级的 Option 实现，而不只是理论了解
3. 能否对比 Function Options vs Builder vs Struct Tag 三种方案的优劣
4. 是否知道这个模式的局限性和适用边界

---

## ⚡ 核心答案（30秒版）

**函数选项模式通过定义 `Option` 接口和闭包，让调用方以链式调用的方式配置结构体。**

- 核心思想：`func WithTimeout(t time.Duration) Option { return func(*Server) { ... } }`
- 每个 option 是一个函数，接收并修改目标 struct 的指针
- 调用时传入多个 option：`NewServer(WithTimeout(5*time.Second), WithTLS(true))`
- **优点**：向前向后兼容、链式调用可读性好、零依赖
- **缺点**：option 之间不能共享临时状态、不支持复杂条件逻辑

---

## 🔬 深度展开

### 1. 问题的起源

Go 没有默认参数（default arguments），也没有方法重载（method overloading）。当你的结构体有越来越多可选字段时，构造函数会变得非常尴尬：

```go
// ❌ 糟糕的设计：太多参数，顺序容易错
type Server struct {
    Host     string
    Port     int
    Timeout  time.Duration
    MaxConn  int
    Debug    bool
    LogLevel string
    TLS      bool
}

func NewServer(host string, port int, timeout time.Duration, ...) *Server {
    // 调用者很难记住参数的正确顺序
    s := &Server{Host: host, Port: port}
    if timeout > 0 { s.Timeout = timeout }
    // ...
    return s
}
```

### 2. 基础实现

```go
package server

import (
    "fmt"
    "time"
)

// Server 是我们想要创建的服务器
type Server struct {
    host     string
    port     int
    timeout  time.Duration
    maxConn  int
    debug    bool
    tls      bool
    logger   string
}

func (s *Server) String() string {
    return fmt.Sprintf("Server{%s:%d timeout=%v maxConn=%d debug=%v tls=%v}",
        s.host, s.port, s.timeout, s.maxConn, s.debug, s.tls)
}

// Option 接口：所有配置项都实现这个接口
type Option interface {
    apply(*Server)
}

// serverConfigurator 实现了 Option 接口
type serverConfigurator struct {
    fn func(*Server)
}

func (c *serverConfigurator) apply(s *Server) {
    c.fn(s)
}

// WithHost 设置主机名
func WithHost(host string) Option {
    return &serverConfigurator{fn: func(s *Server) {
        s.host = host
    }}
}

// WithPort 设置端口
func WithPort(port int) Option {
    return &serverConfigurator{fn: func(s *Server) {
        s.port = port
    }}
}

// WithTimeout 设置超时
func WithTimeout(d time.Duration) Option {
    return &serverConfigurator{fn: func(s *Server) {
        s.timeout = d
    }}
}

// WithMaxConn 设置最大连接数
func WithMaxConn(n int) Option {
    return &serverConfigurator{fn: func(s *Server) {
        if n <= 0 {
            panic("maxConn must be positive")
        }
        s.maxConn = n
    }}
}

// WithDebug 开启调试模式
func WithDebug(on bool) Option {
    return &serverConfigurator{fn: func(s *Server) {
        s.debug = on
    }}
}

// WithTLS 开启 TLS
func WithTLS(on bool) Option {
    return &serverConfigurator{fn: func(s *Server) {
        s.tls = on
    }}
}

// NewServer 构造函数，接受任意数量的 Option
func NewServer(opts ...Option) *Server {
    // 先设置默认值
    s := &Server{
        host:    "localhost",
        port:    8080,
        timeout: 30 * time.Second,
        maxConn: 100,
        debug:   false,
        tls:     false,
    }
    
    // 依次应用所有 option
    for _, opt := range opts {
        opt.apply(s)
    }
    
    return s
}
```

使用方式：

```go
func main() {
    // ✅ 简洁明了，可以按需选择
    s1 := NewServer(
        WithHost("0.0.0.0"),
        WithPort(9090),
        WithTimeout(60*time.Second),
    )
    fmt.Println(s1) // Server{0.0.0.0:9090 timeout=1m0s maxConn=100 debug=false tls=false}

    // ✅ 完全不用任何 option → 使用默认值
    s2 := NewServer()
    fmt.Println(s2) // Server{localhost:8080 timeout=30s maxConn=100 debug=false tls=false}

    // ✅ 链式添加，灵活组合
    s3 := NewServer(
        WithHost("example.com"),
        WithPort(443),
        WithTLS(true),
        WithDebug(true),
        WithMaxConn(1000),
    )
    fmt.Println(s3) // Server{example.com:443 timeout=30s maxConn=1000 debug=true tls=true}
}
```

### 3. 进阶技巧

#### 技巧一：带条件的 Option（避免空值问题）

```go
// 只有当提供的值不为零时才设置
func WithOptionalLogger(logger *slog.Logger) Option {
    return &serverConfigurator{fn: func(s *Server) {
        if logger != nil {
            s.logger = logger.String()
        }
    }}
}
```

#### 技巧二：Option 之间的依赖与互斥

```go
// WithFastHTTP 自动启用 gzip 压缩
func WithFastHTTP(enabled bool) Option {
    return &serverConfigurator{fn: func(s *Server) {
        s.debug = enabled // FastHttp 模式下强制开启 debug
    }}
}

// 在 constructor 中处理互斥关系
func NewServer(opts ...Option) *Server {
    s := &Server{ /* defaults */ }
    
    for _, opt := range opts {
        opt.apply(s)
    }
    
    // 后处理的互斥逻辑
    if s.tls && !s.debug {
        // TLS 生产环境通常不需要 debug
        slog.Warn("TLS is enabled but debug is off")
    }
    
    return s
}
```

#### 技巧三：Grouped Options（批量配置）

```go
// 预设组合选项
func ProductionOptions() []Option {
    return []Option{
        WithTimeout(60 * time.Second),
        WithMaxConn(5000),
        WithTLS(true),
    }
}

func DevelopmentOptions() []Option {
    return []Option{
        WithDebug(true),
        WithTimeout(5 * time.Second),
    }
}

// 使用：基于预设 + 额外覆盖
s := NewServer(
    ProductionOptions()...,
    WithHost("prod.example.com"),
)
```

### 4. 对比其他方案

#### vs Builder 模式

```go
// Builder 模式
type ServerBuilder struct {
    server *Server
}

func NewServerBuilder() *ServerBuilder {
    return &ServerBuilder{server: &Server{}}
}

func (b *ServerBuilder) WithHost(host string) *ServerBuilder {
    b.server.host = host
    return b
}

func (b *ServerBuilder) Build() *Server {
    return b.server
}

// 调用：NewServerBuilder().WithHost("x").WithPort(80).Build()
```

| 维度 | Functional Options | Builder |
|------|--------------------|---------|
| 代码量 | 少 | 多（需要 builder struct） |
| 链式调用 | ✅ | ✅ |
| 支持不可变对象 | ❌（需要指针） | ✅（可以返回新对象） |
| 内存开销 | 无额外 alloc（内联优化） | 需要 builder struct |
| 适用场景 | 简单配置 | 复杂构建逻辑、条件分支 |
| Go 惯用法 | ✅✅✅ | ✅ |

**结论**：大多数情况下 Functional Options 更简洁，Go 社区广泛采用。Builder 适用于构建逻辑复杂、有大量中间状态的场景（如 SQL query builder）。

#### vs Struct Tag / YAML Config

```go
// Struct tag 方案
type ServerConfig struct {
    Host     string        `yaml:"host"`
    Port     int           `yaml:"port"`
    Timeout  time.Duration `yaml:"timeout"`
}

cfg := ServerConfig{}
yaml.Unmarshal(data, &cfg)
s := NewServerFromConfig(cfg)
```

| 维度 | Functional Options | Struct Tag |
|------|--------------------|-----------|
| 编译期检查 | ✅ 类型安全 | ❌ 运行时解析 |
| IDE 支持 | ✅ autocomplete | ❌ |
| 灵活性 | 高（可加自定义验证） | 低（纯数据映射） |
| 适用场景 | 代码中配置 | 外部配置文件 |

### 5. 标准库中的类似模式

虽然 `net/http` 没有用完整的 Option 模式，但有一些类似的实践：

```go
// http.Client 使用 struct 直接配置
client := &http.Client{
    Timeout:   30 * time.Second,
    Transport: customTransport,
}

// log/slog.HandlerOptions 也是类似的思路
handler := slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
})

// database/sql.DB 的配置也类似
db, err := sql.Open("postgres", connStr)
db.SetMaxOpenConns(100)
db.SetMaxIdleConns(10)
```

这说明 Go 标准库的风格倾向：简单结构体 + setter 方法，或者直接用 struct 初始化。Functional Options 模式更多出现在第三方库（gin、kratos、etcd client 等）。

### 6. 常见陷阱

#### 陷阱一：Option 的执行顺序依赖

```go
// 如果 WithTLS 需要在 WithHost 之前执行才能正常工作
s := NewServer(
    WithHost("example.com"),
    WithTLS(true),
)
// 这没问题，但如果某个 option 依赖另一个 option 的结果...
```

**解决方案**：保持 option 独立，不要制造隐式依赖。如果需要跨-option 协调，在 constructor 的最后统一处理。

#### 陷阱二：忘记处理零值

```go
// ❌ 如果用户传了 WithTimeout(0)，会怎么行为？
// 可能需要特殊处理
func WithTimeout(d time.Duration) Option {
    return &serverConfigurator{fn: func(s *Server) {
        if d == 0 {
            return // 跳过零值
        }
        s.timeout = d
    }}
}
```

#### 陷阱三：过多的 Option 导致调用冗长

```go
// ❌ 这太长了，说明需要重新思考设计
NewServer(
    WithHost("x"), WithPort(80), WithTimeout(...),
    WithMaxConn(100), WithDebug(false), WithTLS(true),
    WithLogger(l), WithRateLimit(r), WithMetrics(m),
    WithMiddleware(h1, h2), WithGRPC(g),
)

// ✅ 用 Grouped Options 分组
s := NewServer(
    BasicNetworkOpts(),
    AdvancedFeatures(),
    WithHost("api.example.com"),
)
```

---

## 💻 完整生产示例

```go
package config

import (
    "fmt"
    "net"
    "time"
)

// --- Type Definitions ---

type Server struct {
    addr          string
    readTimeout   time.Duration
    writeTimeout  time.Duration
    idleTimeout   time.Duration
    maxHeaderBytes int
    tlsConfig     *tlsConfig
    middleware    []Middleware
}

type Middleware func(http.Handler) http.Handler

type tlsConfig struct {
    certFile string
    keyFile  string
}

type Option interface{ apply(*Server) }

type optFunc func(*Server)

func (f optFunc) apply(s *Server) { f(s) }

// --- Option Functions ---

func WithAddr(addr string) Option {
    return optFunc(func(s *Server) { s.addr = addr })
}

func WithTimeouts(read, write, idle time.Duration) Option {
    return optFunc(func(s *Server) {
        s.readTimeout = read
        s.writeTimeout = write
        s.idleTimeout = idle
    })
}

func WithMaxHeaders(n int) Option {
    return optFunc(func(s *Server) {
        if n > 0 {
            s.maxHeaderBytes = n
        }
    })
}

func WithTLS(certFile, keyFile string) Option {
    return optFunc(func(s *Server) {
        s.tlsConfig = &tlsConfig{certFile: certFile, keyFile: keyFile}
    })
}

func WithMiddleware(mw ...Middleware) Option {
    return optFunc(func(s *Server) {
        s.middleware = append(s.middleware, mw...)
    })
}

// --- Constructor ---

func NewServer(opts ...Option) *Server {
    s := &Server{
        addr:           ":8080",
        readTimeout:    15 * time.Second,
        writeTimeout:   15 * time.Second,
        idleTimeout:    60 * time.Second,
        maxHeaderBytes: 1 << 20, // 1MB
    }
    for _, o := range opts {
        o.apply(s)
    }
    return s
}

func (s *Server) ListenAndServe() error {
    l, err := net.Listen("tcp", s.addr)
    if err != nil {
        return err
    }
    fmt.Printf("Server listening on %s\n", s.addr)
    _ = l.Close()
    return nil
}

// --- Presets ---

func ProdDefault() []Option {
    return []Option{
        WithTimeouts(30*time.Second, 30*time.Second, 120*time.Second),
        WithMaxHeaders(1 << 20),
    }
}

func DevDefault() []Option {
    return []Option{
        WithTimeouts(5*time.Second, 5*time.Second, 30*time.Second),
    }
}
```

---

## 🗣️ 面试话术

> "函数选项模式的核心思想是用一个 Option 接口和闭包来封装配置的变更逻辑。每个 WithXXX 函数返回一个 Option，构造函数接收若干个 Option 后依次应用到默认实例上。优点是调用清晰、向前兼容——加新 option 不影响老代码。缺点是 option 之间不能有依赖关系，太复杂的配置逻辑不适合这种方式。Go 标准库里没有原生实现这个模式，但 gin、kratos、etcd 等主流项目都在用。我个人觉得它比 Builder 模式更简洁，适合配置类场景。"

> "一句话：用 Option 接口把 '如何配置' 封装成一等公民，让调用者组合它们而不是 memorize 参数顺序。"

<div align="right">
<i>最后更新：2026-08-12 ｜ 模块：Go 语言深度 · 并发编程</i>
</div>
