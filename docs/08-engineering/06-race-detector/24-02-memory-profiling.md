# 内存深度分析：alloc_space vs inuse_space vs mallocs

> 考察频率：★★★★★  难度：★★★★★
> 关键词：pprof、alloc_space、inuse_space、mallocs、对象生命周期、GC 压力、分配热点

## 面试官考察意图

OOM 排查文章只讲了"用 pprof/heap"，但**没有讲清楚 pprof 中不同 view 的区别和各自的应用场景**。高级工程师必须知道：当线上出现内存问题时，该看哪个 profile、哪个 metric、为什么。

这道题考的是：**你能不能从内存行为的角度诊断问题，而不是只会说"内存泄漏了重启就行"。**

---

## 核心答案（30 秒版）

Go pprof 的内存 profile 有三种关键 view：`inuse_space` 是当前活跃占用（适合定位泄漏），`alloc_space` 是累计分配量（适合定位 GC 压力和性能瓶颈），`mallocs/inuse_objects` 是分配次数统计（适合找出高频分配模式）。一个典型的流程是：先看 `inuse_space` 找泄漏 → 再用 `alloc_space` 对比基线找增量分配 → 最后用 `-alloc_space` 而非 `-inuse_space` 来做 flame graph，因为 `inuse_space` 可能把已回收对象的调用栈掩盖掉。

---

## 三种 View 的本质区别

### 理解 Go 堆内存的生命周期

```
时间线 ────────────────────────────────────────────────►

         alloc               free/return to heap
           ↓                   ↓
         [████████]          [████████]
         
         ↑                    ↑
      alloc_space         inuse_space
      累计分配             当前持有
      
      +---------------------+
                  ↓
              mallocs count
          分配次数统计
```

| Metric | 含义 | 适用场景 |
|--------|------|---------|
| **inuse_space** | 当前驻留在堆上的字节数 | 找泄漏：哪些对象一直不被释放 |
| **inuse_objects** | 当前驻留在堆上的对象个数 | 同上，按对象计数 |
| **alloc_space** | 累计分配总量（含已回收） | 找性能瓶颈：哪些函数分配最多 → GC 压力大 |
| **alloc_objects** | 累计分配总次数 | 同上，按次数统计 |

### 核心洞察

> **inuse_space 高 ≠ alloc_space 高**：如果一个函数频繁创建并释放小对象，inuse_space 可能很低（因为都及时被 GC 回收了），但 alloc_space 很高（累计分配量惊人）。这种情况下程序不会 OOM，但 GC 会非常频繁 → TP99 飙升。

> **alloc_space 高 ≠ inuse_space 高**：很多服务长期处于这种状态——他们不是有泄漏，而是设计上就是"大量临时对象 → GC 回收"的模式。这类问题的优化方向是减少分配（复用对象池、预分配缓冲区），而不是修 leak。

---

## 实战：如何解读 pprof 输出

### Step 1：确定你要找什么

```bash
# 情况 A：怀疑泄漏 → 看 inuse_space
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/heap

(pprof) top -inuse_space
# 如果某个函数的 inuse_space 持续增长 → 泄漏

# 情况 B：GC 太频繁 → 看 alloc_space
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/heap?alloc_space

(pprof) top -alloc_space
# 如果某个函数的 alloc_space 极高但 inuse_space 正常 → 高频分配，需要减少分配
```

### Step 2：对比两次采样找差异

```bash
# 1. 采集两次 heap
curl -s "http://localhost:6060/debug/pprof/heap?gc=1" > heap_before.prof
# ... 等待一段时间或执行一些操作 ...
curl -s "http://localhost:6060/debug/pprof/heap?gc=1" > heap_after.prof

# 2. 对比
go tool pprof -base=heap_before.prof heap_after.prof

# 3. 查看增长最多的调用栈
(pprof) top 10
# 重点看 inuse_space 显著增长的函数
```

### Step 3：解读 PPROF 输出的关键列

