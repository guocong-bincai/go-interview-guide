# 连接池原理与调优：database/sql、Redis、gRPC

> 考察频率：★★★★★  优先级：P0
> 关键词：MaxOpenConns、连接泄漏、WaitCount、PoolSize、keepalive

---

## 1. 连接池的本质

### 1.1 为什么需要连接池

一次完整的数据库连接建立过程，远不止"发个请求"这么简单。以 MySQL 为例：

```
客户端                              服务器
  │                                   │
  │──── TCP 三次握手 ─────────────────→│  ~1ms
  │←──── SYN + ACK ───────────────────│
  │──── ACK ──────────────────────────→│
  │                                   │
  │──── TLS 握手（可选）────────────────→│  ~5-10ms
  │←──── 证书 + 协商密钥 ────────────────│
  │                                   │
  │──── MySQL Authentication ─────────→│  ~1ms
  │←──── 认证结果 ──────────────────────│
  │                                   │
  │──── Query ────────────────────────→│  执行
  │←──── Result ───────────────────────│
  │                                   │
  │──── TCP 四次挥手 ─────────────────→│  ~1ms
```

整个过程耗时 **20ms ~ 100ms**（取决于网络、TLS、数据库负载）。如果是短连接，每次请求都重新建联，开销巨大。

**连接池的核心思想**：预先建立 N 条连接，请求用完归还池子而非关闭，复用 TCP/TLS 会话。

```
# 无连接池
请求1 → 建连(50ms) → 查询(5ms) → 关连(5ms) = 60ms
请求2 → 建连(50ms) → 查询(5ms) → 关连(5ms) = 60ms
请求3 → 建连(50ms) → 查询(5ms) → 关连(5ms) = 60ms

# 有连接池（3连接）
请求1 → 借连接(0ms) → 查询(5ms) → 还连接(0ms) = 5ms
请求2 → 借连接(0ms) → 查询(5ms) → 还连接(0ms) = 5ms
请求3 → 借连接(0ms) → 查询(5ms) → 还连接(0ms) = 5ms
```

### 1.2 连接池核心参数

| 参数 | 含义 | 过小的后果 | 过大的后果 |
|------|------|-----------|-----------|
| **最大连接数** (MaxOpenConns) | 池子能持有的最多连接数 | 请求排队等待，高并发吞吐低 | 数据库连接过多，资源耗尽 |
| **最大空闲数** (MaxIdleConns) | 空闲状态最多保留几条连接 | 突发流量时频繁建连 | 浪费数据库连接资源 |
| **连接最大生命周期** (ConnMaxLifetime) | 一条连接最多存活多久 | 无影响 | 数据库的 wait_timeout 到了会主动踢连接 |
| **空闲超时** (ConnMaxIdleTime) | 空闲多久的连接会被关闭 | 同上 | 突发流量时预热的连接被关闭 |

### 1.3 连接池状态机

```
                    ┌─────────────────────────────────────┐
                    │           connection_pool             │
                    │                                      │
   request ────────►│  idle_pool ──────► in_use_pool ────►│─── response
                    │      ▲                 │              │
                    │      │                 ▼              │
                    │      │          ┌──────────┐          │
                    │      └──────────│  closed  │◄─────────┘
                    │                 └──────────┘   error
                    └─────────────────────────────────────┘

状态说明：
- idle：连接在池中等待复用
- in_use：被业务代码持有，正在执行查询
- closed：连接已关闭（主动关闭或被数据库 server 踢掉）

关键点：in_use 状态的连接不能被其他请求借用，必须等持有者主动归还！
```

---

## 2. database/sql 连接池（Go 重点）

### 2.1 四大核心参数

`database/sql` 是 Go 标准库提供的通用 SQL 抽象层，它的连接池有四个关键参数：

