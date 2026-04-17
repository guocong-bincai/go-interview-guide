# pprof 性能调优实战

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：CPU Profile、Memory Profile、Goroutine Profile、火焰图、go tool pprof

## 🎯 面试官考察意图

- 候选人是否能在真实生产环境中定位性能瓶颈
- 是否理解各种 Profile 的区别和使用场景
- 是否掌握火焰图阅读能力，能否从火焰图中快速定位热点
- 是否有系统性调优思路，而不是靠猜

---

## ⚡ 核心答案（30秒）

> pprof 是 Go 标准库的 Profiling 工具，通过 `net/http/pprof` 暴露端点。主要看四种 Profile：**CPU**（哪个函数占用 CPU 最多）、**Heap**（谁分配的内存最多）、**Goroutine**（哪里的 goroutine 最多）、**Mutex**（哪里的锁竞争最严重）。生产环境用 `go tool pprof -http=:8080 profile_file` 生成火焰图，直观看出调用栈深度和宽度。

---

## 🔬 深度展开

### 1. pprof 快速上手

#### 引入端点

```go
// 方式1：在已有 http server 上挂载（推荐）
import (
    _ "net/http/pprof"
    "net/http"
)

func main() {
    // 你的业务路由
    mux := http.NewServeMux()
    mux.HandleFunc("/api/", apiHandler)
    
    // pprof 路由（通常绑定到仅内网可访问的端口）
    go func() {
        http.ListenAndServe(":6060", nil)
    }()
    
    // 主服务
    http.ListenAndServe(":8080", mux)
}
```

```bash
# 方式2：独立端口
import "net/http/pprof"

func main() {
    go func() {
        http.ListenAndServe(":6060", pprof.Handler())
    }()
}
```

#### 四种核心 Profile

```bash
# ① CPU Profile（30秒采样，Go 程序默认每 100Hz 采样一次，即每秒100次）
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof

# ② Heap Memory Profile（按alloc 或 inuse 统计）
curl http://localhost:6060/debug/pprof/heap > heap.prof

# ③ Goroutine Profile（当前所有 goroutine 的堆栈）
curl http://localhost:6060/debug/pprof/goroutine > goroutine.prof

# ④ Mutex Profile（锁竞争采样）
curl http://localhost:6060/debug/pprof/mutex > mutex.prof

# ⑤ 查看所有可用的 profile 类型
curl http://localhost:6060/debug/pprof/
```

---

### 2. CPU Profile 分析

#### 场景：接口响应慢，CPU 占用高

```go
package main

import (
    "encoding/json"
    "net/http"
    _ "net/http/pprof"
    "sync"
    "time"
)

// 模拟慢查询 + JSON 序列化热路径
var cache = make(map[string][]byte)
var mu sync.RWMutex

func heavyHandler(w http.ResponseWriter, r *http.Request) {
    key := r.URL.Query().Get("key")
    
    // 热路径1：读缓存（RWMutex 竞争）
    mu.RLock()
    data, ok := cache[key]
    mu.RUnlock()
    
    if !ok {
        // 热路径2：模拟 DB 查询延迟
        time.Sleep(50 * time.Millisecond)
        data = []byte("computed result for " + key)
        
        // 热路径3：写缓存
        mu.Lock()
        cache[key] = data
        mu.Unlock()
    }
    
    // 热路径4：JSON 序列化（CPU 密集）
    json.Marshal(map[string]interface{}{
        "data":      string(data),
        "timestamp": time.Now().Unix(),
        "count":     12345,
    })
    
    w.Write([]byte("ok"))
}

func main() {
    http.HandleFunc("/heavy", heavyHandler)
    go func() { http.ListenAndServe(":6060", nil) }()
    http.ListenAndServe(":8080", nil)
}
```

```bash
# 采集 30 秒 CPU Profile（在压测或生产流量下）
wrk -t4 -c100 -d30s http://localhost:8080/heavy?key=test
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof

# 分析
go tool pprof -http=:9090 cpu.prof
# 浏览器打开 http://localhost:9090 查看火焰图
```

