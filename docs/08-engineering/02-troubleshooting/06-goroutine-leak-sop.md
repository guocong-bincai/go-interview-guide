[🏠 首页](../../../README.md) · [📦 工程素养](../../README.md) · [🔧 线上问题排查](../README.md)

---

# Goroutine 泄漏排查 SOP

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：pprof、runtime.NumGoroutine、goroutine profile、leak hunt

## 面试官考察意图

goroutine 泄漏是 Go 线上故障的高频诱因，面试中考察的是**系统性排查能力**。
初级候选人只会说"用 pprof 看"，高级候选人要能讲清楚**从告警触发→定位泄漏点→根因分析→修复验证的完整闭环**，并能结合生产经验说出预防措施。

---

## 核心答案（30 秒版）

goroutine 泄漏排查四步：

```
① 告警触发：监控发现 goroutine 数量异常上涨
② 采集证据：pprof goroutine profile + /debug/pprof/goroutine?debug=1
③ 定位泄漏点：对比两份 profile diff，定位泄漏的调用栈
④ 修复验证：修复后压测/观察，确认 goroutine 数量回落
```

泄漏的根因通常是：**channel 无人读取/关闭、mutex 未解锁、for 循环错误**。

---

## SOP 详细步骤

### 第一步：告警触发与初步确认

**监控告警**（推荐 Prometheus + Grafana）：

```yaml
# Prometheus 告警规则
- alert: GoroutineLeak
  expr: go_goroutines > 50000  # 根据历史基线调整阈值
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Goroutine 数量异常，当前 {{ $value }}"
    runbook_url: "https://wiki.example.com/runbooks/goroutine-leak"
```

**初步确认是否真的是泄漏**（排除流量突增）：

```bash
# 查看当前 goroutine 总数
curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | grep -c "^goroutine"

# 对比 5 分钟前的数量
# 如果数量持续上涨（不看流量是否上涨），则是泄漏
watch -n 5 'curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | grep -c "^goroutine"'
```

> **判断标准**：流量平稳但 goroutine 持续上涨 → 泄漏。流量和 goroutine 同步上涨 → 正常扩容。

---

### 第二步：采集 goroutine profile

```bash
# 方式 1：pprof HTTP 接口（生产推荐，有符号信息）
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/goroutine

# 方式 2：下载原始 profile 文件
curl http://localhost:6060/debug/pprof/goroutine > /tmp/goroutine_$(date +%s).prof

# 方式 3：debug=1 文本输出（快速看调用栈）
curl "http://localhost:6060/debug/pprof/goroutine?debug=1" > /tmp/goroutine_debug.txt
```

> **生产注意**：pprof 采集本身有轻微开销（约 1-5ms 暂停），对延迟敏感的服务使用 `-scope=inuse` 或设置采样率。

---

### 第三步：定位泄漏点

#### 方法 A：goroutine profile 火焰图对比（推荐）

```bash
# 1. 正常时刻采集基线
curl http://localhost:6060/debug/pprof/goroutine > /tmp/goroutine_baseline.prof

# 2. 泄漏时刻采集
curl http://localhost:6060/debug/pprof/goroutine > /tmp/goroutine_leak.prof

# 3. 对比两份 profile（最有效）
go tool pprof -base=/tmp/goroutine_baseline.prof /tmp/goroutine_leak.prof
(pprof) top 10
# 哪个函数占用 goroutine 数突增，就是泄漏点

# 4. 生成 diff 火焰图
go tool pprof -http=:9090 -diff_base=/tmp/goroutine_baseline.prof /tmp/goroutine_leak.prof
# 红色 = 泄漏新增的调用栈
```

#### 方法 B：debug=1 文本分析（快速粗筛）

```bash
# 统计每个调用栈出现的次数
grep -A1 "^goroutine" /tmp/goroutine_debug.txt | sort | uniq -c | sort -rn | head -20

# 查找异常的 channel 等待
grep -B5 "chan receive" /tmp/goroutine_debug.txt | head -40

# 查找疑似死循环
grep "for {" /tmp/goroutine_debug.txt | head -10
```

#### 方法 C：Go 1.26+ 内置 Goroutine Leak Profile（最精准）

```go
// 开启方法：在服务启动时添加
import _ "runtime/trace"

// 或者通过 GODEBUG设置
// GODEBUG=gctrace=1,trace=1
```

```bash
# Go 1.26+ 可以直接生成泄漏报告
go tool trace /tmp/trace.out
# 在 Web UI 中查看 "Goroutine Analysis" 面板
```

---

### 第四步：常见泄漏根因与修复

#### 根因 1：channel 无人读取（最常见）

**泄漏代码（生产踩坑案例）**：

```go
// ❌ 错误：请求处理函数中启动 goroutine 发送结果，但调用方从未读取
func HandleRequest(ctx context.Context, ch chan Result) {
    result := compute(ctx)
    go func() {
        ch <- result  // 如果调用方没读，这里永远阻塞
    }()
}

// 调用方
ch := make(chan Result)
go HandleRequest(ctx, ch)
time.Sleep(time.Second)
// ch 永远没人读，G 泄漏
```

**定位**：在 debug=1 输出中搜索 `"chan receive"` 或 `"chan send"` 的数量异常。

**修复**：