```go
import "database/sql"

db, err := sql.Open("mysql", "user:password@tcp(localhost:3306)/mydb")
if err != nil {
    log.Fatal(err)
}

// === 核心配置 ===

// MaxOpenConns: 最大打开的连接数
// 超过这个数量的新请求会阻塞等待（不是拒绝！）
// 默认 0 = 无限制（但受限于系统 fd 限制）
db.SetMaxOpenConns(100)

// MaxIdleConns: 最大空闲连接数
// 多余的空闲连接会在下次归还时被关闭
// 默认 2 = 最多保留 2 条空闲连接
// 建议：设的和 MaxOpenConns 一样，减少频繁建连
db.SetMaxIdleConns(20)

// ConnMaxLifetime: 连接最大生命周期
// 到期后连接会被替换（即使在 in_use 也会在归还时关闭）
// 默认永久存活
// 推荐：设为 DB 的 wait_timeout 的一半，如 30min
db.SetConnMaxLifetime(30 * time.Minute)

// ConnMaxIdleTime: 空闲超时（Go 1.15+）
// 空闲超过这个时间的连接会被关闭
// 默认永久
// 推荐：设为 5-10min
db.SetConnMaxIdleTime(5 * time.Minute)
```

### 2.2 生产推荐配置计算公式

```
MaxOpenConns = min(DB最大连接数 / 实例数, 单实例最大并发承受力)

举例：
- MySQL 单实例 max_connections = 5000
- 应用实例数 = 20
- 每个实例的 MaxOpenConns = min(5000/20, 500) = 250

实际生产中还要考虑：
- 每个请求的 QPS
- 请求的平均执行时间
- 业务逻辑中是否有慢查询（占着连接不放）
```

**一个典型的生产配置：**

```go
func NewDB(dsn string) *sql.DB {
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        log.Fatalf("open db failed: %v", err)
    }

    // 连接池配置
    db.SetMaxOpenConns(100)
    db.SetMaxIdleConns(20)          // 不等于 MaxOpenConns，因为空闲连接会消耗 DB 资源
    db.SetConnMaxLifetime(30 * time.Minute)
    db.SetConnMaxIdleTime(5 * time.Minute)

    // 验证连接可用
    if err := db.Ping(); err != nil {
        log.Fatalf("ping db failed: %v", err)
    }

    return db
}
```

### 2.3 监控：db.Stats() 是你的眼睛

```go
// 在 HTTP handler 或定时任务中打印连接池状态
func handleHealth(w http.ResponseWriter, r *http.Request) {
    stats := db.Stats()

    fmt.Fprintf(w, `Database Connection Pool Stats:
  OpenConnections: %d
  InUse:           %d
  Idle:            %d
  WaitCount:       %d      // 等待连接的次数（累计）
  WaitDuration:    %v     // 等待连接的总时长（累计）
  MaxIdleClosed:   %d      // 因超过 MaxIdleConns 而关闭的连接数
  MaxLifetimeClosed: %d    // 因超过 ConnMaxLifetime 而关闭的连接数
`, stats.OpenConnections, stats.InUse, stats.Idle,
        stats.WaitCount, stats.WaitDuration,
        stats.MaxIdleClosed, stats.MaxLifetimeClosed)

    // 告警条件
    if stats.InUse > 80 {
        // 实际告警逻辑（上报 Prometheus / 发送告警）
        fmt.Fprintf(w, "\n⚠️  WARNING: InUse connections > 80%%")
    }
    if stats.WaitCount > 1000 {
        fmt.Fprintf(w, "\n⚠️  WARNING: WaitCount is high, check MaxOpenConns")
    }
}
```

**Stats 关键字段解读：**

| 字段 | 含义 | 告警阈值建议 |
|------|------|-------------|
| `OpenConnections` | 当前打开的总连接数 | 接近 MaxOpenConns 时告警 |
| `InUse` | 正在使用的连接数 | 持续 > 80% 告警 |
| `Idle` | 空闲连接数 | — |
| `WaitCount` | 累计等待次数 | 快速增长说明连接不够用 |
| `MaxIdleClosed` | 因超过 MaxIdleConns 关闭的连接数 | 快速增长说明 MaxIdleConns 设太小 |
| `MaxLifetimeClosed` | 因超过 ConnMaxLifetime 关闭的连接数 | 快速增长说明 lifetime 设太短 |