#### CPU 火焰图阅读

```
火焰图方向：底部是入口，顶部是热点
宽度：CPU 占用比例（越宽越热）
顶尖：最热函数

        [json.Marshal]           ← 顶尖，CPU最热点
                |
    [heavyHandler]               ← 调用者
            |
    [serveHTTP]                  ← net/http
                |
    [main]                       ← 入口
```

```
常见 CPU 问题模式：

① "平顶山"（Flat top）：某个函数占用特别多 CPU
   → 优化这个函数本身（算法、缓存）

② "火焰山"（Wide plate）：多个函数都很热
   → 整体架构问题，考虑减少调用次数

③ "尖刺"（Spike）：某次采样捕获的调用栈
   → 大概率是 GC 或 runtime 开销
```

#### top + list 命令行分析

```bash
go tool pprof cpu.prof

# top 10 - 占用CPU最多的函数
(pprof) top 10
Showing nodes accounting for 12.34s, 82% of 15.06s total
  flat  flat%   sum%   cum    cum%
  5.21s  34.6%  34.6%  5.21s  34.6%  encoding/json.Marshal
  2.10s  13.9%  48.5%  2.10s  13.9%  runtime.mallocgc
  1.80s  12.0%  60.5%  1.80s  12.0%  heavyHandler
  ...

# 查看特定函数的调用栈详情
(pprof) list heavyHandler
Total: 1.82s
ROUTINE = heavyHandler in .../main.go:XX
  1.80s  1.80s  (flat%  cum%)
         1.80s  1.80s  encoding/json.Marshal  ← JSON序列化占用最多

# 查看函数的调用者（谁在调用这个函数）
(pprof) peek heavyHandler
```

---

### 3. Heap Memory Profile 分析

#### 两种统计视角

```bash
# alloc_space：自程序启动以来累计分配的内存（含已回收的）
go tool pprof -alloc_space heap.prof

# inuse_space：当前仍然活跃的内存（排查泄漏用这个）
go tool pprof -inuse_space heap.prof

# inuse_objects：当前活跃对象数量
go tool pprof -inuse_objects heap.prof
```

#### 场景：内存持续增长（内存泄漏）

```go
// 泄漏场景：goroutine 本地缓存持续增长
func leakyHandler(w http.ResponseWriter, r *http.Request) {
    // 每个请求分配一个 slice，请求结束后本应回收
    // 但如果有 closure 引用了 slice，会导致无法回收
    data := make([]byte, 1024*1024) // 1MB
    
    go func() {
        // 模拟异步处理，此时 data 仍被引用
        time.Sleep(10 * time.Second)
        process(data) // goroutine 结束后 data 才释放
    }()
    
    // 但这里有个bug：如果请求超时，goroutine 可能永远不调用 process
    // data 就泄漏了（更糟糕的情况是 context 泄漏导致整个链路无法GC）
}
```

```bash
# 对比两个时间点的 heap profile
curl http://localhost:6060/debug/pprof/heap > heap_10min.prof
# ... 运行 30 分钟 ...
curl http://localhost:6060/debug/pprof/heap > heap_40min.prof

# 对比增长
go tool pprof -http=:9090 heap_40min.prof
# 在 top 界面看 flat 最大的函数，即为泄漏点

# 或用 diff 功能
go tool pprof -diff_base=heap_10min.prof heap_40min.prof
```

#### 内存火焰图阅读

```
memory flame graph 的含义：
每个函数的 width = 该函数直接或间接分配的内存总量
颜色通常用暖色（红色/橙色）标注

        [sync.(*Pool).Get]              ← 从 pool 获取对象
                |                         ← 实际上不分配
    [makeSlice]                          ← 真正分配的地方
            |
    [readFile]                           ← 调用链
                    |
    [main.apiHandler]
```

---

### 4. Goroutine Profile 分析

#### 场景：goroutine 数量爆炸

