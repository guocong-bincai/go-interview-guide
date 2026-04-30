[🏠 首页](../../../README.md) · [📦 Linux / 操作系统基础](../README.md)

---

# Linux 线上排障基础

> 考察频率：★★★★☆  优先级：P0
> 关键词：CPU 高、内存满、磁盘 IO 高、FD 泄漏、TIME_WAIT、goroutine leak、pprof

---

## 1. 排障总览：Linux 三大资源

线上问题归根结底是 **CPU / 内存 / I/O** 三种资源的异常：

```
线上告警 → 定位资源类型
     │
     ├── CPU 高 → top → 进程 → 线程 → 代码热点
     │
     ├── 内存 高 → free / top → OOM → 泄漏分析
     │
     ├── I/O 高 → iostat → 磁盘/网络 → 文件/系统调用
     │
     └── 磁盘满 → df / du → 大文件 → 清理或扩容
```

**排障黄金法则：先定性（是哪种资源），再定量（定位到哪个进程/指标）**

---

## 2. CPU 排查：从 load 高到代码热点

### 2.1 快速定位：load 和 CPU 指标

```bash
# 查看系统负载（1/5/15 分钟）
uptime
# 10:00:01 up 100 days,  1 user,  load average: 5.23, 4.15, 3.21
# load average = CPU 运行 + 等待进程数（不是 CPU 使用率！）

# 判断：load > CPU 核数 → 存在 CPU 排队（负载过高）
# 32 核机器，load = 5 → 轻松
# 32 核机器，load = 40 → 严重排队

# 查看 CPU 使用详情
top -H   # -H 显示线程
top -p $(pidof your-app)  # 只看特定进程

# CPU 使用率各列含义：
# %us（user）：用户态 CPU
# %sy（system）：内核态 CPU
# %wa（iowait）：CPU 等待 I/O
# %hi（hardware interrupt）：硬件中断
# %si（soft interrupt）：软中断（网络、调度）
# %id（idle）：空闲
```

### 2.2 定位热点函数：perf 和 pprof

```bash
# perf：采样 CPU 栈（需要调试符号）
perf record -F 99 -p $(pidof your-app) -g -- sleep 30
perf report

# Go pprof（在线）
# 1. 开启 pprof 端口
import _ "net/http/pprof"
go http.ListenAndServe(":6060", nil)

// 2. 采样 30 秒 CPU
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

// 3. 查看火焰图
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/profile
# 打开 http://localhost:8080/ui/flamegraph
```

**perf vs pprof 对比：**

| 维度 | perf | pprof |
|------|------|-------|
| 采样范围 | 全系统（包括内核）| 仅 Go 进程 |
| 内核函数 | ✅ 可见 | ❌ 不可见 |
| Go runtime | 需要 debuginfo | ✅ 原生支持 |
| 火焰图 | 需额外工具 | `go tool pprof --http` |

### 2.3 常见 CPU 热点场景

**① GC 导致的 CPU spike**
```go
// GOGC 默认 100%，GC 时 STW + 并发 GC 标记
// P99 停顿可能 > 100ms（大量 live objects 时）

// 诊断：pprof 看 GC pause
go tool pprof http://localhost:6060/debug/pprof/gc?debug=1

// 解决：
// 1. 降低 GOGC（减少 GC 频率）
// GOGC=50  → 堆增长 50% 就触发 GC（更频繁但每次更轻）
// GOGC=200 → 堆增长 200% 才触发 GC（更少 GC 但每次更重）

// 2. 使用 Go 1.20+ 的 GOGC=auto（根据 live heap 自适应）
// 3. 减少对象分配（对象池、减少 string concat）
```

**② 正则表达式灾难性回溯**
```go
// ⚠️ 灾难性回溯的正则（ReDoS）
// (a+)+b  对 "aaaaaaaaaab" 匹配会指数级回溯

// CPU spike 特征：正则处理函数占 80%+ CPU
// 解决：用 re2（Go 内置）或确定性正则
import "regexp"
re := regexp.MustCompile(`^[a-z]+$`)  // re2 自动机，无回溯
```