---

## 3. 连接泄漏排查（P0 高频问题）

### 3.1 什么是连接泄漏

连接被借出后，持有者没有归还（没有 `Close()`），导致连接永远处于 `in_use` 状态，池子逐渐耗尽。

```
正常流程：
借连接 → 执行查询 → 归还连接

泄漏流程：
借连接 → 执行查询 → ???（没有归还）→ 连接永远 in_use
```

### 3.2 泄漏的三大根因

**根因一：忘记 Close rows**

```go
// ❌ 错误：rows 没关闭，连接泄漏
func badQuery() {
    rows, err := db.Query("SELECT * FROM users WHERE id = ?", userID)
    if err != nil {
        return
    }
    // 忘记 rows.Close()，连接泄漏
    for rows.Next() {
        // 处理 rows
    }
    return // rows 还在 in_use，连接无法归还
}

// ✅ 正确：用 defer 关闭
func goodQuery() {
    rows, err := db.Query("SELECT * FROM users WHERE id = ?", userID)
    if err != nil {
        return
    }
    defer rows.Close() // 无论如何都会执行

    for rows.Next() {
        // 处理 rows
    }
}

// ✅ 也正确：用 defer 但先判断 err
func betterQuery() {
    rows, err := db.Query("SELECT * FROM users WHERE id = ?", userID)
    if err != nil {
        return
    }
    if closeErr := rows.Close(); closeErr != nil {
        // log error but don't mask the original err
    }
}
```

**根因二：事务未提交/回滚**

```go
// ❌ 错误：defer Rollback 写错位置
func badTransaction() {
    tx, err := db.Begin()
    if err != nil {
        return
    }
    // 这里如果 panic，Rollback 不会执行（defer 还没注册）
    doSomething(tx)
    tx.Commit()
}

// ✅ 正确：defer 紧跟 tx 创建
func goodTransaction() {
    tx, err := db.Begin()
    if err != nil {
        return
    }
    defer tx.Rollback() // 立即注册，无论后面发生什么都会执行

    doSomething(tx)

    err = tx.Commit()
    if err != nil {
        return // defer 仍然会执行 Rollback（已提交则 Rollback 是 no-op）
    }
}

// ✅ 更健壮：用匿名函数限制 defer 作用域
func bestTransaction() {
    tx, err := db.Begin()
    if err != nil {
        return
    }

    // 将 tx 封装进闭包，defer 更安全
    err = func() error {
        defer tx.Rollback()

        if err := doSomething(tx); err != nil {
            return err
        }
        return tx.Commit()
    }()
}
```

**根因三：QueryRow 后没调用 Scan**

```go
// ❌ 错误：QueryRow 后没 Scan，连接可能未正确释放
func badQueryRow() {
    var name string
    err := db.QueryRow("SELECT name FROM users WHERE id = ?", userID).Err()
    // QueryRow.Err() 返回的是扫描错误，不是查询本身的错误
    // 而且没有 Scan()，连接可能没有正确归还

    // ❌ 没调用 Scan()
    fmt.Println(name)
}

// ✅ 正确：明确调用 Scan
func goodQueryRow() {
    var name string
    err := db.QueryRow("SELECT name FROM users WHERE id = ?", userID).Scan(&name)
    if err != nil {
        if err == sql.ErrNoRows {
            // 处理没找到的情况
        } else {
            // 其他错误
        }
    }
    fmt.Println(name)
}
```

### 3.3 排查 SOP

**Step 1：观察 Stats**

```go
func checkLeak() {
    stats := db.Stats()
    fmt.Printf("InUse: %d, WaitCount: %d, Idle: %d\n",
        stats.InUse, stats.WaitCount, stats.Idle)
    // 如果 InUse 持续增长且不回落 = 泄漏
    // 如果 WaitCount 快速增长 = 连接不够用（可能是泄漏，也可能是真的高并发）
}
```

**Step 2：pprof goroutine dump**

```go
import _ "net/http/pprof"

// 在启动时开启 pprof
go http.ListenAndServe(":6060", nil)
```

