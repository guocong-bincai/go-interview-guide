[🏠 首页](../../../README.md) · [📦 Linux / 操作系统基础](../README.md)

---

# cgroup、namespace 与容器资源隔离

> 考察频率：★★★★☆  优先级：P0
> 关键词：cgroup v2、namespace、容器资源限制、OOMKilled、GOMAXPROCS、K8s resource

---

## 1. 容器技术的本质：Linux 命名空间

### 1.1 namespace 是什么

Linux **命名空间（namespace）** 是内核对全局资源进行**视图隔离**的机制。
同一个全局资源，在不同 namespace 中看到不同的值：

```
宿主机                              容器 namespace
  │                                      │
  │  hostname = "prod-server"           │  hostname = "web-01"
  │  PID 1 进程 = systemd                │  PID 1 进程 = nginx
  │  网络 = 物理网卡 eth0                │  网络 = 虚拟网卡 eth0（隔离）
  │  用户 = root(UID 0)                 │  用户 = root(UID 0，但映射到宿主机 UID 100000）
  │  文件系统 = 完整根目录 /            │  文件系统 = overlay 层（只读镜像 + 可写层）
```

**Linux 主要 namespace 类型：**

| namespace | 隔离内容 | 关键标志 |
|-----------|---------|---------|
| **PID** | 进程号 | 不同容器内可都有 PID 1 |
| **NET** | 网络栈（IP、端口、路由）| 容器有独立网卡和 IP |
| **MNT** | 文件系统挂载点 | 容器有独立根文件系统 |
| **UTS** | hostname / domainname | 容器可独立设置主机名 |
| **USER** | 用户/组 UID/GID 映射 | 容器 root 映射为非 root 宿主用户 |
| **IPC** | 共享内存、信号量、消息队列 | 容器 IPC 与宿主机隔离 |
| **TIME** | 进程时钟（Linux 5.6+）| 容器可独立设置启动时间 |

### 1.2 namespace 查看与操作

```bash
# 查看进程的 namespace
ls -la /proc/$$/ns/
# lrwxrwxrwx 1 root root 0 Apr 30 10:00 pid -> pid:[4026531834]
# lrwxrwxrwx 1 root root 0 Apr 30 10:00 net -> net:[4026531957]
# lrwxrwxrwx 1 root root 0 Apr 30 10:00 mnt -> mnt:[4026531838]

# 进入容器的 namespace（需要特权）
nsenter --target <pid> --pid --net --mount

# 查看容器进程
docker ps
docker inspect <container> | grep -i pid

# 宿主机看到的 PID vs 容器内 PID
docker run --rm busybox cat /proc/1/status | grep Pid
# 容器内 PID 1 = 宿主机 PID 12345（映射关系由 namespace 管理）
```

---

## 2. cgroup：容器的资源控制

### 2.1 cgroup 是什么

**cgroup（Control Group）** 是内核对进程组进行**资源配额控制**的机制。
namespace 负责隔离，cgroup 负责限制。

```
cgroup v2 层级结构：
/sys/fs/cgroup/
├── memory.max          # 内存上限
├── cpu.max              # CPU 配额
├── io.max               # I/O 带宽上限
├── pids.max             # 进程数上限
└── memory.high          # 内存软上限（超过后降低优先级）
```

### 2.2 cgroup v1 vs v2

| 特性 | cgroup v1 | cgroup v2 |
|------|----------|----------|
| 层级 | 多棵树（每种资源独立层级）| **单棵树（统一层级）** |
| 容器支持 | 需要 mount 多个子系统 | 一个 unified hierarchy |
| 内存保护 | 资源竞争问题 | 更清晰的资源隔离 |
| 推广 | Docker 默认用 v1 | K8s 1.25+ 默认用 v2 |

```bash
# 查看 cgroup 版本
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime)

# 查看 cgroup 限制
cat /sys/fs/cgroup/memory.max      # 内存上限（字节）
cat /sys/fs/cgroup/cpu.max         # CPU 配额（period + quota）
cat /sys/fs/cgroup/pids.max        # 最大进程数
```

### 2.3 关键资源限制参数