---

## 3. 内存排查：泄漏 vs 缓存 vs GC

### 3.1 内存指标解读

```bash
# 查看内存使用
free -h
#               total        used        free      shared  buff/cache   available
# Mem:          125Gi       45Gi        3Gi        200Mi    77Gi         78Gi
# Swap:          2Gi         0B          2Gi

# 关键：available = 真正可分配内存（不包括 Page Cache 可回收部分）
# used（45Gi）= actual RSS = 程序真实占用 + Page Cache 可回收部分

# 查看进程内存
ps aux --sort=-rss | head -20
# RSS = Resident Set Size = 实际物理内存占用
# VSZ = Virtual Size = 虚拟地址空间大小（包含未映射的）

# 查看 /proc/<pid>/smaps 详细内存分布
cat /proc/$(pidof your-app)/smaps_rollup | head -30
```

### 3.2 Go 内存问题排查

```go
// 1. 查看 Go 堆内存
// curl http://localhost:6060/debug/pprof/heap?debug=1
// 输出中的 Alloc = 当前 live 对象大小
// TotalAlloc = 累计分配（不含 GC）

// 2. goroutine 泄漏排查
curl http://localhost:6060/debug/pprof/goroutine?debug=1 | grep -c "goroutine"
// 正常：几十到几百 goroutine
// 泄漏：goroutine 数随时间线性增长

// 3. goroutine leak 检测（单元测试）
import "go.uber.org/goleak"
func TestLeak(t *testing.T) {
    defer goleak.VerifyNone(t)
    // 测试代码
}
```

**典型内存泄漏模式：**

```go
// 泄漏1：goroutine 泄漏（channel 无人接收）
func leak() {
    ch := make(chan int)
    go func() {
        // 永远阻塞在这里
        result := computeHeavy()
        ch <- result
    }()
    // ch 永远没人读 → goroutine 永久阻塞 → 栈+堆对象泄漏
}

// 泄漏2：map 持续写入
var globalMap = make(map[string]interface{})
func leak() {
    for {
        key := generateKey()
        globalMap[key] = heavyData()  // map 只增不减 → 内存泄漏
    }
}

// 泄漏3：闭包捕获大对象
func leak() {
    largeData := make([]byte, 10*1024*1024)  // 10MB
    go func() {
        // largeData 被闭包捕获，goroutine 生命周期内无法 GC
        use(largeData)
    }()
    // largeData 在 goroutine 退出前不会被释放
}
```

### 3.3 内存告警处理流程

```
内存告警（>80%）
     │
     ├── free -h → 确认是容器限制还是真实泄漏
     │
     ├── docker stats / kube top pod → 容器/ Pod 内存
     │
     ├── go tool pprof heap → 定位谁在分配
     │
     ├── grep "goroutine" pprof → goroutine 泄漏？
     │
     └── dmesg | grep OOM → 是否被 OOMKill？
```

---

## 4. 磁盘 I/O 排查

### 4.1 I/O 指标

```bash
# iostat（需要 sysstat 包）
iostat -xz 1
# %util = 设备利用率（100% = 饱和）
# r/s, w/s = 每秒读写次数
# avgrq-sz = 平均请求大小（扇区数）
# await = 平均 I/O 等待时间（ms）

# 常见问题判断：
# %util > 80% + await > 20ms → 磁盘 I/O 瓶颈
# %util 低但 %iowait 高 → CPU 在等 I/O（可能是网络 I/O）
# read I/O 高 → 大量顺序读（mmap 大文件？）
# write I/O 高 → 大量日志/数据写入

# 查看哪个进程在 I/O
iotop -o  # 只显示活跃 I/O 的进程
```

### 4.2 文件描述符（FD）泄漏