```bash
# 抓取 goroutine dump
curl -o goroutine.txt http://localhost:6060/debug/pprof/goroutine?debug=1

# 找阻塞在 (*DB).conn 的 goroutine
grep -A 5 "(*DB).conn" goroutine.txt
```

**Step 3：代码审查（最高效）**

```
重点检查：
1. Query / Exec / QueryRow 后是否 Close/Scan
2. Transaction 是否 defer Rollback
3. for rows.Next() 循环后是否 rows.Close()
4. defer 是否在正确的位置（紧跟创建资源的语句后面）
```

### 3.4 waitTimeout vs interactive_timeout（MySQL）

```sql
-- MySQL 服务端配置
show variables like '%timeout%';

-- wait_timeout: 非交互连接（如 JDBC、Go 连接池）的空闲超时
-- interactive_timeout: 交互式连接（如 mysql client）的空闲超时
-- 建议：两者都设为 300-600 秒

SET GLOBAL wait_timeout = 300;
SET GLOBAL interactive_timeout = 300;
```

**Go 端的对策：**

```go
// 如果 DB 的 wait_timeout = 300s
// Go 端的 ConnMaxLifetime 应该设为 < 300s（建议 30min 其实太长了）
// 更安全的做法：设为 DB wait_timeout 的 80%
db.SetConnMaxLifetime(4 * time.Minute) // 240s = 300 * 0.8
```

---

## 4. Redis 连接池（go-redis）

### 4.1 go-redis 连接池核心参数

```go
import "github.com/redis/go-redis/v9"

rdb := redis.NewClient(&redis.Options{
    Addr:     "localhost:6379",

    // 连接池大小：每个 redis.Client 实例的连接池
    // go-redis 默认是 CPU 核数 × 10（goroutine 友好）
    // 但实际上 Redis 单实例 QPS 有限，设太大会浪费连接
    PoolSize: 100,

    // 最小空闲连接数：预热时保持的最小连接
    // 避免冷启动时第一次请求建连慢
    MinIdleConns: 10,

    // 连接最大生命周期
    // 注意：Redis 的 maxmemory-policy 也会在内存不足时驱逐 key
    ConnMaxLifetime: time.Hour,

    // 空闲超时：连接多久不用就关闭
    ConnMaxIdleTime: 30 * time.Minute,

    // 等待连接超时（区别于命令执行超时）
    // 从池中获取连接时，超过这个时间还没拿到就报错
    PoolTimeout: 4 * time.Second,

    // 命令执行超时（读/写命令各自的 timeout）
    ReadTimeout:  3 * time.Second,
    WriteTimeout: 3 * time.Second,

    // 最大重试次数
    MaxRetries: 3,
})
```

### 4.2 PoolSize 计算

```
Redis 连接池大小计算公式：

PoolSize = min(单实例 QPS 峰值 / 单连接 QPS, Redis maxclients / 实例数)

举例：
- Redis 单实例 QPS = 10万
- 单连接每秒可处理 = 5000 请求
- 每个实例应该的 PoolSize = min(100000/5000, 1000) = 20

实际生产中：
- 读多写少：PoolSize 可以设大一些（受限于 Redis 单线程，不是连接数）
- 写多读少：PoolSize 建议保守，连接占用时间长
- 批处理：Pipeline 可以显著减少连接数需求
```

### 4.3 Pipeline 的 RTT 优化

Pipeline 将多个命令打包成一个请求发送，大幅减少 Round Trip Time：