```
(pprof) top 10 -alloc_space

Flat    Flat%   Sum%     Cum    Cum%  Function
500MB   45.5%   45.5%   500MB   45.5%  github.com/xxx.UnmarshalJSON
300MB   27.3%   72.8%   800MB   72.8%  github.com/xxx.ProcessRequest  
200MB   18.2%   90.9%   200MB   18.2%  github.com/xxx.createTempSlice
100MB    9.1%  100.0%   500MB   45.5%  main.handleAPI

解释：
Flat        = 该函数自身分配的（不含子函数）
Cum         = 该函数及所有子函数累计分配的
Flat%       = 占总 allocation 的百分比
Sum%        = 累加百分比（从上到下逐行累加到 100%）
```

---

## 常见场景与修复方案

### 场景 1：未泄漏但 GC 压力大（alloc_space 高，inuse_space 低）

```go
// ❌ 问题：每次请求都创建新 slice → 大量短暂分配
func getItems(ctx context.Context) ([]Item, error) {
    items := make([]Item, 0, 100) // 每次都是新分配
    for _, raw := range fetchFromDB(ctx) {
        item := parse(raw)
        items = append(items, item) // 反复扩容触发更多分配
    }
    return items, nil
}

// ✅ 修复：使用 sync.Pool 复用 slice
var slicePool = sync.Pool{
    New: func() any { return make([]Item, 0, 100) },
}

func getItemsFixed(ctx context.Context) ([]Item, error) {
    buf := slicePool.Get().(*[]Item)
    defer func() {
        *buf = (*buf)[:0]
        slicePool.Put(buf)
    }()
    
    for _, raw := range fetchFromDB(ctx) {
        *buf = append(*buf, parse(raw))
    }
    
    result := make([]Item, len(*buf))
    copy(result, *buf)
    return result, nil
}
```

### 场景 2：Map 泄漏（inuse_space 持续走高）

```go
// ❌ 全局 map 只增不减
var metrics = make(map[string]int64)

func record(name string, value int64) {
    metrics[name] = value  // 永远不会清理
}

// ✅ 修复 1：定期清理
var (
    mu      sync.Mutex
    metrics = make(map[string]int64)
)

func recordFixed(name string, value int64) {
    mu.Lock()
    defer mu.Unlock()
    metrics[name] = value
    
    // 每 1000 个 key 清理一次
    if len(metrics) > 1000 {
        cleanOldMetrics()
    }
}

// ✅ 修复 2：用 TTL cache 替代
type cacheEntry struct {
    value     int64
    expiresAt time.Time
}

var cache = ttlcache.New(...)
cache.Set("cpu", 75, 5*time.Minute)
```

### 场景 3：json 解码时的临时分配爆炸

```go
// ❌ 每个 JSON 对象创建新的 map/slice，累积 GC 压力
func processUsers(data []byte) ([]User, error) {
    var users []User
    err := json.Unmarshal(data, &users)
    return users, err
}

// ✅ 分析：如果 data 很大且处理频率高，unmarshal 可能是最大 allocator
// 解决方案：改用 streaming decoder
func processUsersFixed(data []byte) ([]User, error) {
    dec := json.NewDecoder(bytes.NewReader(data))
    dec.Token()  // skip [{
    
    var users []User
    for dec.More() {
        var u User
        if err := dec.Decode(&u); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    return users, nil
}

// 或者直接用 json-iterator / ffjson 等更快的库
```

### 场景 4：字符串拼接导致的隐藏分配

```go
// ❌ 看起来无害，但每轮循环都创建新字符串
func buildLog(msg string) string {
    result := "[" + time.Now().Format(time.RFC3339) + "] " + msg
    return result
}

// ✅ 修复：strings.Builder
func buildLogFixed(msg string) string {
    sb := strings.NewReplacer("", "") // 不适用这里
    var b strings.Builder
    b.Grow(50 + len(msg))
    b.WriteByte('[')
    time.Now().AppendFormat(&b, time.RFC3339)
    b.WriteString("] ")
    b.WriteString(msg)
    return b.String()
}
```

---

## pprof 进阶技巧

### 使用 `-inuse_space` vs `-alloc_space` 选择

