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
io.Copy(io.Discard, resp.Body)  // 消费 body 避免连接池污染（Go 1.16+ 用 io.Discard）

# 2. 数据库连接未 Close
rows, err := db.Query("SELECT ...")
defer rows.Close()  // 必须！否则对应 socket FD 泄漏，积累后触发 EMFILE
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
io.Copy(io.Discard, resp.Body) // 必须消费 body（Go 1.16+ 用 io.Discard）
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

---

## 8. CPU Throttling 实战排障：P99 飙升但 CPU 使用率不高的场景（2026 高频）

> 考察频率：★★★★★  关键词：nr_throttled、throttled_usec、pprof schedule、Guaranteed QoS

### 8.1 典型症状

```
P99 延迟从 15ms 飙到 200ms+
CPU 使用率看起来正常（~70%）
prometheus 显示容器没超 memory limit
go tool pprof /profile → 看不出明显热点，所有函数都在 "running" 状态
```

### 8.2 快速定位三板斧

```bash
# 第一步：检查 cgroup 节流数据
cat /sys/fs/cgroup/cpu.stat
# nr_throttled 164        ← 非零！被节流过
# throttled_usec 15383840 ← 总共节流了 ~15 秒

# 第二步：对比 GOMAXPROCS 和容器限制
# Go 中查看当前 GOMAXPROCS
go version  # Go 版本（是否 >= 1.25）
grep GOMAXPROCS /proc/self/environ 2>/dev/null || echo "未显式设置"

# 如果是 Go < 1.25 且未设置，那默认就是物理核数
# 假设节点 64 核，容器 limit = 2 核 → 64 个 P 抢 2 核 → 严重 Throttling

# 第三步：在 kubectl exec 中看 K8s 事件
kubectl get events --field-selector reason=Throttled -n namespace
# 或查看 metrics-server
kubectl top pods -n namespace | grep go-service
```

### 8.3 pprof 中的特殊信号

```go
// cpu profile 中出现以下特征，基本确认是 Throttling 而非代码问题：
// 1. runtime.schedule 栈占大量时间
// 2. goroutine 状态显示为 "running" 但实际吞吐量极低
// 3. GC 相关栈占比异常（后台标记线程互相抢占）
// 4. netpoll 栈占比低（网络本身不是瓶颈）

// 验证命令
curl http://localhost:6060/debug/pprof/profile?seconds=60 > cpuprof.prof
go tool pprof -text cpuprof.prof | head -20
# 如果 schedule 前 3 名 → Throttling 嫌疑大
```

### 8.4 Go 1.25 vs 旧版本的根因差异

| 维度 | Go < 1.25（未设 GOMAXPROCS）| Go 1.25+（默认行为）|
|------|------------------------------|---------------------||
| 启动时 GOMAXPROCS | 物理核数（如 64）| min(物理核数, available, ceil(cgroup_limit)) |
| P 数量 | 过多（远超配额）| 接近配额 |
| GC worker 数量 | GOMAXPROCS × 25%（可能 16 个）| 按比例减少 |
| Throttling 概率 | 极高（尤其 limit < 4 核时）| 大幅降低但仍存在 |
| 动态调整 | ❌ | ✅ updatemaxprocs（适应 IPVS）|

### 8.5 面试话术

> 「排查 CPU Throttling 我一般先看 cgroup 的 cpu.stat，`nr_throttled` 非零就说明有节流。然后在 pprof 里看 `runtime.schedule` 是否占大头——如果是，说明线程在等调度而非做计算。根本解法是设 requests = limits 拿 Guaranteed QoS，Go 1.25 已经自动感知 cgroup 设 GOMAXPROCS，不需要 automaxprocs 了。」

---

## 附录：Go 容器镜像安全与体积优化（2026 趋势）

> 考察频率：★★★☆☆  关键词：distroless、alpine/musl、多阶段构建、静态链接、攻击面

### A.1 为什么镜像大小和安全很重要

