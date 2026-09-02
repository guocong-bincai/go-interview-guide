# 资源泄漏与安全问题分析

> 考察频率：★★★★★  优先级：P0

---

## 问题导航

| # | 问题 | 核心关键词 |
|---|------|----------|
| 1 | [goroutine 泄漏排查与根治](#1-goroutine-泄漏排查与根治) | pprof goroutine profile、select+ctx.Done、通道未消费 |
| 2 | [HTTP Response.Body 泄漏导致连接耗尽](#2-http-responsebody-泄漏导致连接耗尽) | http.Client、Body.Close()、连接池耗尽 |
| 3 | [文件句柄泄漏导致 EMFILE 崩溃](#3-文件句柄泄漏导致-emfile-崩溃) | os.Open、CloseOnExec、fd 数量限制 |
| 4 | [Data Race：并发读写竞态条件](#4-data-race并发读写竞态条件) | go test -race、互斥锁、channel通信 |
| 5 | [panic/recover 误用导致服务不可预测](#5-panicrecover-误用导致服务不可预测) | defer panic recover、错误处理规范 |
| 6 | [TCP TIME_WAIT 过多导致端口耗尽](#6-tcp-time_wait-过多导致端口耗尽) | KeepAlive、SO_REUSEADDR、连接复用 |

---

## 1. goroutine 泄漏排查与根治

### 面试官考察意图

这是 Go 后端最经典也是最高频的线上故障之一。面试官想看你有没有**系统性的排查方法论**：怎么发现 → 怎么定位 → 怎么修复 → 怎么防止复发。光说"加个 ctx.Done()"是不够的，要展示完整的诊断能力。

### 什么是 goroutine 泄漏

```
goroutine 启动了但永远不会退出，导致它持有的所有资源永远不会释放。
随时间推移，活跃 goroutine 数无限增长 → OOM / 调度器过载 / 服务卡死。
```

常见泄漏模式：
```
1. Channel 读阻塞：goroutine 在读一个永远不会被关闭/发送数据的 channel
2. Timer 未 Stop：time.After/ticker 创建的 timer 没被 Stop，即使不再需要
3. select 无退出条件：只有 case <-ch，但没有 <-ctx.Done() 或 break 条件
4. Context 不传递：启动 goroutine 时传的是 context.Background() 而不是请求级 context
```

### 排查 SOP

```bash
# Step 1: 观察 goroutine 总数是否持续增长
curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | grep "^goroutine" | wc -l

# Step 2: 采集 goroutine profile，看堆栈分布
curl -s http://localhost:6060/debug/pprof/goroutine?debug=2 > goroutine.pb.gz

# Step 3: 分析具体卡在哪个函数
go tool pprof -top goroutine.pb.gz
# 关注 "wait on channel", "sleeping", "IO wait" 状态的数量

# Step 4: diff 对比两个时间点的 goroutine 差异
go tool pprof -base old.pb.gz new.pb.gz
(pprof) list 关键函数名
```

### 典型泄漏案例与修复

#### 案例1：select 没有退出条件

```go
// ❌ 泄漏：worker 永远在等 ch，没人取消它
func badWorker(ch chan int) {
    for {
        val := <-ch // 如果 ch 永远不关，这个 goroutine 永远不死
        process(val)
    }
}

// ✅ 修复：传入 ctx，支持取消
func goodWorker(ctx context.Context, ch chan int) {
    for {
        select {
        case val, ok := <-ch:
            if !ok {
                return // channel 关闭，正常退出
            }
            process(val)
        case <-ctx.Done():
            return // 上下文取消，主动退出
        }
    }
}
```

#### 案例2：Timer 未 Stop

```go
// ❌ 泄漏：定时任务每轮新建 timer，但旧 timer 一直没被 Stop
func tickBad(interval time.Duration, fn func()) {
    for {
        time.Sleep(interval) // 这种方式不会泄漏，但不精确
        fn()
    }
}

// ❌ 更隐蔽：每次都创建新 timer，旧的 never released
func tickWithLeak(interval time.Duration, fn func()) {
    for {
        timer := time.NewTimer(interval)
        <-timer.C // 收到信号后 timer 对象还活着！需要 Stop
        fn()
        // 忘记 timer.Stop() = 泄漏！每次循环创建一个 timer，never GC'ed
    }
}

// ✅ 修复：重用 timer，循环结束时 Stop
func tickFixed(interval time.Duration, fn func()) {
    timer := time.NewTimer(interval)
    for {
        <-timer.C
        fn()
        timer.Reset(interval) // 重用时用 Reset 而非新建
    }
}

// ✅ 更好的方式：Ticker 自动管理循环
func tickBest(interval time.Duration, fn func()) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop() // 配合 defer 确保退出时清理
    for range ticker.C {
        fn()
    }
}
```

#### 案例3：Context 未传递

```go
// ❌ 泄漏：后台任务用 Background，父请求取消了也不退
func handleRequest(w http.ResponseWriter, r *http.Request) {
    data := fetchFromDB(r.URL.Query().Get("id"))

    // 注意：这里用了 context.Background()，不是 request.Context()
    go func() {
        logProcess(data) // 即使用户取消了请求，这个 goroutine 还在跑
        notifyUser(data)
    }()

    w.Write([]byte("OK"))
}

// ✅ 修复：始终使用 request 的 context
func handleRequestGood(w http.ResponseWriter, r *http.Request) {
    data := fetchFromDB(r.URL.Query().Get("id"))

    go func() {
        ctx, cancel := context.WithTimeout(r.Context(), 30*time.Second)
        defer cancel()
        // 所有下游调用都带 ctx，任一超时都会停止整个流程
        logProcessCtx(ctx, data)
        notifyUserCtx(ctx, data)
    }()

    w.Write([]byte("OK"))
}
```

### 监控告警建议

```go
// 周期性采集 goroutine 数，超出阈值触发告警
func monitorGoroutines(stopCh <-chan struct{}) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    threshold := 5000 // 超过 5000 个 goroutine 报警
    for {
        select {
        case <-stopCh:
            return
        case <-ticker.C:
            count := runtime.NumGoroutine()
            if count > threshold {
                log.Warnf("HIGH GOROUTINE COUNT: %d (threshold: %d)", count, threshold)
                // 发送告警到 Prometheus + DingTalk
                alertGoGoroutines(count)
            }
        }
    }
}
```

### 面试话术

> "我们有一次服务上线后 goroutine 数从正常的 200 涨到 8000 多，导致 P99 飙升。通过 pprof goroutine profile 定位到一个异步通知的 goroutine 里用的是 `context.Background()`，用户取消请求后 goroutine 还在执行。修复方式是全局扫描代码，把所有后台 goroutine 的启动 context 改为 request context，并加上 timeout 控制。加了 goroutine 数监控告警后，这类问题可以在 1 分钟内发现。"

---

## 2. HTTP Response.Body 泄漏导致连接耗尽

### 面试官考察意图

这是一个非常经典但容易被忽视的问题。很多开发者只知道对 db.Query() 做 defer rows.Close()，却忘了 HTTP 响应的 Body 也需要关闭。这道题考察的是**资源管理的细节意识**。

### 为什么需要 Close Body

```
http.Client 的内部连接池依赖于 TCP 连接的复用。
如果不关闭 Body，连接会标记为"不可复用"，
每次请求都新建 TCP 连接 → 连接池耗尽 → 端口耗尽 → NewConnection refused
```

### 典型泄漏场景

```go
// ❌ 场景1：忽略了 Body
func getAPI(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    // 忘了 resp.Body.Close()！连接泄漏！
    io.ReadAll(resp.Body)
    return nil, nil // 连 error 都没返回
}

// ❌ 场景2：提前 return 了
func checkAPI(url string) (bool, error) {
    resp, err := http.Get(url)
    if err != nil {
        return false, err
    }
    if resp.StatusCode == 200 {
        return true, nil // 忘了 Close！
    }
    body, _ := io.ReadAll(resp.Body)
    fmt.Println(string(body))
    resp.Body.Close() // 只有在 else 分支才 close，return 时漏了
    return false, nil
}

// ❌ 场景3：defer 放错位置
func fetchAndParse(url string) (Result, error) {
    resp, err := http.Get(url)
    defer resp.Body.Close() // resp 可能为 nil（Get 失败），会 panic！
    if err != nil {
        return Result{}, err
    }
    var result Result
    json.NewDecoder(resp.Body).Decode(&result)
    return result, nil
}

// ❌ 场景4：短连接场景下不关闭 Body
func callManyEndpoints(urls []string) {
    client := &http.Client{} // 每个请求一个新 client？
    for _, url := range urls {
        resp, _ := client.Get(url)
        // 不 close body，短时间内大量请求会把 TCP 端口的 TIME_WAIT 用完
    }
}
```

### 标准模板

```go
// ✅ 正确的 HTTP 调用模板
func safeHTTPRequest(method, url string, body io.Reader) (*http.Response, error) {
    req, err := http.NewRequest(method, url, body)
    if err != nil {
        return nil, err
    }

    resp, err := httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    // 🔑 第一时间 defer close body
    defer resp.Body.Close()

    // 检查状态码（此时 body 已经保证了会被 close）
    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return resp, fmt.Errorf("unexpected status: %d", resp.StatusCode)
    }

    return resp, nil
}

// ✅ 读取全部响应体的安全写法
func readResponseBody(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    // 可选：限制读取大小，防止 big response 撑爆内存
    limitedReader := io.LimitReader(resp.Body, 10*1024*1024) // 最大 10MB
    return io.ReadAll(limitedReader)
}

// ✅ 只读状态码、不需要 body 的情况也要 Close
func headCheck(url string) error {
    req, _ := http.NewRequest("HEAD", url, nil)
    resp, err := httpClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close() // 即使是 HEAD，也要 Close！
    if resp.StatusCode != 200 {
        return fmt.Errorf("head failed: %d", resp.StatusCode)
    }
    return nil
}
```

### HTTP Client 最佳实践

```go
var httpClient = &http.Client{
    // 设置超时
    Timeout: 10 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,              // 空闲连接总数上限
        MaxIdleConnsPerHost: 10,               // 每个 host 的空闲连接数
        IdleConnTimeout:     90 * time.Second, // 空闲连接回收时间
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,
            KeepAlive: 30 * time.Second, // TCP KeepAlive
        }).DialContext,
        ForceAttemptHTTP2:     true,         // 强制 HTTP/2，减少连接数
        DisableCompression:    false,        // 开启 gzip 压缩
    },
}
```

### 面试话术

> "有一个批量调用的服务，同时向 100 个上游 API 发请求，结果运行几分钟后报 'no such host' 错误。排查发现是所有上游 API 的连接都变成了 TIME_WAIT 状态，操作系统端口用完了。根因是每个请求的 resp.Body 都没有 Close。修复后加上了统一的 Body Close 模板，并把 http.Client 复用成全局单例（配置 MaxIdleConnsPerHost），连接复用率从 0% 提升到 90% 以上。"

---

## 3. 文件句柄泄漏导致 EMFILE 崩溃

### 面试官考察意图

EMFILE（too many open files）在生产环境是常见的 OOM 级别故障，但很多人第一反应是改 ulimit 而不是找真正的泄漏点。这道题考察的是候选人能不能**区分症状和根因**。

### 故障表现

```
日志突然出现大量错误：open xxx: too many open files
进程直接退出，k8s restart
查看 lsof -p PID | wc -l 发现 fd 数量远超预期（几万甚至十几万）
```

### 典型泄漏场景

```go
// ❌ 场景1：for 循环中反复 open 不 close
func readFileAll(contentDir string) map[string]string {
    results := make(map[string]string)
    files, _ := os.ReadDir(contentDir)
    for _, f := range files {
        fp, _ := os.Open(filepath.Join(contentDir, f.Name()))
        // 忘了 fp.Close()！
        data, _ := io.ReadAll(fp)
        results[f.Name()] = string(data)
    }
    return results
}

// ❌ 场景2：错误路径上跳过了 defer
func processFile(path string) error {
    fp, err := os.Open(path)
    if err != nil {
        return err
    }
    // 假设后面有复杂的逻辑，多个 return 点
    if hasPermission(fp) {
        return nil // fp 没有 Close！
    }
    content := readContent(fp)
    err := transform(content)
    if err != nil {
        return err // 又一个没 Close 的路径
    }
    fp.Close() // 只有一个地方 close 了
    return nil
}

// ❌ 场景3：临时文件忘记清理
func tempUpload(filename string) error {
    tmp, _ := os.CreateTemp("", "*.tmp")
    // 忘了 defer os.Remove(tmp.Name())
    // 程序异常退出后临时文件残留在磁盘
    defer tmp.Close()
    io.Copy(tmp, os.Stdin)
    return os.Rename(tmp.Name(), filename)
}

// ❌ 场景4：io.Pipe 使用不当
func pipeReadWriter() {
    pr, pw := io.Pipe()
    go func() {
        pw.Write([]byte("data"))
        // pw.Close() 忘记了
    }()
    io.ReadAll(pr) // pr 也会一直等到 pw.Close()
}
```

### 标准模板

```go
// ✅ 通用打开文件的模板
func safeOpenFile(path string) (*os.File, error) {
    fp, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    return fp, nil
    // 调用方需要在适当的地方 defer fp.Close()
}

// ✅ 正确处理：defer 放在成功判断之后
func processFileCorrect(path string) error {
    fp, err := os.Open(path)
    if err != nil {
        return err
    }
    defer fp.Close() // 🔑 只要拿到 fp 就 defer close，不管后续有多少 return

    // 下面可以有任意数量的 return，fp 都会被正确关闭
    content, _ := io.ReadAll(fp)
    return doSomething(content)
}

// ✅ 临时文件：打开 + 删除一并 defer
func uploadSecure(src io.Reader) (string, error) {
    tmp, err := os.CreateTemp("", "upload-*")
    if err != nil {
        return "", err
    }
    defer func() {
        tmp.Close()                          // 无论如何都关掉文件
        os.Remove(tmp.Name())                 // 无论如何都删除临时文件
    }()

    io.Copy(tmp, src)
    tmp.Seek(0, io.SeekStart)
    dest, _ := os.Create("/path/to/dest")
    defer dest.Close()
    io.Copy(dest, tmp)

    return dest.Name(), nil
}

// ✅ 批量处理文件：使用 ioutil/readall 代替手动 open/close
func batchReadDir(dir string) (map[string][]byte, error) {
    entries, err := os.ReadDir(dir)
    if err != nil {
        return nil, err
    }

    results := make(map[string][]byte)
    for _, entry := range entries {
        if entry.IsDir() {
            continue
        }
        path := filepath.Join(dir, entry.Name())
        data, err := os.ReadFile(path) // 🔑 ReadFile 内部自动 manage open/close
        if err != nil {
            log.Printf("read error %s: %v", path, err)
            continue
        }
        results[entry.Name()] = data
    }
    return results, nil
}
```

### 诊断命令

```bash
# 查看某个进程的 fd 数量和类型
lsof -p $(pgrep your-go-binary) | head -50
lsof -p $(pgrep your-go-binary) | grep REG   # 只看普通文件 fd
lsof -p $(pgrep your-go-binary) | awk '{print $NF}' | sort | uniq -c | sort -rn | head

# 查看操作系统级别的 fd 限制
ulimit -n          # 当前 shell 的软限制
cat /proc/sys/fs/file-max  # 系统硬限制

# 查看系统中 TIME_WAIT 连接数量
ss -tan | grep TIME_WAIT | wc -l
```

### 面试话术

> "有一次凌晨服务报 EMFILE 退出，k8s 重启了三次。查 lsof 发现打开了 6 万个文件描述符，全是日志文件没有 close。原因是日志轮转函数里有个 bug：当写入失败时会 retry，但 retry 前没有 close 旧的 file handle。修复方法是给每个 open 的文件都加上 defer close，同时在 retry 逻辑显式 close 旧句柄后再重新 open。事后也在 prometheus 上加了 fd 用量监控，超过 10000 就触发告警。"

---

## 4. Data Race：并发读写竞态条件

### 面试官考察意图

Data Race 是 Go 并发编程中最隐蔽也最危险的 bug，平时测试很难复现，但一旦上线就会导致数据错乱甚至 panic。考察重点是：**怎么发现 → 怎么解决 → 怎么预防**。

### Data Race 的危害

```
🔴 严重性排序：
1. 数据错乱（最常见）：共享计数器值不对、map 读到脏数据
2. Panic（较少见）：concurrent map read and map write → fatal error
3. 安全风险（最危险）：认证状态被篡改、支付金额被修改

Data Race 不一定是 bug（值对了就行），但一定是缺陷（不可预测）。
```

### 检测工具

```bash
# Go 内置 race detector（编译时有约 10x overhead，仅用于开发/测试阶段）
go run -race main.go           # 运行时检测
go test -race ./...             # 单元测试检测
CGO_ENABLED=1 go build -race -o myapp .  # 编译启用

# Go 1.21+ 支持编译期检测（部分 unsafe 场景无法检测）
# 建议 CI/CD 中加入 -race 测试环节
```

### 典型竞态案例

#### 案例1：共享变量未加锁

```go
// ❌ 竞态：多个 goroutine 同时读写 counter
var counter int

func increment() {
    counter++ // ← 不是原子操作！(read -> add -> write)，会有 race
}

func getCount() int {
    return counter // ← 另一个 goroutine 可能在写，读到半个值
}

// ✅ 修复1：sync/atomic
import "sync/atomic"

var counter atomic.Int64

func incrementSafe() {
    counter.Add(1) // 🔒 原子操作，无竞态
}

// ✅ 修复2：mutex 保护
var (
    mu      sync.Mutex
    counter2 int
)

func incrementMutex() {
    mu.Lock()
    counter2++
    mu.Unlock()
}

// ✅ 修复3：channel 通信（Go 哲学：不要通过共享内存来通信）
var ch = make(chan int)

func worker(id int) {
    ch <- id // 每个 worker 通过 channel 发送结果，无需共享变量
}
```

#### 案例2：map 并发读写（必考！）

```go
var cache = make(map[string]string) // ❌ 不是并发安全的！

// ❌ 竞态：不同 goroutine 同时读写同一个 map
func writeCache(key, val string) {
    cache[key] = val
}

func readCache(key string) (string, bool) {
    v, ok := cache[key]
    return v, ok
}

// ✅ 修复1：sync.RWMutex
var (
    mu      sync.RWMutex
    cache2  = make(map[string]string)
)

func writeCacheSafe(key, val string) {
    mu.Lock()
    cache2[key] = val
    mu.Unlock()
}

func readCacheSafe(key string) (string, bool) {
    mu.RLock()
    defer mu.RUnlock()
    v, ok := cache2[key]
    return v, ok
}

// ✅ 修复2：sync.Map（适合读多写少、key 稳定的场景）
var sm = sync.Map{}

sm.Store("key", "value")
if v, ok := sm.Load("key"); ok {
    fmt.Println(v.(string))
}
```

#### 案例3：结构体字段并发访问

```go
type Stats struct {
    Requests int
    Errors   int
    Latency  float64
}

// ❌ 竞态：stats 被多个 handler 同时读取和更新
var stats Stats

func handler(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    // ... 处理请求 ...
    stats.Requests++       // 并发写
    elapsed := time.Since(start)
    stats.Latency = elapsed.Seconds() // 并发写
}

func getStats() Stats {
    return stats // 并发读，可能读到不一致的中间状态
}

// ✅ 修复：用原子操作分别更新每个字段
type AtomicStats struct {
    requests atomic.Int64
    errors   atomic.Int64
    latency  atomic.Float64 // Go 1.19+
}

func (s *AtomicStats) RecordLatency(d time.Duration) {
    s.requests.Add(1)
    s.latency.Store(d.Seconds())
}
```

### 竞态修复决策树

```
发现数据竞争？
├── 单个计数器/数值 → sync/atomic（Add, Load, Store, CompareAndSwap）
├── 数据结构整体读写 → sync.Mutex / sync.RWMutex
├── 读多写少、key 稳定 → sync.Map
├── 完全不需要共享 → 改用 channel 通信，各 goroutine 各自维护自己的状态
└── 不确定 → go test -race 验证修复后无报告
```

### 预防策略

```go
// 1. CI/CD 集成 -race 测试
// .github/workflows/test.yml:
# steps:
#   - run: go test -race ./...

// 2. 代码规范：静态分析
// golangci-lint 启用 misspell + staticcheck + gosimple 等

// 3. 架构层面：避免跨 goroutine 共享可变状态
// 优先用 channel 通信，而非共享内存
```

### 面试话术

> "我们的支付系统在压测时发现偶尔会出现金额计算错误。用 go test -race 跑完，发现一个统计 goroutine 的变量被并发读写——handler 在写 stats.Requests++，而 dashboard 在同一时间读整个 Stats 结构体，导致读到半更新的中间值。修复方式是拆成 atomic.Int64，每次读最新的值。后来在 CI 里加入了 `-race` 参数，所有 PR 合并前必须通过 race test。"

---

## 5. panic/recover 误用导致服务不可预测

### 面试官考察意图

Go 的 panic/recover 是最容易被滥用的特性。面试官想看你有没有**明确的工程纪律**：什么时候用 panic（真的不能恢复的情况），什么时候应该用 error 返回。这道题区分了"背过八股文的人"和"真正有生产经验的工程师"。

### panic vs Error：何时用哪个

```
✅ 用 panic 的场景（极少）：
   - 初始化阶段严重错误：配置缺失、数据库连不上、必需常量不对
   - 程序员 bug：断言失败了、不应该到达的代码路径被触发了
   - 不能恢复的系统级故障：内存不足（runtime.GoExit 也不行）

❌ 不要用 panic 的场景（绝大多数）：
   - HTTP 请求处理：应该返回 500 + error
   - 数据库查询失败：应该返回 error
   - 网络调用超时：应该返回 error
   - 任何外部输入校验：应该返回 validation error
```

### 常见误用模式

```go
// ❌ 误用1：把 panic 当 error handling
func divide(a, b int) int {
    if b == 0 {
        panic("division by zero") // ← 这应该在调用方检查！
    }
    return a / b
}

func handler(w http.ResponseWriter, r *http.Request) {
    result := divide(10, getUserInput()) // ← 每次都要 try-catch 风格 recover
    // ...
}

// ❌ 误用2：在最底层 recover，吞掉了有用信息
func ProcessData(data RawData) ProcessedData {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("something went wrong: %v", r) // ← 只打了日志，静默失败
        }
    }()
    return transform(data)
}

// ❌ 误用3：recover 里不调用 panic 继续传播，让上层根本不知道出过错
func handler(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if r := recover(); r != nil {
            w.WriteHeader(200) // ← 伪装成功，实际是错的！
            w.Write([]byte("ok"))
        }
    }()
    processData(r)
}

// ❌ 误用4：recover 只写在 main()，其他所有 panic 都让程序崩溃
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("panic:", r)
        }
    }()
    // 但 HTTP server 的 handler panic 在这里 recover 不到！
    srv.ListenAndServe()
}
```

### 正确使用模式

```go
// ✅ 原则1：顶层 recover（main/main goroutine / HTTP handler）
func main() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("FATAL PANIC recovered at top level: %v\n%s",
                r, debug.Stack()) // 输出栈信息
            // 考虑 graceful shutdown 而非静默退出
        }
    }()

    // 业务逻辑
    runApplication()
}

// ✅ 原则2：HTTP handler 里的 recover
func orderHandler(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("PANIC in orderHandler: %v\n%s", r, debug.Stack())
            http.Error(w, "internal server error", http.StatusInternalServerError)
        }
    }()

    // 业务处理 — 不应该用 panic，但如果真有 bug 能被 recover 住
    order := parseOrder(r)
    result := createOrder(order)
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(result)
}

// ✅ 原则3：初始化阶段的 panic（可以 panic，因为确实无法继续）
func initConfig() Config {
    cfg := Config{}
    cfg.DatabaseURL = os.Getenv("DATABASE_URL")
    if cfg.DatabaseURL == "" {
        panic("DATABASE_URL environment variable is required")
    }
    cfg.Port, _ = strconv.Atoi(os.Getenv("PORT"))
    if cfg.Port <= 0 {
        panic("PORT must be a positive number")
    }
    return cfg
}

// ✅ 原则4：error 返回替代 panic（大多数场景）
func divideSafely(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("divisor cannot be zero")
    }
    return a / b, nil
}

// 调用方决定怎么处理：重试？降级？还是返回错误给上层？
```

### 生产环境的 panic 监控

```go
import "runtime/debug"

func crashReporter() {
    defer func() {
        if r := recover(); r != nil {
            // 收集现场信息
            stackBuf := make([]byte, 64*1024)
            n := runtime.Stack(stackBuf, true) // true = 包含所有 goroutine

            log.Fatalf("=== PANIC RECOVERED ===\npanic: %v\nstack:\n%s",
                r, stackBuf[:n])

            // 在实际生产中，这里应该发送到 Sentry / Bugsnag 等监控系统
            // sentry.CaptureException(fmt.Sprintf("%v\n%s", r, stackBuf[:n]))
        }
    }()
}
```

### 面试话术

> "我们团队有一个明确规范：除了 main/init 初始化阶段，其他地方一律不用 panic。HTTP handler 可以用 recover 兜底防止单个请求的 bug 拖垮整个服务，但 recover 后必须记录完整栈信息并返回 500，绝不能吞掉错误。我们还上了 Sentry 自动收集 panic 信息。实际上线以来， recover 捕获的主要是两种情况：1）有人写错了空指针解引用（代码 review 发现的 bug）；2）第三方 SDK 内部的 bug。这两种都不是应用层面的错误，不能用 error 机制处理。"

---

## 6. TCP TIME_WAIT 过多导致端口耗尽

### 面试官考察意图

这是 Linux 网络层面的经典问题。当 Go 服务频繁创建新的 TCP 连接到下游时，如果没做好连接复用，会产生大量的 TIME_WAIT 状态，最终耗尽力口（ephemeral port space ~28000~65535）。这道题考察的是**对 TCP 状态机的理解和实际调优经验**。

### TCP TIME_WAIT 的原理

```
正常的 TCP 四次挥手：
  Client ──FIN──▶ Server
  Client ◀─FIN+ACK── Server
  Client ◀─ACK── Server
                    ↓
              Client 进入 TIME_WAIT 状态（等待 2×MSL，通常 60s）
              Server 直接关闭

为什么需要 TIME_WAIT？
1. 保证最后的 ACK 能到达对方（如果丢了，对方会重传 FIN）
2. 防止旧的连接报文混入新连接（network ghost packets）

问题：每个短连接都会产生一个 TIME_WAIT，如果 QPS 高：
  每秒 1000 个新连接 × 60 秒 = 60000 个 TIME_WAIT
  但 OS 可用端口只有 ~64000 个！→ 很快耗尽
```

### 症状诊断

```bash
# 查看 TIME_WAIT 数量
ss -tan | grep TIME-WAIT | wc -l
# 或者
netstat -an | grep TIME_WAIT | wc -l

# 查看哪些 IP 产生了最多 TIME_WAIT
ss -tan | grep TIME-WAIT | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head

# 查看系统的端口范围
cat /proc/sys/net/ipv4/ip_local_port_range
# 默认通常是 32768  60999（约 28000 个端口可用）

# 查看所有状态的连接数
ss -s
```

### 解决方案

#### 方案1：HTTP 连接复用（最根本的解决方式）

```go
// ❌ 每次请求新建 client → 每个请求都是独立的 TCP 连接
func callDownstream(url string) ([]byte, error) {
    client := &http.Client{Timeout: 5 * time.Second} // 新 client，新连接！
    resp, err := client.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}

// ✅ 复用 Transport 和 Client → 连接池化，TCP 复用
var defaultClient = &http.Client{
    Timeout: 5 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,          // 关键：每个 host 保持 10 个空闲连接
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 5 * time.Second,
    },
}

func callDownstreamOptimized(url string) ([]byte, error) {
    resp, err := defaultClient.Get(url) // 复用全局 client 的连接池
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}

// ✅ 对于高并发短连接场景：预建连接池
func warmUpConnections(client *http.Client, url string, count int) {
    conns := make([]*http.Response, count)
    for i := 0; i < count; i++ {
        resp, _ := client.Get(url)
        if resp != nil {
            resp.Body.Close()
            conns[i] = resp
        }
    }
    // 预热完成，Transport 里有 N 个已建立的 idle 连接
}
```

#### 方案2：Keep-Alive（HTTP/1.1 默认开启，但有时需要确认）

```go
// HTTP/1.1 默认 keep-alive，但某些中间件/proxy 会关闭它
// 确保你的 Transport 启用了 KeepAlive

transport := &http.Transport{
    DialContext: (&net.Dialer{
        Timeout:   5 * time.Second,
        KeepAlive: 30 * time.Second, // TCP 层 Keep-Alive 探测间隔
    }).DialContext,
    TLSHandshakeTimeout: 5 * time.Second,
    // MaxIdleConnsPerHost 设为非零值即可启用 HTTP 层 keep-alive
    MaxIdleConnsPerHost: 10,
}
```

#### 方案3：OS 内核调优（辅助手段）

```bash
# ⚠️ 修改 sysctl 参数需要 root 权限，不建议作为主要解决方案
# 仅在无法控制应用代码时才考虑

# 缩短 TIME_WAIT 时间（风险：可能导致旧数据包混入新连接）
sysctl -w net.ipv4.tcp_tw_reuse=1    # 允许 reuse TIME_WAIT socket（安全选项）
sysctl -w net.ipv4.tcp_fin_timeout=30  # 将默认的 60s 改为 30s

# 扩大 ephemeral 端口范围
sysctl -w net.ipv4.ip_local_port_range="1024 65535"

# 增大 backlog 队列
sysctl -w net.core.somaxconn=65535
```

> ⚠️ 注意：`tcp_tw_reuse` 在 Linux 5.x 后已经默认开启，无需手动设置。

### 面试话术

> "有一次压测后发现服务器 TIME_WAIT 到了 5 万多，导致新用户请求频繁报错。排查发现是 HTTP client 每次新建了一个实例，没有复用连接。把 client 改成全局单例并配置 MaxIdleConnsPerHost=10 后，连接复用率从接近 0 提升到 95%，TIME_WAIT 降到几百个。同时开启了 tcp_tw_reuse 做兜底。根本原因还是连接复用没做好，内核调参只是治标不治本。"

---

## 高频追问汇总

| 问题 | 核心答案 |
|------|---------|
| goroutine 数和内存的关系？ | 每个 goroutine 初始栈 2KB，可扩展到几 MB。10 万个 goroutine 理论上占用 200GB 内存，但在实际中由于大多是小任务快速结束，峰值通常在几十 MB 到几 GB。关键是长期存在的 goroutine 不能超过机器内存。 |
| go test -race 的性能开销？ | 运行时约 5-10x 的 CPU 开销，内存约 2-4x。因此只在 CI/开发和单元测试中使用，生产环境不开启。 |
| http.Client 要不要自己管 Transport？ | 如果要复用连接就必须自己配置 Transport。简单情况下可以用 http.DefaultClient，但它没有超时设置，推荐使用自定义的全局 client。 |
| EMFILE 和 ENFILE 的区别？ | EMFILE = 单个进程的 fd 数超过了 ulimit 限制（常见于文件泄漏）。ENFILE = 整个系统的 fd 数超过了 fs.file-max（罕见，通常是大量进程同时泄漏）。 |
| TCP TIME_WAIT 一定要消除吗？ | 不一定。如果你的 QPS 不高（比如 < 1000/s），几个 TIME_WAIT 连接不会影响。但批量调用/代理服务的场景，连接复用是必须的。 |
| 为什么 Go 不推荐大量用 panic？ | 因为 recover 的语法成本高（必须在每个调用链的顶层），而且 panic 会导致 deferred function 全执行一遍，影响性能。error 返回值更清晰、更可组合。 |