```go
// ❌ 不用 Pipeline：N 次 RTT
func badPipeline(rdb *redis.Client) {
    ctx := context.Background()
    for _, key := range keys {
        val, _ := rdb.Get(ctx, key).Result() // 每次都 RTT
        // 处理 val
    }
}

// ✅ Pipeline：1 次 RTT
func goodPipeline(rdb *redis.Client) {
    ctx := context.Background()
    pipe := rdb.Pipeline()

    cmds := make([]*redis.StringCmd, len(keys))
    for i, key := range keys {
        cmds[i] = pipe.Get(ctx, key)
    }

    // Exec 一次性发送所有命令，1 次 RTT
    _, err := pipe.Exec(ctx)
    if err != nil && err != redis.Nil {
        // 处理错误
    }

    for _, cmd := range cmds {
        val, err := cmd.Result()
        if err == redis.Nil {
            // key 不存在
        } else if err != nil {
            // 其他错误
        }
        // 处理 val
    }
}

// ✅ 吞吐对比
// 1000 个 GET：不用 Pipeline = 1000 RTT ≈ 1-2s
// 1000 个 GET：Pipeline = 1 RTT ≈ 1-2ms
```

### 4.4 Redis 连接健康检查

```go
// go-redis 默认使用空闲检查来验证连接可用性
// 每隔一段时间从池中取空闲连接 PING 一下

rdb := redis.NewClient(&redis.Options{
    Addr: "localhost:6379",
    PoolSize: 100,

    // 空闲检查频率
    // 默认 PoolCheckInterval = time.Minute
    // 即每分钟检查一次空闲连接的可用性
    PoolCheckInterval: 30 * time.Second,

    // 是否在执行命令前检查连接可用性（轻微性能损耗，但更安全）
    // DisableIndentity: false 时，每次命令前可能多一次 PING
})
```

---

## 5. gRPC 连接复用

### 5.1 gRPC 为什么不需要传统意义的"连接池"

gRPC 基于 HTTP/2，HTTP/2 天然支持**多路复用**：

```
HTTP/1.1（短连接）：
请求A ──────────────────────────────────►
请求B ─────────►（必须等 A 完成）
请求C ──────►（必须等 B 完成）

HTTP/2（多路复用）：
请求A ────►
请求B ──►
请求C ► （三个请求在同一个 TCP 连接上并行）
```

**一个 gRPC `ClientConn` 默认就是一个 HTTP/2 连接，承载所有 RPC 请求。**

```go
// ✅ 错误示范：每个请求都新建 ClientConn（重建连接，浪费）
func badGRPC() {
    for i := 0; i < 100; i++ {
        conn, _ := grpc.NewClient("localhost:50051", grpc.WithInsecure())
        client := pb.NewMyServiceClient(conn)
        client.DoSomething(ctx, &pb.Request{})
        conn.Close() // 建了 100 个连接！
    }
}

// ✅ 正确示范：复用 ClientConn
func goodGRPC() {
    conn, err := grpc.NewClient("localhost:50051", grpc.WithInsecure())
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    client := pb.NewMyServiceClient(conn)

    for i := 0; i < 100; i++ {
        client.DoSomething(ctx, &pb.Request{}) // 复用同一个 HTTP/2 连接
    }
}
```

### 5.2 什么时候需要多个连接

**场景一：负载均衡模式**

```go
// pick_first：所有请求都发到一个后端（默认）
// round_robin：每次请求轮询到不同后端（需要多个子连接）
conn, err := grpc.NewClient(
    "passthrough:///localhost:50051,localhost:50052",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"round_robin":{}}]}`),
)
```

**场景二：连接数限制**

gRPC 一个 TCP 连接上可以承载的并发 stream 数有限（默认 100）。如果你的并发需求超过这个限制，可以创建多个连接：

```go
// 创建多个连接（类似连接池）
conns := make([]*grpc.ClientConn, 3)
for i := 0; i < 3; i++ {
    conns[i], _ = grpc.NewClient(fmt.Sprintf("localhost:%d", 50051+i))
    defer conns[i].Close()
}

// 轮询使用
var idx atomic.Int64
func getConn() *grpc.ClientConn {
    return conns[idx.Add(1) % 3]
}
```

### 5.3 Keepalive 配置

gRPC 的 Keepalive 用于保活空闲连接，防止中间件（如负载均衡器）关闭空闲连接：

```go
import "google.golang.org/grpc/keepalive"