```go
// ✅ 修复 1：使用 select + timeout，确保不会永久阻塞
func HandleRequest(ctx context.Context, ch chan Result) {
    result := compute(ctx)
    select {
    case ch <- result:
    case <-ctx.Done():
        log.Printf("请求已取消，丢弃结果")
    }
}

// ✅ 修复 2：调用方用 select 读取，超时则放弃
select {
case r := <-ch:
    return r
case <-time.After(3 * time.Second):
    return nil  // 超时不等待
}
```

#### 根因 2：sync.Mutex 未 unlock（次常见）

**泄漏代码**：

```go
// ❌ 错误：提前 return 导致 mutex 未释放
func UpdateCache(key string, value []byte) error {
    mu.Lock()
    defer mu.Unlock()  // ✅ defer 可以避免这个问题

    data, err := fetchFromDB(key)
    if err != nil {
        return err  // ❌ 没有 defer 时，这里直接 return，锁未释放
    }

    cache[key] = data
    return nil
}
```

**定位**：在 debug=1 输出中搜索 `semacquire` 或 `mutex` 阻塞数量。

**修复**：始终使用 `defer mu.Unlock()`。

#### 根因 3：goroutine 中 for 循环错误（最隐蔽）

**泄漏代码**：

```go
// ❌ 错误：for 循环变量被闭包捕获，永远是最后一个值
func StartWorkers(tasks []Task) {
    for _, task := range tasks {
        go func() {
            process(task)  // 所有 goroutine 处理同一个 task
        }()
    }
}

// ✅ 修复：传入循环变量
for _, task := range tasks {
    task := task  // 创建新变量
    go func() {
        process(task)
    }()
}

// ✅ 现代 Go（1.22+）：循环变量语义变更，闭包直接捕获正确值
for _, task := range tasks {
    go func(t Task) {
        process(t)
    }(task)
}
```

---

### 第五步：修复验证

**压测 + 监控验证**：

```bash
# 1. 用 hey/wrk 压测接口
hey -n 10000 -c 200 http://localhost:8080/api/endpoint

# 2. 同时监控 goroutine 数量
# 正常：压测结束后 goroutine 应回落到基线 ±10%
# 泄漏：压测结束后 goroutine 持续在高位

watch -n 2 'curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | grep -c "^goroutine"'
```

**线上灰度验证**：

```go
// 灰度放量：先 5% 流量验证
if rand.Float64() < 0.05 {
    // 走修复后的逻辑
} else {
    // 走原有逻辑
}
```

---

## 生产级预防方案

### 1. 监控告警体系

```go
// 在监控中埋点，定期上报 goroutine 数量
func reportGoroutineCount() {
    n := runtime.NumGoroutine()
    metrics.GoroutineCount.WithLabelValues(instanceID).Set(float64(n))
}

// Prometheus 告警：基于 P99 历史基线
// 如果当前 P99 > 历史 P99 * 1.5，告警
```

### 2. 接入链路追踪

```go
// 为每个 goroutine 绑定 traceID，便于泄漏时定位请求
ctx, span := tracer.Start(ctx, "worker")
defer span.End()

go func() {
    doWork(ctx)  // ctx 包含 traceID，出问题可追溯
}()
```

### 3. 并发控制

```go
// 限制最大并发数，防止失控
var (
    maxWorkers = flag.Int("workers", 100, "max concurrent workers")
    workerSem  = make(chan struct{}, *maxWorkers)
)

func processRequest(req Request) {
    workerSem <- struct{}{}       // 获取信号量
    defer func() { <-workerSem }()

    // ... 处理逻辑
}
```

---

## 高频追问

**Q：pprof 采集会暂停服务吗？**
A：pprof profile 本身是采样（非侵入式），但如果用 `?debug=1` 的文本模式，它会遍历所有 goroutine 栈，约 1-5ms 暂停。低延迟服务建议用二进制 profile 模式。

**Q：goroutine 泄漏和内存泄漏有什么区别？**
A：goroutine 泄漏一定伴随内存泄漏（goroutine 栈 + 堆上变量），但反过来不一定。内存泄漏可能来自全局 map、sync.Pool 不断增长。goroutine 泄漏的特征是 goroutine 数量持续上涨，内存泄漏特征是 RSS 持续上涨。

**Q：生产环境 goroutine 数量多少算正常？**
A：没有绝对值，要建立基线。普通 HTTP 服务 100-500 个，Worker Pool 模式受任务数影响，万人同时在线聊天服务可能达 10000+。**关注趋势比关注绝对值更重要**。

**Q：如何区分正常 goroutine 增长 vs 泄漏？**
A：流量涨 → goroutine 涨 → 流量降 → goroutine 回落 = 正常。流量平稳但 goroutine 持续上涨 = 泄漏。可以用 Prometheus 的 `rate(go_goroutines[5m])` 判断趋势。

---

## 延伸阅读

- [Go 官方 pprof 文档](https://github.com/golang/go/tree/master/src/runtime/pprof)
- [Go 1.26 Release Notes - Goroutine Leak Profile](https://go.dev/blog/go1.26)
- [Contrib 博客：Goroutine Leaks: The Top 5 Causes](https://www.contrib.sh/blog/goroutine-leaks)
- [深度解析 Go GC 与 Goroutine 调度](https://www.youtube.com/watch?v=q1oJ4Re9z2U)