```bash
# 采样所有 goroutine 的堆栈
curl http://localhost:6060/debug/pprof/goroutine?debug=1 > goroutine.txt

# 或生成 profile 用 pprof 分析
curl http://localhost:6060/debug/pprof/goroutine > goroutine.prof
go tool pprof -http=:9090 goroutine.prof
```

#### goroutine 泄漏常见模式

```go
// 模式1：channel 发送阻塞
func badProducer() {
    ch := make(chan int, 10)
    go func() {
        for i := 0; ; i++ {
            ch <- i  // ch 满后阻塞，永不退出
        }
    }()
    // goroutine 泄漏
}

// 模式2：context 未取消
func badWithContext(ctx context.Context) {
    for i := 0; ; i++ {
        select {
        case <-ctx.Done():
            return
        default:
            doWork(i)
        }
    }
}

// 模式3：time.Ticker 未 stop
func badTicker() {
    ticker := time.NewTicker(time.Second)
    go func() {
        for {
            <-ticker.C  // ticker.C 永不平仓，goroutine 泄漏
        }
    }()
}

// 修复
func goodTicker() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop() // 记得 defer Stop
    go func() {
        for {
            select {
            case <-ticker.C:
                doWork(time.Now().Unix())
            case <-stopChan:
                return
            }
        }
    }()
}
```

#### 分析 goroutine profile

```bash
# 查看 goroutine 堆栈
curl http://localhost:6060/debug/pprof/goroutine?debug=1 | head -100

# goroutine profile 格式说明
# goroutine count profile
# 格式：goroutine 数量  @ 堆栈帧
# 示例：
# 500 @  # 说明有 500 个 goroutine 的堆栈是下面这个

# 查看具体 goroutine 泄漏
go tool pprof goroutine.prof
(pprof) top
# 看 cum% 最大的函数，即创建最多 goroutine 的地方
(pprof) traces
# 打印所有 goroutine 的堆栈 traces
```

---

### 5. Mutex Profile 分析

#### 场景：锁竞争严重

```go
// 高并发下 RWMutex 的写锁竞争
var globalCache = make(map[string]interface{})
var mu sync.RWMutex

func getWithLock(key string) interface{} {
    mu.RLock()              // 读锁，所有读可以并发
    defer mu.RUnlock()
    return globalCache[key] // 这个操作很快
}

func setWithLock(key string, val interface{}) {
    mu.Lock()               // 写锁，阻塞所有读
    defer mu.Unlock()
    globalCache[key] = val
}

// 问题：如果写操作频繁，所有读操作都会被阻塞
// 解决方案：用 sync.Map 或分片锁
```

```bash
# 采集 mutex profile（采样有锁竞争的堆栈）
curl http://localhost:6060/debug/pprof/mutex > mutex.prof
go tool pprof -http=:9090 mutex.prof
```

---

### 6. 生产环境采集最佳实践

#### 安全采集建议

```go
// 生产环境：pprof 绑定到仅内网可访问的端口
import (
    "log"
    "net/http"
    "net/http/pprof"
)

func startPprof(addr string) {
    mux := http.NewServeMux()
    mux.HandleFunc("/debug/pprof/", pprof.Index)
    mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
    mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
    mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
    mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
    
    srv := &http.Server{
        Addr:         addr,
        Handler:      mux,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 70 * time.Second, // profile 可能耗时较长
    }
    
    go func() {
        if err := srv.ListenAndServe(); err != nil {
            log.Printf("pprof server stopped: %v", err)
        }
    }()
}

// 建议绑定 127.0.0.1:6060 或仅 Kubernetes ClusterIP
```

#### benchmark 中使用 pprof

```go
// 在 benchmark 中自动生成 CPU 和 Memory Profile
// go test -bench=. -benchmem -cpuprofile=cpu.prof -memprofile=mem.prof

// 使用 ppфprof 查看
// go tool pprof -http=:9090 cpu.prof

// 自动对比不同实现的性能
// go test -bench=. -benchmem -cpuprofile=cpu1.prof -count=1
// [修改代码后]
// go test -bench=. -benchmem -cpuprofile=cpu2.prof -count=1
// go tool pprof -diff_base=cpu1.prof cpu2.prof
```