**① 内存限制（memory.max / memory.limit_in_bytes）**

```bash
# Docker 设置内存限制
docker run -m 512m nginx

# K8s 设置内存限制
# resources:
#   limits:
#     memory: "512Mi"
#   requests:
#     memory: "256Mi"
```

**② CPU 限制（cpu.max = quota / period）**

```bash
# Docker CPU 限制：0.5 核
docker run --cpus=0.5 nginx

# 解释：cpu.max = 50000 / 100000
# period=100000us（100ms），quota=50000us（50ms）
# 即：每 100ms 周期内，最多使用 50ms CPU（0.5 核）

# K8s CPU 限制
# resources:
#   limits:
#     cpu: "500m"   # 0.5 CPU
#   requests:
#     cpu: "250m"
```

**③ I/O 限制（io.max）**

```bash
# 设置块设备最大带宽
echo "253:0 104857600" > /sys/fs/cgroup/io.max
# 253:0 = /dev/vda 的 major:minor
# 104857600 = 100MB/s
```

---

## 3. OOMKilled：内存超限的代价

### 3.1 容器内存超限 → OOM Killer

```
容器内进程申请内存
       │
       ▼
  达到 memory.limit_in_bytes
       │
       ▼
  触发 cgroup OOM（内存耗尽）
       │
       ▼
  内核选择最"该死"的进程杀掉
  （通常是占用最多内存的那个）
       │
       ▼
  进程收到 SIGKILL（OOMKilled）
       │
       ▼
  docker ps -a → STATUS = OOMKilled
```

### 3.2 Go 服务遭遇 OOMKilled 的典型场景

**场景 1：GOGC 默认 100%，突发流量时堆增长 2 倍**
```go
// 初始：live heap = 100MB，GOGC=100 → 触发 GC 的阈值 = 200MB
// 突发流量：live heap = 190MB
// GC 还没来得及运行，流量继续 → 瞬间超过 200MB → OOM

// 解决：降低 GOGC 或设置更保守的内存限制
```

**场景 2：goroutine 泄漏 + 大数据流**
```go
// goroutine 持续增长（泄漏）
// 每个 goroutine 栈 2KB
// 100万 goroutine × 2KB = 2GB（只是栈）
// 加上业务堆数据 → 超过容器内存限制 → OOM
```

**场景 3：mmap 大文件到容器内存**
```go
// mmap 了 5GB 日志文件到内存
// mmap 的文件也计入 RSS
// 加上 Go 堆 → 超过限制 → OOM
```

### 3.3 排查 OOMKilled

```bash
# 1. 查看容器退出原因
docker inspect <container> | grep -i oom
# "OOMKilled": true

# 2. 查看 dmesg（内核日志）
dmesg | tail -50 | grep -i "oom\|killed"
# [Wed Apr 30 10:00:01 2026] Memory cgroup out of memory: Killed process 12345 (nginx)

# 3. 查看 Go 服务的内存使用
go tool pprof http://localhost:6060/debug/pprof/heap?debug=1

# 4. 对比容器限制和实际使用
docker stats <container>
# MEM USAGE / LIMIT = 450MB / 512MB → 接近上限
```

---

## 4. GOMAXPROCS 与容器 CPU 配额

### 4.1 问题：Go 默认按 CPU 核数设置 GOMAXPROCS

**Go 的 GOMAXPROCS 默认 = 宿主机 CPU 核数**

```go
// 如果在 64 核宿主机上运行容器，Go 会认为有 64 核
// 设置 GOMAXPROCS = 64

// 但容器 CPU 限制只有 0.5 核！
// 问题：Go 会创建大量 M（P 的数量 = 64）
// 但实际调度到的时间片只有 0.5 核 → 大量 M 空转，浪费资源
```

### 4.2 Go 1.25（2025年8月）前的解决方案

```go
// 方案1：手动设置 GOMAXPROCS = 容器 CPU 限制
func init() {
    // 读取 cgroup CPU quota
    if quota, err := readCgroupCPUQuota(); err == nil {
        if quota > 0 {
            numCPU := int(math.Ceil(quota))
            runtime.GOMAXPROCS(numCPU)
        }
    }
}

// 方案2：使用库自动检测
import _ "github.com/uber-go/automaxprocs"
// init() 自动检测并设置 GOMAXPROCS
```