```bash
# 找泄漏（对象未被释放）
go tool pprof -top -inuse_space heap.prof

# 找 GC 压力（大量分配导致频繁 GC）
go tool pprof -top -alloc_space heap.prof

# 火焰图也支持切换
go tool pprof -http=:8080 -alloc_space http://localhost:6060/debug/pprof/heap
```

### `-count` 参数控制样本数

```bash
# 默认 1 秒采集，可以延长
curl "http://localhost:6060/debug/pprof/heap?seconds=60" > heap_60s.prof

# -gc=1 强制先触发 GC 再采样（更准确地反映活跃内存）
curl "http://localhost:6060/debug/pprof/heap?gc=1" > heap_gc.prof
```

### 结合 MemStats 做运行时监控

```go
import "runtime"

func monitorMem(interval time.Duration) {
    ticker := time.NewTicker(interval)
    var prev runtime.MemStats
    ticker.C:
    for {
        <-ticker.C
        
        var m runtime.MemStats
        runtime.ReadMemStats(&m)
        
        delta := m.TotalAlloc - prev.TotalAlloc
        log.Printf(
            "mem: Alloc=%.1fMB Sys=%.1fMB NumGC=%d AllocRate=%.1fMB/s",
            float64(m.Alloc)/1024/1024,
            float64(m.Sys)/1024/1024,
            m.NumGC,
            float64(delta)/interval.Seconds()/1024/1024,
        )
        
        prev = m
    }
}
```

---

## 生产案例：从 pprof 发现真实泄漏

```
背景：某 API 服务上线后 RSS 每天涨 2%，一个月后 OOM restart

排查过程：

1. 初始判断："RSS 上涨 → 肯定是内存泄漏"
   go tool pprof -inuse_space heap.prof → top:
   UnmarshalJSON 占了 60%
   
2. 怀疑：json 解码有问题？再看 alloc_space
   go tool pprof -alloc_space heap.prof → top:
   ProcessEvent 占了 45%，UnmarshalJSON 只有 10%
   
3. 对比发现：inuse_space 中 UnmarshalJSON 很高，
   但 alloc_space 中 ProcessEvent 更高
   
4. 结论：**json 本身没 leak，是 ProcessEvent 创建的大量
   中间结构体中的某个字段引用了大对象，导致 gc 无法回收
   
5. 进一步 grep 代码：发现 ProcessEvent 里有个 
   debugInfo[requestID] = largePayload 的全局 map，
   requestID 不重复 → 永远增长

修复：加上 cleanup timer，每小时清理 24h 前的记录。
RSS 涨幅从 2%/天降到 <0.1%/天。
```

---

## 高频追问

**Q：什么时候该看 inuse_space，什么时候该看 alloc_space？**
A：在找内存泄漏时看 inuse_space（谁占着不放），在调优 GC 压力时看 alloc_space（谁分配最多）。两者结合起来才能全面理解内存行为。

**Q：`pprof -alloc_space` 火焰图中，flat 和 cum 分别代表什么？**
A：flat 是该函数自身直接分配的，cum 是该函数及它调用的所有子函数累计分配的。如果 flat 很小但 cum 很大，说明瓶颈在下面调用链的某个函数。

**Q：GOGC/GOMEMLIMIT 怎么调？**
A：GOGC 默认 100（上次 GC 后存活对象翻倍时触发）。设为 200 会减少 GC 频率但增加峰值内存。GOMEMLIMIT 是 Go 1.19+ 引入的硬上限（如 GOMEMLIMIT=4GiB），比 GOGC 更精确，推荐优先用 GOMEMLIMIT。

**Q：如何验证修复效果？**
A：修复前采一份 heap.prof → 修复后同样的 workload 再采一份 → 用 `go tool pprof -base=before after` 对比。如果目标函数的 inuse_space 降下去了，就证明有效。

---

## 延伸阅读

- [PPROF documentation](https://pkg.go.dev/runtime/pprof)
- [Go Blog: Debugging Memory Leaks](https://go.dev/blog/pprof)
- [Uber Go Style Guide: Profiling](https://github.com/uber-go/guide/blob/master/style.md#profiling)