conn, err := grpc.NewClient(
    "localhost:50051",
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        // 客户端在空闲多长时间后发送 PING
        Time: 10 * time.Second,

        // 等待 PONG 的超时时间
        Timeout: 5 * time.Second,

        // 是否允许 ping 而没有活跃的 RPC
        PermitWithoutStream: true,
    }),
)
```

**推荐配置：**

```go
// 服务端
server := grpc.NewServer(
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle: 15 * time.Minute, // 空闲超时
        MaxConnectionAge:  2 * time.Hour,     // 连接最大生命周期
        Time:              5 * time.Hour,     // 发送 keepalive 的间隔
        Timeout:           20 * time.Second, // keepalive 超时
    }),
)

// 客户端
conn, _ := grpc.NewClient(
    "localhost:50051",
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                10 * time.Second,
        Timeout:              5 * time.Second,
        PermitWithoutStream:  true, // 即使没有 RPC 也允许 keepalive
    }),
)
```

---

## 6. 连接池满了怎么办（生产处理链路）

### 6.1 现象识别

```
典型告警：
1.响应时间 P99 > 500ms，持续上升
2.error log: "context deadline exceeded"
3.db.Stats().WaitCount 快速增长
4.MySQL: "Too many connections"
5.Redis: "redis: connection pool exhausted"
```

### 6.2 分层处理策略

```
请求到达
    │
    ▼
┌─────────────────────────────────────┐
│ L1: 连接池排队（PoolTimeout）          │
│  → 设置合理的 PoolTimeout（不要无限等）│
│  → 监控 WaitCount，增长说明不够用       │
└────────────────┬────────────────────┘
                 │ PoolTimeout 超时
                 ▼
┌─────────────────────────────────────┐
│ L2: 熔断（快速失败，不打满 DB）         │
│  → 统计最近 N 秒的 error rate         │
│  → 超过阈值进入熔断状态                 │
│  → 直接返回降级响应                     │
└────────────────┬────────────────────┘
                 │ 熔断触发
                 ▼
┌─────────────────────────────────────┐
│ L3: 降级（走缓存 / 返回友好错误）        │
│  → Redis 缓存兜底                      │
│  → 返回 "服务繁忙，请稍后重试"          │
└────────────────┬────────────────────┘
                 │ 持续告警
                 ▼
┌─────────────────────────────────────┐
│ L4: 扩容（长期方案）                    │
│  → 扩容 DB 实例（读写分离）             │
│  → 增加应用实例数                       │
│  → 优化慢查询（减少连接占用时间）         │
└─────────────────────────────────────┘
```

### 6.3 完整降级代码示例

```go
type DBProxy struct {
    db          *sql.DB
    circuitBreaker func() error
    cache        *redis.Client
}

func (p *DBProxy) QueryWithFallback(ctx context.Context, key string) (string, error) {
    // L1: 尝试走缓存
    cached, err := p.cache.Get(ctx, key).Result()
    if err == nil {
        return cached, nil
    }

    // L2: 尝试查询 DB
    span, ctx := opentracing.StartSpanFromContext(ctx, "db.query")
    defer span.Finish()

    var result string
    queryCtx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
    defer cancel()

    err = p.db.QueryRowContext(queryCtx,
        "SELECT value FROM config WHERE key = ?", key).Scan(&result)

    if err != nil {
        // 查询失败，尝试降级
        span.SetTag("error", true)

        // 降级：从缓存读旧值（即使过期了也比报错好）
        stale, _ := p.cache.Get(ctx, key+"_stale").Result()
        if stale != "" {
            // 异步回填（不阻塞用户请求）
            go p.backfillCache(key)
            return stale, nil
        }

        return "", fmt.Errorf("query failed: %w", err)
    }

    // 写缓存
    p.cache.Set(ctx, key, result, 10*time.Minute)

    return result, nil
}