### 4.3 Go 1.25 的改进：容器感知 GOMAXPROCS

Go 1.25 引入了**容器感知自动调整 GOMAXPROCS**：

```bash
# Go 1.25+ 自动读取 cgroup CPU 限制
# /sys/fs/cgroup/cpu.max = 50000 / 100000 → GOMAXPROCS 自动设为 1
# 不再需要 automaxprocs 库

# 环境变量控制
GOMAXPROCS=4 go run app.go  # 仍可手动覆盖
```

**Go 1.25+ 读取 cgroup 的顺序：**
```
1. 读取 /sys/fs/cgroup/cpu.max（v2）
2. 读取 /sys/fs/cgroup/cpu/cpu.cfs_quota_us + cpu.cfs_period_us（v1）
3. 两者都没有 → 使用 sched_getaffinity（物理 CPU 数）
```

### 4.4 生产配置建议

```yaml
# K8s deployment 配置（CPU limit = request 是最佳实践）
containers:
  - name: app
    resources:
      limits:
        cpu: "2000m"        # 上限 2 核
        memory: "2Gi"
      requests:
        cpu: "2000m"        #Guaranteed QoS：请求=上限
        memory: "1Gi"
---
# 为什么 requests = limits：
# 1. Guaranteed QoS：kubelet 优先调度到固定资源
# 2. 配合 GOMAXPROCS = CPU 核数 = 确定性行为
# 3. 避免"共享核"导致的调度抖动
```

---

## 5. namespace 与 cgroup 在 K8s 中的使用

### 5.1 K8s Pod 是如何利用 namespace 和 cgroup 的

```
K8s Pod 中的容器共享：
  ✅ 同一 NET namespace（Pod 内容器通过 localhost 通信）
  ✅ 同一 PID namespace（可以看到彼此的进程，容器的 PID 1 是各容器自己的）
  ❌ 各自的 MNT namespace（各自的文件系统根）
  ❌ 各自的 USER namespace（可选）
  
K8s 对 Pod 的资源限制：
  → 每个容器有独立的 cgroup 限制
  → Pod 整体可以设置 limitrange / resourcequota
```

### 5.2 K8s 资源限制的类型

```bash
# LimitRange：设置命名空间内所有容器的默认资源限制
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limit
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi

# ResourceQuota：限制命名空间总资源
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "100"
```

---

## 6. 高频追问

### Q1：容器内看到的 CPU 核数和宿主机一样，怎么知道容器实际配额？

```bash
# 方法1：读取 cgroup（Go 1.25+ 自动处理）
cat /sys/fs/cgroup/cpu.max
# 50000 100000 = 0.5 核

# 方法2：nproc（不准确，返回的是物理核数，不是容器配额）
nproc
# 64（物理核数，不是容器配额）

# 正确做法：使用 cgroup 的 cpu.max
```

### Q2：容器内存限制是 512MB，Go 堆能用到多少？

**保守估计：容器内存的 70~80%**

```
容器内存 512MB 分配：
  OS 内核：~50MB（cgroup 内核开销）
  Go 堆：~350MB（GOGC=100%）
  goroutine 栈：~50MB（25K goroutine × 2KB）
  其他：mmap、共享库、Page Cache 等
```

**建议：** Go 服务内存限制设置为 `requests.memory × 1.3~1.5`

### Q3：Pod 的 QoS 级别影响什么？

| QoS | 触发条件 | 特点 |
|-----|---------|------|
| **Guaranteed** | limits = requests（CPU + 内存）| 最优先调度，最不易被 OOMKill |
| **Burstable** | requests < limits | 允许突发，可超过 requests 但不超过 limits |
| **BestEffort** | 无 requests/limits | 最低优先级，资源紧张时最先被驱逐 |

---

## 延伸阅读

- [cgroup v2 官方文档](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [K8s 资源管理](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Go 1.25 GOMAXPROCS 自动检测](https://go.dev/doc/go1.25)