```dockerfile
# ❌ 坏做法：用 ubuntu/fedora 作为基础镜像（臃肿且有 shell）
FROM golang:1.25 AS builder
RUN go build -o app .
FROM ubuntu:24.04          # ~80MB，有 apt/dpkg/bash
COPY --from=builder /app /usr/local/bin/app
ENTRYPOINT ["/usr/local/bin/app"]

# ✅ 推荐做法：Dockerfile 多阶段构建
FROM golang:1.25-bookworm AS builder
ARG TARGETOS TARGETARCH
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build -ldflags '-w -s' -o /app .

# Distroless 镜像：无 shell、无包管理器、无多余库（~20MB）
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app /app
USER 65534:65534  # non-root user
ENTRYPOINT ["/app"]
# ⚠️ distroless/static 没有 libc → 必须 CGO_ENABLED=0
```

### A.2 Alpine (musl) vs Debian (glibc) vs Distroless 对比

| 维度 | Alpine (musl) | Debian Slim (glibc) | Distroless static |
|------|--------------|--------------------|--------------------||
| 镜像大小 | ~7MB | ~70MB | ~20MB |
| 安全性 | 好（最小攻击面）| 中（有包管理器）| 最好（无 shell/工具） |
| CGO 支持 | ⚠️ musl libc 兼容性有限 | ✅ 完整支持 | ❌ 需静态链接 |
| TLS/crypto | ⚠️ BoringSSL，部分 edge case | ✅ OpenSSL | ✅ boringssl (static) |
| NSS/DNS | ⚠️ musl DNS 解析偶有差异 | ✅ glibc 标准兼容 | ✅ 通过 tdnf/boringssl |
| Go 兼容性 | ✅ CGO_ENABLED=0 完美 | ✅ CGO_ENABLED=1 正常 | ✅ CGO_ENABLED=0 |
| 适用场景 | 小内存边缘服务 | 需要 CGO 的场景 | 生产环境首选 |

**关键细节：**
- **musl 的性能影响**：纯 Go（CGO_ENABLED=0）下 musl vs glibc 性能差异极小；但如果用了 CGO 调用 C 库，musl 可能在 DNS 解析、多线程场景下有微妙差异
- **distroless 的选择**：`distroless/static` 无运行时依赖（需静态编译），`distroless/base` 包含 glibc 但无 shell（适合 CGO 场景）

### A.3 推荐的 Dockerfile 模板

```dockerfile
# === Build Stage ===
FROM golang:1.25-bookworm AS builder
WORKDIR /src

# Cache dependencies layer
COPY go.mod go.sum ./
RUN --mount=type=cache=target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    go mod download

# Build with optimizations
COPY . .
ARG VERSION="dev"
ARG COMMIT="unknown"
ARG BUILD_TIME="unknown"
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags "-s -w -X main.version=${VERSION} -X main.commit=${COMMIT} -X main.buildTime=${BUILD_TIME}" \
    -trimpath -o /app .

# === Runtime Stage ===
FROM gcr.io/distroless/static-debian12 AS runtime
COPY --from=builder /app /app
USER 65534:65534  # non-root user
ENTRYPOINT ["/app"]
```

### A.4 安全检查清单

```yaml
# K8s Pod Security Standards (PSS) 配置
apiVersion: v1
kind: Pod
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted  # 最严格级别
spec:
  containers:
  - image: your-app:v1
    securityContext:
      runAsNonRoot: true        # 强制非 root 运行
      readOnlyRootFilesystem: true  # 只读根文件系统
      allowPrivilegeEscalation: false  # 禁止提权
      capabilities:
        drop:
          - ALL                 # 放弃所有能力
```

---

## 延伸阅读

- [Netflix Linux Performance](https://www.brendangregg.com/linuxperf.html)
- [Go pprof 官方文档](https://github.com/golang/go/wiki/Pprof)
- [Golang otel + Prometheus 监控实战](https://povilasv.me/go-monitoring/)