func (p *DBProxy) backfillCache(key string) {
    var value string
    err := p.db.QueryRow("SELECT value FROM config WHERE key = ?", key).Scan(&value)
    if err == nil {
        // 标记为降级缓存，过期时间短
        p.cache.Set(context.Background(), key+"_stale", value, 5*time.Minute)
    }
}
```

---

## 高频追问

### Q1：MaxOpenConns 设太大会怎样？设太小会怎样？

**设太大：**
- 数据库连接数暴涨，超过 `max_connections` 后被拒绝
- 大量并发请求同时打 DB，DB CPU 飙升
- 连接竞争激烈，实际吞吐反而下降

**设太小：**
- 请求排队等待，延迟上升
- WaitCount 持续增长
- 吞吐上不去，资源浪费

**最佳实践：** 公式 `MaxOpenConns = DB max_connections / 实例数`，然后压测调优。

---

### Q2：Go 的 `database/sql` 连接池是并发安全的吗？

**是的。** `*sql.DB` 是并发安全的。

- 多个 goroutine 可以同时调用 `db.Query()`、`db.Exec()` 等
- 连接池内部有 mutex 保护，goroutine 安全地借用和归还连接
- 你不需要自己加锁

**但要注意：**
- `*sql.Rows` 和 `*sql.Tx` 不是并发安全的，一个 rows 不能同时被多个 goroutine 使用
- 如果你需要事务并发执行，每个事务用独立的 `db.Begin()`

```go
// ✅ 安全：多个 goroutine 共用 *sql.DB
var wg sync.WaitGroup
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        db.QueryContext(ctx, "SELECT 1") // 完全安全
    }()
}
wg.Wait()

// ❌ 不安全：rows 被多 goroutine 共用
rows, _ := db.QueryContext(ctx, "SELECT * FROM users")
go func() { rows.Next() }()   // 危险！
go func() { rows.Close() }()  // 数据竞争！
```

---

### Q3：为什么 Redis 连接池满比 MySQL 连接池满更危险？

**MySQL (`database/sql`) 满了：**
- 新请求会**等待**（不拒绝）
- 等待队列满了之后才报 `context deadline exceeded`
- 有一定的缓冲空间

**Redis (go-redis) 满了：**
- `PoolTimeout` 超过后直接**报错** `"redis: connection pool exhausted"`
- 没有等待队列机制
- 对业务是纯错误，不像 MySQL 那样排队等待

**根本原因：**
- MySQL 请求通常是查询/写入，连接占用时间相对长，MySQL 驱动设计为等待
- Redis 请求通常是快速 GET/SET，连接占用时间极短，go-redis 期望连接快速释放
- go-redis 的 `PoolTimeout` 默认为 0（无限等待），但实际生产中设太短容易出问题

**生产建议：**
```go
rdb := redis.NewClient(&redis.Options{
    PoolSize:    100,
    PoolTimeout: 4 * time.Second, // 4秒还没拿到连接就报错（不要设太短）
    MinIdleConns: 10,              // 预热，避免冷启动
})
```

---

### Q4：如何判断连接是被泄漏了还是真的高并发？

看 `WaitCount` 和 `InUse` 的趋势：

```
泄漏特征：
  InUse = 100（持续）  // 一直在用，不释放
  Idle = 0              // 没有空闲连接
  WaitCount = 0（或缓慢增长） // 没有排队，说明不是等不到连接，是拿到了不还

高并发特征：
  InUse = 100（峰值）   // 偶尔打满
  Idle = 0 → 20 → 0     // 波动，有释放
  WaitCount = 10000+（快速增长） // 大量排队等连接
```

---

### Q5：连接池参数调优的压测方法

```go
// 用 pg 开源库做基准压测
// https://github.com/jmoiron/sqlx

func BenchmarkDBPool(b *testing.B) {
    db, _ := sql.Open("mysql", "root:@/test")
    db.SetMaxOpenConns(100)
    db.SetMaxIdleConns(20)

    defer db.Close()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        var id int
        db.QueryRowContext(context.Background(), "SELECT 1").Scan(&id)
    }
}

// 运行：
// go test -bench=. -benchmem -run=^$

// 观察：
// - 吞吐量（ops/sec）
// - 内存分配（allocs/op）
// - 延迟分布
```

---

## 导航

← [缓存三大问题：穿透/击穿/雪崩](../02-redis/03-cache-problems.md) ｜ [ClickHouse 实战](../07-clickhouse/01-clickhouse.md) →