#### 自动化采集脚本

```bash
#!/bin/bash
# deploy_profiling.sh - 生产部署时自动采集基线 profile

ADDR="localhost:6060"
OUTDIR="/tmp/profiles/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$OUTDIR"

echo "采集 pprof 基线数据到 $OUTDIR"

# CPU Profile（30秒）
curl -s "http://${ADDR}/debug/pprof/profile?seconds=30" > "$OUTDIR/cpu.prof"

# Heap Profile
curl -s "http://${ADDR}/debug/pprof/heap" > "$OUTDIR/heap.prof"

# Goroutine Profile
curl -s "http://${ADDR}/debug/pprof/goroutine" > "$OUTDIR/goroutine.prof"

# Mutex Profile
curl -s "http://${ADDR}/debug/pprof/mutex" > "$OUTDIR/mutex.prof"

echo "Profile 采集完成！"
echo "分析命令：go tool pprof -http=:9090 $OUTDIR/cpu.prof"
```

### 5. Go 1.26 新变化：pprof Web UI 默认火焰图

> Go 1.26（2026-02）将 pprof Web UI 的默认视图从**有向图（Graph）**改为**火焰图（Flame Graph）**，并移除了旧版 `go tool doc` / `go tool pprof doc` 命令（合并至 `go doc`）。

**变化对比：**

| 版本 | `go tool pprof -http=:9090` 默认视图 |
|------|-------------------------------------|
| Go 1.25 及之前 | 有向图（Graph），需手动切换到火焰图 |
| Go 1.26 及之后 | **火焰图（Flame Graph）** 默认展示 |

```bash
# Go 1.26 及之后：直接生成火焰图，无需额外操作
go tool pprof -http=:9090 cpu.prof
# 浏览器自动打开火焰图

# 切换到旧版有向图：View → Graph（或访问 /ui/graph）
```

**面试提示：** 如果面试官问到「pprof 火焰图怎么生成」，Go 1.26 之后的答案是「直接 `go tool pprof -http=:9090 profile_file`，默认就是火焰图」；Go 1.25 及之前需要加 `-火焰图` 参数。

---

## ❓ 高频追问

**Q：pprof 本身对性能的影响？**

> pprof 本身有采样开销（CPU profile 每 100Hz 采样一次，开销 < 5%），Heap profile 几乎无额外开销。但 Goroutine/Mutex profile 在高并发下（>10万 goroutine）会有一定开销。建议生产环境开启但不频繁访问，用完即关闭。

**Q：火焰图中"gcBgMarkWorker"占很多正常吗？**

> 正常，GC worker 本身会消耗 CPU。关键看它和业务代码的比例。如果 GC 消耗 > 20% CPU，说明 GC 太频繁或 STW 时间太长，需要调优 GOGC 或减少内存分配频率。

**Q：top 命令显示的 flat 和 cum 有什么区别？**

> flat = 函数自身的执行时间（不含调用的子函数）；cum = 累计时间（含所有子函数调用）。如果 flat ≈ cum，说明这个函数是叶子节点（没有子调用）。如果 cum >> flat，说明函数本身不慢但它调用的函数慢。

**Q：生产环境可以直接 curl 线上 pprof 吗？**

> 不建议。pprof 会Stop The World采集，线上直接采集可能造成延迟毛刺。建议：1）在本地压测环境复现问题；2）或在 Kubernetes 中用 exec 进容器采集；3）设置采样时间短一点（5秒而非30秒）。

---

## 📚 参考资料

- [Go pprof 官方文档](https://pkg.go.dev/net/http/pprof)
- [Go 性能分析工具](https://go.dev/blog/pprof)
- [火焰图作者 Brendan Gregg 的 profile 教程](http://www.brendangregg.com/flamegraphs.html)
- [go tool pprof 命令详解](https://github.com/gperftools/gperftools/wiki/pprof.1)