```bash
# 查看进程 FD 数量
ls -la /proc/$(pidof your-app)/fd | wc -l

# 系统级 FD 限制
cat /proc/sys/fs/file-max          # 系统最大 FD 数
ulimit -n                           # 当前 shell 的 FD 上限

# 进程 FD 限制（limits.conf）
# your-app soft nofile 65535
# your-app hard nofile 65535

# 典型泄漏场景：未关闭的文件/连接
# 1. HTTP 客户端未关闭 response body
resp, _ := http.Get("http://example.com")
defer resp.Body.Close()  // 必须！否则 FD 泄漏
io.Copy(ioutil.Discard, resp.Body)  // 消费 body 避免连接池污染

# 2. 数据库连接未 Close
rows, err := db.Query("SELECT ...")
defer rows.Close()  // 必须！
```

---

## 5. 网络排查：TIME_WAIT 与连接问题

### 5.1 网络相关指标

```bash
# 查看网络连接状态
ss -s
# ESTAB = 活跃连接
# TIME-WAIT = 主动关闭方等待 2MSL（60秒）
# CLOSE-WAIT = 被动关闭方等待应用程序关闭

# 常见问题：TIME_WAIT 堆积（端口耗尽）
# 原因：大量短连接，连接关闭后进入 TIME_WAIT（持续 60 秒）
# 症状：新建连接失败（本地端口 65535 耗尽）

# 解决：
# 1. 服务端：tcp_tw_reuse = 1（允许重用 TIME_WAIT 连接）
sysctl -w net.ipv4.tcp_tw_reuse=1

# 2. 客户端：使用 HTTP 长连接 / 连接池
# 3. 调整短连接超时
sysctl -w net.ipv4.netfilter/ip_conntrack_tcp_timeout_time_wait=15

# 查看端口使用情况
ss -ant | awk '{print $1}' | sort | uniq -c | sort -rn
# TIME_WAIT 数量过多 → 检查是否有过多短连接
```

### 5.2 Goroutine 连接泄漏

```go
// HTTP 客户端未读取 body 导致连接池阻塞
resp, _ := client.Get("http://example.com")
// ⚠️ 如果不读 body，连接不会被归还连接池
// 高并发时会耗尽 MaxIdleConns
io.Copy(ioutil.Discard, resp.Body) // 必须消费 body
resp.Body.Close()
```

---

## 6. Go 服务的排障工具箱

```bash
# ① pprof（性能剖析）
go tool pprof http://localhost:6060/debug/pprof/
# /profile     CPU 采样
# /heap        堆内存
# /goroutine   goroutine 数量和堆栈
# /threadcreate 线程创建
# /block        阻塞操作（Mutex、channel）
# /mutex       Mutex 争用

# ② expvar（应用指标）
import _ "expvar"
// 暴露 /debug/vars，JSON 格式应用指标

# ③ trace（Go 1.11+）：查看调度延迟、GC 暂停、系统调用
go tool trace http://localhost:6060/debug/trace?seconds=5

# ④ 运行时统计
curl http://localhost:6060/debug/stats?debug=1
# 显示：goroutine 数量、内存分配、GC 次数、GC pause

# ⑤ 死亡检测（leak goroutine 检测）
go test -run=XXX -test.leak=false -leakck=false  # 跳过 leak 检测
# 生产环境用 goleak
```

---

## 7. 排障 SOP 总结

```
告警：CPU/内存/网络告警
     │
     ▼
① 快速确认：uptime / free / ss -s
     │
     ▼
② 定位进程：top / ps / docker stats
     │
     ▼
③ 深入分析：pprof / perf / strace
     │
     ▼
④ 定位根因：代码？配置？容量？
     │
     ▼
⑤ 止血：重启？限流？扩容？GC调参？
     │
     ▼
⑥ 复盘：加监控、加告警、加防护
```

---

## 延伸阅读

- [Netflix Linux Performance](https://www.brendangregg.com/linuxperf.html)
- [Go pprof 官方文档](https://github.com/golang/go/wiki/Pprof)
- [Golang otel + Prometheus 监控实战](https://povilasv.me/go-monitoring/)
