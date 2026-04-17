# 性能问题排查与优化

> 考察频率：★★★★★
> 面试官考察点：能不能讲出排查思路 → 定位手段 → 解决方案的完整链路，要有真实数据。

---

## 问题导航

| # | 问题 | 核心关键词 |
|---|------|----------|
| 1 | [接口 P99 突然升高](#1-接口-p99-突然升高) | 链路追踪、pprof、GC停顿 |
| 2 | [服务内存持续增长](#2-服务内存持续增长) | goroutine泄漏、全局缓存、pprof heap |
| 3 | [CPU 突然打满](#3-cpu-突然打满) | 死循环、正则回溯、JSON序列化 |
| 4 | [流量突增服务不稳定](#4-流量突增服务不稳定) | 连接池打满、GC频繁、sync.Pool |
| 5 | [Redis 变慢](#5-redis-变慢) | bigkey、热key、slowlog、pipeline |

---

## 1. 接口 P99 突然升高

### 面试官考察意图
P99 升高是最常见的线上告警，考察候选人能不能系统性地排查，而不是靠猜。

### 排查 SOP（标准操作流程）

```
Step 1: 看监控面板
  - 对比 P50/P95/P99 曲线，判断是全量变慢还是尾延迟
  - 检查 CPU/内存/网络是否有异常

Step 2: 看链路追踪（Jaeger/Zipkin）
  - 找到慢请求的 TraceID
  - 展开 Span，定位是哪个下游（DB/Redis/HTTP）慢

Step 3: pprof 采样
  - CPU profile：看哪个函数耗时占比高
  - Block profile：看 goroutine 阻塞在哪里

Step 4: 看 GC 日志
  - GODEBUG=gctrace=1 观察 GC 频率和 STW 时间
```

### GC 导致 P99 升高（最常见）

```go
// 现象：P99 每隔几秒出现一个尖刺，正好和 GC 频率一致
// 原因：大量短期对象分配，GC 频繁触发 STW

// 问题代码：每次请求都分配大量临时对象
func handleRequest(data []byte) Response {
    // 每次都 new，加大 GC 压力
    buf := make([]byte, 4096)
    result := process(data, buf)
    return result
}

// 优化：sync.Pool 复用对象
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

func handleRequestOptimized(data []byte) Response {
    buf := bufPool.Get().([]byte)
    defer bufPool.Put(buf)
    return process(data, buf)
}
```

### 下游超时拖慢 P99

```go
// 现象：P99 高，但自身 CPU 不高，链路追踪看 DB span 耗时 2 秒
// 原因：无超时限制，等待下游

// 问题代码
func getUser(db *sql.DB, id int) (*User, error) {
    row := db.QueryRow("SELECT * FROM users WHERE id=?", id)
    // 如果 DB 卡住，这里会永久等待
    // ...
}

// 优化：所有下游调用带 Context 超时
func getUserWithTimeout(db *sql.DB, id int) (*User, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()

    row := db.QueryRowContext(ctx, "SELECT * FROM users WHERE id=?", id)
    var user User
    if err := row.Scan(&user.ID, &user.Name); err != nil {
        return nil, err
    }
    return &user, nil
}
```

### 面试话术

> "有一次我们的下单接口 P99 从 200ms 涨到 1.2s，P50 正常。用 Jaeger 看链路，发现 DB 的 SELECT span 有时候会到 800ms。查 MySQL 慢查询日志，发现某个 SQL 因为索引失效做了全表扫描，大表 500 万行，慢的时候扫了 20 万行。加上复合索引后，P99 降回 180ms。"

---

## 2. 服务内存持续增长

### 面试官考察意图
内存泄漏是 Go 生产环境的常见问题，考察候选人能否用 pprof 定位。

### 常见原因

```
1. goroutine 泄漏（最常见）：goroutine 启动了但永远不会退出
2. 全局变量无限增长：map/slice 只加不删
3. 缓存没有过期策略：本地缓存没有 TTL
4. Context 泄漏：WithCancel 的 cancel 没调用
```

### 排查流程

```bash
# 1. 观察 goroutine 数量是否持续增长
curl http://localhost:6060/debug/pprof/goroutine?debug=1 | head -50

# 2. 采集 heap profile，对比两次
go tool pprof http://localhost:6060/debug/pprof/heap
(pprof) top20

# 3. 用 diff 对比前后两次 heap（排查增量）
go tool pprof -base heap1.pb.gz heap2.pb.gz
(pprof) top20 -cum
```

### goroutine 泄漏示例与修复

```go
// 问题：goroutine 在 channel 上永久阻塞
func badCode() {
    for i := 0; i < 1000; i++ {
        ch := make(chan int)
        go func() {
            val := <-ch // 如果没人发送，永远阻塞！goroutine 泄漏
            process(val)
        }()
        // ch 没有发送数据就超出作用域了
    }
}

// 修复：用 Context 控制 goroutine 生命周期
func goodCode(ctx context.Context) {
    ch := make(chan int, 1)
    go func() {
        select {
        case val := <-ch:
            process(val)
        case <-ctx.Done(): // 超时或取消时退出
            return
        }
    }()
}

// 全局缓存泄漏
var cache = make(map[string][]byte) // 只增不删，内存不断增长

// 修复：用带 TTL 的缓存库，或定期清理
import "github.com/patrickmn/go-cache"

var c = cache.New(5*time.Minute, 10*time.Minute) // 5分钟过期，10分钟清理
```

### 面试话术

> "服务上线一天后内存从 200MB 涨到 1.5GB，k8s OOM 重启了。采集 goroutine profile，发现 goroutine 从启动时的 50 个涨到了 8000 个。看 goroutine 的 stack trace，都卡在一个 channel 读取上。定位到代码里一个没有传 Context 的后台 goroutine，收到请求就启动一个，但没有退出条件。加上 Context 超时控制后，goroutine 数量稳定在 50 个以内，内存不再增长。"

---

## 3. CPU 突然打满

### 面试官考察意图
CPU 飙升的原因多种多样，考察候选人能否快速定位热点函数。

### 排查步骤

```bash
# 1. 采集 30s CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 2. 看 top10 函数
(pprof) top10

# 3. 生成火焰图
(pprof) web
# 或者
go tool pprof -http=:8080 cpu.prof
```

### 常见 CPU 热点及修复

```go
// 场景1：正则表达式每次编译（高频调用时极耗CPU）
// 问题：每次请求都 Compile 正则
func extractEmail(text string) string {
    re := regexp.MustCompile(`[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}`)
    // 编译正则很贵！
    return re.FindString(text)
}

// 修复：全局变量预编译
var emailRegex = regexp.MustCompile(`[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}`)

func extractEmailFast(text string) string {
    return emailRegex.FindString(text)
}

// 场景2：JSON 序列化热点
// 标准库 encoding/json 用反射，高频调用 CPU 高

// 优化：使用 sonic（字节跳动开源，比标准库快 3-6 倍）
import "github.com/bytedance/sonic"

// 替换标准库
// json.Marshal(v)   → sonic.Marshal(v)
// json.Unmarshal()  → sonic.Unmarshal()

// 场景3：字符串拼接在循环里（产生大量临时对象）
// 问题
func buildQuery(fields []string) string {
    result := ""
    for _, f := range fields {
        result += f + "," // 每次都分配新字符串
    }
    return result
}

// 修复：strings.Builder
func buildQueryFast(fields []string) string {
    var sb strings.Builder
    sb.Grow(len(fields) * 16) // 预分配容量
    for i, f := range fields {
        if i > 0 {
            sb.WriteByte(',')
        }
        sb.WriteString(f)
    }
    return sb.String()
}
```

### 面试话术

> "有次监控告警 CPU 持续 95%。采集 CPU profile，火焰图上看到 `regexp.Compile` 占了 40% 的时间。定位到一个解析日志的函数，每次调用都在函数内部编译正则表达式。把正则改成 package 级别的全局变量后，CPU 直接从 95% 降到 20%。"

---

## 4. 流量突增服务不稳定

### 面试官考察意图
高并发下服务的稳定性保障，考察候选人对连接池、对象复用等优化手段的掌握。

### 连接池打满的症状和处理

```go
// 症状：大量请求超时，日志报 "too many connections"
// 诊断
func diagnoseDB(db *sql.DB) {
    stats := db.Stats()
    log.Printf("MaxOpen: %d, Open: %d, InUse: %d, Idle: %d, WaitCount: %d",
        stats.MaxOpenConnections,
        stats.OpenConnections,
        stats.InUse,
        stats.Idle,
        stats.WaitCount,
    )
    // WaitCount 很大说明连接不够用
}

// 调优连接池参数
func setupDB(dsn string) *sql.DB {
    db, _ := sql.Open("mysql", dsn)
    db.SetMaxOpenConns(100)              // 最大连接数（根据 DB 服务器承受能力设）
    db.SetMaxIdleConns(20)               // 空闲连接数（保持活跃，避免频繁建连）
    db.SetConnMaxLifetime(time.Hour)     // 连接最大存活时间（避免被 DB 侧关闭）
    db.SetConnMaxIdleTime(10 * time.Minute) // 空闲连接超时（清理长时间不用的连接）
    return db
}
```

### GC 压力导致卡顿

```go
// 高并发场景下，每个请求都分配大量临时对象，GC 频繁触发
// 用 sync.Pool 减少 GC 压力

type RequestContext struct {
    Buffer []byte
    Header map[string]string
}

var ctxPool = sync.Pool{
    New: func() interface{} {
        return &RequestContext{
            Buffer: make([]byte, 0, 4096),
            Header: make(map[string]string, 10),
        }
    },
}

func handleHTTP(w http.ResponseWriter, r *http.Request) {
    ctx := ctxPool.Get().(*RequestContext)
    // 重置状态（重要！Pool 的对象可能是脏的）
    ctx.Buffer = ctx.Buffer[:0]
    for k := range ctx.Header {
        delete(ctx.Header, k)
    }
    defer ctxPool.Put(ctx)

    // 使用 ctx 处理请求...
}
```

---

## 5. Redis 变慢

### 面试官考察意图
Redis 是高频考点，考察候选人能否从多个维度排查 Redis 性能问题。

### 排查步骤

```bash
# 1. 查看慢日志（默认记录超过 10ms 的命令）
redis-cli SLOWLOG GET 10

# 2. 查看内存使用
redis-cli INFO memory

# 3. 扫描 bigkey（不要用 KEYS *！）
redis-cli --bigkeys --scan

# 4. 实时监控命令
redis-cli MONITOR  # 注意：生产慎用，有性能影响
```

### bigkey 问题

```go
// bigkey：单个 key 的 value 过大（如存了几 MB 的 JSON）
// 后果：序列化/反序列化慢，网络传输慢，可能阻塞 Redis 主线程

// 检测：查看 key 大小
func checkKeySize(rdb *redis.Client, key string) {
    ctx := context.Background()
    size, _ := rdb.MemoryUsage(ctx, key).Result()
    if size > 100*1024 { // 超过 100KB 就算 bigkey
        log.Warnf("bigkey detected: %s, size: %d bytes", key, size)
    }
}

// 解决：拆分 bigkey
// 原来：一个 key 存整个用户信息（几十个字段）
// rdb.Set(ctx, "user:1001", largeJSON, 0)

// 优化：用 Hash 存，按需读取
func setUserInfo(rdb *redis.Client, userID int, user User) {
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", userID)
    rdb.HSet(ctx, key,
        "name", user.Name,
        "email", user.Email,
        "level", user.Level,
        // 只存常用字段，不常用的存 DB
    )
    rdb.Expire(ctx, key, time.Hour)
}
```

### 热key 问题

```go
// 热key：某个 key 被极高频访问（如热点商品、明星微博）
// 后果：单个 Redis 节点 CPU 打满

// 解决1：本地缓存 + Redis 二级缓存
var localCache sync.Map // 本地缓存

func getProduct(rdb *redis.Client, productID int) (*Product, error) {
    key := fmt.Sprintf("product:%d", productID)

    // 先查本地缓存（无网络开销）
    if v, ok := localCache.Load(key); ok {
        return v.(*Product), nil
    }

    // 再查 Redis
    data, err := rdb.Get(context.Background(), key).Bytes()
    if err == nil {
        var p Product
        json.Unmarshal(data, &p)
        localCache.Store(key, &p) // 写入本地缓存
        // 注意：本地缓存要有过期机制，这里简化了
        return &p, nil
    }
    return nil, err
}

// 解决2：key 加随机后缀，分散到多个 Redis 节点
func getHotKey(rdb *redis.Client, key string) string {
    suffix := rand.Intn(10) // 10个副本
    realKey := fmt.Sprintf("%s#%d", key, suffix)
    val, _ := rdb.Get(context.Background(), realKey).Result()
    return val
}
```

---

## 高频追问汇总

| 问题 | 核心答案 |
|------|---------|
| pprof 会影响线上性能吗？ | 有轻微影响（约 5%），CPU profile 采样默认 100Hz，可以接受。不要在生产上长时间开 MUTEX profile。 |
| GC STW 时间大概多少？ | Go 1.14 以后 STW 通常 < 1ms，极端情况（大堆+对象多）可能 5-10ms。 |
| 如何减少 GC 次数？ | 减少对象分配（sync.Pool 复用）、调大 GOGC 值（默认100，设到200让堆增长更多再GC）、复用 slice 和 map。 |
| Redis pipeline 和事务的区别？ | Pipeline 是批量发送命令减少 RTT，不保证原子性；MULTI/EXEC 是事务保证原子性，但不支持回滚；Lua 脚本既批量又原子。 |
