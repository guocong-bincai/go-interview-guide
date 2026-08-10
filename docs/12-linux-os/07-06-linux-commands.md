[🏠 首页](../../../README.md) · [📦 Linux / 操作系统基础](../README.md)

---

# Linux 基础高频命令：线上排障必备工具链

> 考察频率：★★★★☆  优先级：P0
> 关键词：top / ps / netstat / ss / lsof / strace / tcpdump / perf

---

## 面试官考察意图

这道题区分度极高。
初级工程师只能背出命令，高级工程师能结合**具体排障场景**讲清楚：
- 每个命令的**核心指标**是什么（不是背参数）
- 多个命令如何**联合使用**定位根因
- 在 Go 服务场景下**特殊关注点**（goroutine、fd、内存模型）

---

## 核心答案（30 秒版）

| 场景 | 首选命令 | 核心看什么 |
|------|----------|-----------|
| CPU 高排查 | `top -Hp pid` | %CPU 按线程排序，找热点线程 |
| 内存泄漏 | `top` + `/proc/<pid>/smaps` | RES 持续增长，RSS vs heap |
| 网络连接 | `ss -tunap` | 连接状态分布，TIME_WAIT |
| 文件描述符 | `lsof -p pid` | FD 泄漏，fd 数量 |
| 系统调用 | `strace -cp pid` | 耗时分布，哪些 syscall |
| 网络抓包 | `tcpdump -i eth0 port 8080` | 延迟、丢包、包内容 |
| 性能剖析 | `perf top -p pid` | CPU 采样的函数热点 |

---

## 深度展开

### 1. top / ps：进程与 CPU

#### 1.1 top 关键指标解读

```bash
top - 10:00:00 up 100 days,  1 user,  load average: 5.23, 4.15, 3.21
Tasks: 200 total,   3 running, 197 sleeping,   0 stopped,   0 zombie
%Cpu(s): 45.2 us,  3.1 sy,  0.0 ni, 51.2 id,  0.0 wa,  0.3 hi,  0.2 si
KiB Mem: 32768 total,  8192 free,  20480 used,  4096 buff/cache
KiB Swap:  8192 total,  8192 free,     0 used, 20000 avail Mem
```

**Go 工程师重点关注：**

| 指标 | 含义 | Go 服务告警阈值 |
|------|------|---------------|
| `load average` | CPU 运行 + 等待进程数 | load > CPU 核数 → 排队 |
| `%Cpu(s): us` | 用户态 CPU（代码执行）| 持续 > 90% → CPU 热点 |
| `id` | 空闲 CPU | 接近 0 + load 高 → CPU 打满 |
| `wa` | I/O 等待 | 持续 > 20% → I/O 阻塞 |

#### 1.2 top 按线程看 CPU

```bash
# 找出 Go 服务中 CPU 占用最高的线程
top -Hp $(pidof your-app)

# 输出：
#   PID USER      PR  NI  %CPU %MEM    TIME+  COMMAND
# 12345 root      20   0  85.3  2.1   5:30.20 your-app   # 主进程
# 12346 root      20   0  12.1  0.1   0:45.30 your-app   # GC 工作线程
# 12347 root      20   0   0.3  0.0   0:01.20 runtime.gmparki
```

**关联 pprof：**
1. 用 `top -Hp` 找到高 CPU 线程 PID
2. 将 PID 转为十六进制：`printf '%x\n' 12345` → `3039`
3. pprof 中找 `runtime.gmparki` 相关堆栈（GC 工作线程 CPU 高）

#### 1.3 ps 命令

```bash
# 查看进程详细信息（启动命令完整）
ps -ef | grep your-app

# 查看进程内存映射（看 heap 是否持续增长）
ps -p pid -o pid,vsz,rss,pmem,command

# 查看进程树（看父子关系）
ps -ef --forest | grep your-app

# 查看线程数
ps -eLf | grep your-app | wc -l
```

**Go 服务告警：线程数异常**
```bash
# Go 服务线程数 = M（内核线程）数量，正常几千个 goroutine 共享少数 M
ps -eLf | grep your-app | awk '{print $3}' | sort | uniq -c | sort -rn | head -5
# 如果 thread 数远超预期 → 可能存在 goroutine 泄漏或 CGO 线程泄漏
```

---

### 2. ss / netstat：网络连接

#### 2.1 ss 是 netstat 的现代替代

```bash
# 查看所有 TCP 连接（状态统计）
ss -s

# 查看监听端口的进程
ss -tlnp

# 查看已建立连接（包含 PID）
ss -tnp

# 查看特定状态的连接（如 TIME_WAIT）
ss -tn state time-wait

# 查看所有 UDP 端口
ss -ulnp

# 查看 Socket 内存使用
ss -m
```

**ss 核心状态快速过滤：**

| 想看什么 | 命令 |
|---------|------|
| 所有监听端口 | `ss -tlnp` |
| 建立了多少连接 | `ss -tn state established` |
| TIME_WAIT 数量 | `ss -tn state time-wait | wc -l` |
| 每个 IP 的连接数 | `ss -tn | awk '{print $4}' | cut -d: -f1 | sort | uniq -c | sort -rn` |
| 慢连接（半开）| `ss -tn state syn-sent` |

#### 2.2 Go 服务 TIME_WAIT 优化实战

```bash
# 场景：Go HTTP 服务对外大量短连接，出现 TIME_WAIT 堆积
$ ss -tn state time-wait | wc -l
5230

# 原因：HTTP client 默认不使用 HTTP keep-alive，每次请求后关闭连接
# 解决：
# 1. 客户端开启 keep-alive（Go http.Client 默认开启）
# 2. 服务端开启 tcp_tw_reuse：sysctl -w net.ipv4.tcp_tw_reuse=1
# 3. 高并发场景使用连接池复用连接

# 监控 TIME_WAIT 连接数的脚本
watch -n 5 'ss -tn state time-wait | wc -l'
```

---

### 3. lsof：文件描述符与资源泄漏

```bash
# 查看某进程打开的所有 FD
lsof -p pid

# 查看某进程打开的网络连接
lsof -i -a -p pid

# 查看某个端口被哪个进程占用
lsof -i :8080

# 查看网络连接对应的 FD（看 socket fd 数量）
lsof -i -a -p pid | grep -c sock
```

**Go FD 泄漏常见原因：**
1. HTTP response.Body 未关闭（最常见）
2. 数据库连接未释放
3. WebSocket 长连接未主动关闭
4. 日志文件打开后不断增长但未刷新

---

### 4. strace：系统调用追踪

```bash
# 统计每种系统调用的次数和耗时
strace -c -p pid

# 实时跟踪系统调用
strace -p pid

# 跟踪特定系统调用
strace -e write,read -p pid
```

**Go 中的限制：** Go 程序通过 runtime 封装系统调用，strace 对 Go 的效果有限。最佳实践是 **strace + pprof 联合使用**：
- strace 找系统层面的异常
- pprof 找 Go 代码层面的热点

---

### 5. tcpdump：网络抓包

```bash
# 抓本机 8080 端口的 TCP 包（显示 ASCII 内容）
tcpdump -i lo -A port 8080

# 保存为 pcap 文件（用 Wireshark 分析）
tcpdump -i eth0 port 8080 -w /tmp/capture.pcap

# 只抓 SYN 包（排查连接建立问题）
tcpdump -i eth0 'tcp[tcpflags] = tcp-syn'

# 抓 DNS 查询
tcpdump -i eth0 port 53
```

---

### 6. 命令联合使用：排障 SOP

**场景：Go 服务 P99 延迟突然升高**

```bash
# 第一步：定性
top
ss -tn state established | wc -l

# 第二步：定位进程/线程
top -Hp $(pidof your-app)

# 第三步：定位代码热点
printf '%x\n' <高CPU的PID>
# pprof 中找对应十六进制 PID

# 第四步：如果是 I/O 问题
strace -c -p <pid>
tcpdump -i eth0 port 8080 -w /tmp/out.pcap

# 第五步：如果是内存问题
ps -p <pid> -o pid,vsz,rss,pmem
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof -text heap.prof
```

---

## 高频追问

**Q：top 里 CPU 100% 但 pprof 显示没有热点代码，可能是什么原因？**
A：① GC 密集型：Go GC 工作线程（`runtime.gcBgMarkWorker`）消耗 CPU。② 系统态高：如果 `%Cpu(s): sy` 高，用 `strace -c` 看哪些 syscall 耗时。

**Q：lsof 显示 fd 数量正常，但 Socket 连接数很多，正常吗？**
A：正常。Socket 也是 FD，Go HTTP 服务每个请求对应一个 socket fd，长连接下连接数不会突增。要关注的是 TIME_WAIT 堆积。

---

---

## 附录：Go 容器镜像构建与优化命令（2026 趋势）

> 考察频率：★★★☆☆  关键词：多阶段构建、distroless、musl/glibc、CGO_ENABLED、安全扫描

### B.1 Dockerfile 常用命令速查

```bash
# === 构建阶段（编译）===
FROM golang:1.25-bookworm AS builder    # 基础镜像选择
WORKDIR /app                             # 工作目录
COPY go.mod go.sum ./                    # 分层缓存：先复制依赖文件
RUN --mount=type=cache,target=/go/pkg/mod go mod download  # BuildKit 缓存
COPY . .                                 # 复制源代码
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags '-w -s' -trimpath -o /app .

# === 运行阶段（最小化）===
FROM gcr.io/distroless/static-debian12  # 无 shell、无包管理器
COPY --from=builder /app /app
USER 65534:65534                         # 非 root 运行
ENTRYPOINT ["/app"]
```

### B.2 镜像大小对比

```bash
# 各基础镜像的实际大小（Go 服务场景）
docker images | grep -E 'golang|debian|alpine|distroless'

# 典型结果：
# golang:1.25-bookworm        ~1.1GB  （编译用，不应作为运行时镜像）
# debian:12-slim              ~70MB   （有 apt + glibc）
# alpine:latest               ~7MB    （musl libc，无包管理时）
# gcr.io/distroless/static    ~20MB   （boringssl，无 shell）
# gcr.io/distroless/base      ~80MB   （含 glibc，可跑 CGO）

# 计算压缩比
# Ubuntu base → Distroless = 缩减约 95%
# Alpine base → Distroless = 缩减约 70%
```

### B.3 musl vs glibc 兼容性排查命令

```bash
# 检查二进制使用的动态库
$ ldd /app
# musl 环境下的 Go 静态编译（CGO_ENABLED=0）：
#     not a dynamic executable
# glibc 环境下的 Go 静态编译：
#     not a dynamic executable
# 但如果有 CGO 链接了 C 库：
#     libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6
#     libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6

# 在 Alpine (musl) 容器中测试：
docker run --rm -v $(pwd):/src alpine /bin/sh -c '
    apk add --no-cache musl-static
    # 验证 musl 下 DNS 行为
    nslookup google.com
'

# glibc vs musl 关键差异点：
# 1. musl 的 getaddrinfo() 不使用 nsswitch.conf，只用 /etc/resolv.conf
# 2. musl 的多线程分配器 (ptmalloc) 在高并发下有不同表现
# 3. musl 不实现 posix_memalign() 在某些旧版本中
# 结论：纯 Go (CGO_ENABLED=0) 两者无差别；CGO 场景 glibc 更稳
```

### B.5 TCP 调优参数与 netstat 分析

```bash
# 查看 TCP 调优参数（生产环境常用）
sysctl net.ipv4.tcp_tw_reuse net.ipv4.tcp_fin_timeout \
  net.ipv4.tcp_max_syn_backlog net.core.somaxconn \
  net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_intvl

# Go 服务推荐的 sysctl 配置（写入 /etc/sysctl.conf）
net.ipv4.tcp_tw_reuse = 1         # 允许重用 TIME_WAIT socket
net.ipv4.tcp_fin_timeout = 30     # FIN-WAIT-2 超时从 60s 降到 30s
net.ipv4.tcp_max_syn_backlog = 8192  # SYN backlog
net.core.somaxconn = 65535        # somaxconn ≥ TCP_MAX_SYN_BACKLOG
net.ipv4.tcp_keepalive_time = 600    # keepalive 检测间隔（秒）
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
```

### B.4 生产部署安全检查清单

```bash
# ① 检查镜像是否有 root shell
docker inspect your-image:tag --format='{{.Config.User}}'
# 输出应为非空值（如 "65534"），不能为空（root）

# ② 检查镜像层数（越多越慢，建议 <= 5 层）
docker inspect your-image:tag --format='{{len .RootFS.Layers}}'
# 推荐：< 10（理想 < 5）

# ③ 扫描镜像漏洞
trivy image your-image:tag --severity HIGH,CRITICAL
# 或 grype your-image:tag

# ④ 检查二进制是否静态链接（无运行时依赖）
elfcheck /app 2>/dev/null || readelf -d /app | head -20
# 如果没有 Dynamic section → 完全静态链接 ✅
# 如果有 DT_NEEDED 条目 → 有动态依赖 ⚠️
```

---

## 高频追问

### Q1：strace 追踪一个慢的系统调用很慢，有什么替代方案？
答：`bcc-tools` 的 `funclatency` 比 strace 轻量得多；或者直接用 `perf record -e syscalls:sys_enter_* -p PID` 做采样分析。

### Q2：ss 和 netstat 的输出有什么区别？
答：ss 读取 `/proc/net/*` 内核数据结构直接输出，速度远快于 netstat 解析 `/proc/net/tcp`；而且 ss 支持 filter 语法如 `ss -tna '( dport = :80 or sport = :80 )'`。

### Q3：tcpdump 抓到了 SYN 但没有 ACK，怎么判断问题在哪一层？
答：SYN 发出无 ACK → 可能是防火墙 DROP、路由不可达或目标端口无监听。先用 `ss -tna | grep SYN-SENT` 看本端状态，再用 `tcpdump -i eth0 host target_ip and tcp[tcpflags] & tcp-syn != 0` 确认 SYN 是否真的出去了。

---

### 7. pstack / gstack：线程栈与 Goroutine 栈分析

#### 7.1 背景：什么时候需要看栈？

线上服务偶尔出现 **P99 延迟飙升** 或 **偶发卡顿**，但平均 CPU 不高。这时候你需要知道：**每个线程此刻在执行什么？**

#### 7.2 pstack：查看内核线程调用栈

```bash
# 查看某个 PID 的完整线程栈（含内核态）
pstack <PID>

# 示例输出片段：
# Thread 2 (pid: 12345):
# #0  0x00007f1234567890 in __epoll_wait () from /lib64/libc.so.6
# #1  0x000000000042eabc in runtime.epollevent () at ../../src/runtime/sys_linux_amd64.s:...
# #2  ... runtime.netpoll () at src/runtime/netpoll_epoll.go:...
```

**Go 工程师重点解读：**
| 函数名 | 含义 | 异常信号 |
|--------|------|----------|
| `__epoll_wait` | Go netpoller 在等 I/O | 正常 |
| `runtime.gopark` | goroutine 被阻塞（锁、chan、timer） | 持续大量 → 可能有锁争用 |
| `write()` | 系统调用写磁盘/网络 | 检查是否有 I/O 阻塞 |
| `pthread_cond_timedwait` | 正在等条件变量超时 | 可能 timer 调度延迟 |

#### 7.3 gstack：golang 专用 goroutine 栈查看器

> gstack 是社区工具，比 pstack 更懂 Go 的 M/P/G 模型。

```bash
# 编译安装 gstack
git clone https://github.com/chengrenz/gstack.git
cd gstack && go build -o gstack .

# 使用：直接传入 PID 即可
./gstack <PID>

# 只看活跃 goroutine（排除阻塞的）
./gstack <PID> -active

# 按包统计 goroutine 分布
./gstack <PID> --summary
```

**关键区别——pstack vs gstack：**
| 维度 | pstack | gstack |
|------|--------|--------|
| 看到的内容 | Linux 线程栈（M） | Go goroutine 栈（G）+ M |
| 输出粒度 | 操作系统级 | 语言运行时级 |
| 适合场景 | 排查 CGO、外部依赖卡住 | 排查 Go 并发死锁、goroutine 泄漏 |
| 是否需要符号 | 需要 libunwind | 需要 DWARF 调试信息 |

#### 7.4 实战：定位偶发 P99 飙升

```bash
# 第一步：pprof 确认问题类型
wget http://localhost:6060/debug/pprof/trace?seconds=5 -o trace.out
go tool trace trace.out   # 浏览器打开看 schedule 和 syscall 延迟

# 第二步：pstack 看阻塞点（多轮采样对比）
for i in $(seq 1 10); do
    pstack <PID> > /tmp/stack_$i.txt
    sleep 1
done
# 对比 10 次栈：如果大量线程停在同一个位置 → 找到根因

grep -c 'runtime.gopark' /tmp/stack_*.txt   # 统计被阻塞的 goroutine
```

---

### 8. ulimit / limits.conf：资源限制管理

#### 8.1 为什么 Go 工程师也要关心这个？

很多 Go 服务的 "诡异" 问题其实不是代码 bug，而是 **系统限制了资源**：

| 现象 | 常见原因 | 解决方案 |
|------|----------|----------|
| `accept4: too many open files` | nofile 限制太低 | 调大 nofile |
| `fork: retry: resource temporarily unavailable` | nproc 限制太低 | 调大 nproc |
| `cannot allocate memory` | memlock 限制 | 检查 cgroup 还是 ulimit |
| `setrlimit: operation not permitted` | 已触及 hard limit | 先改 limits.conf 再改 soft |

#### 8.2 核心命令速查

```bash
# 查看当前 shell 的资源限制
ulimit -a

# 输出示例（重点看这一行）：
# open files                      (-n) 1024        ← Go 常在这里踩坑！

# 临时修改当前会话
ulimit -n 65535     # 增大文件描述符上限
ulimit -u 100000    # 增大最大进程数

# ⚠️ 注意：-H（hard）和 -S（soft）的区别
# S = soft limit（可以超过，但有警告）
# H = hard limit（软限不能突破的值，只有 root 能提高）
ulimit -SHn 65535    # 同时设置 soft + hard
```

#### 8.3 持久化配置：limits.conf

```bash
# /etc/security/limits.conf （重启生效）
your-app-user   soft    nofile      65535
your-app-user   hard    nofile      65535
your-app-user   soft    nproc       100000
your-app-user   hard    nproc       100000
your-app-user   soft    memlock     unlimited
your-app-user   hard    memlock     unlimited
```

#### 8.4 Go 容器中的陷阱

```bash
# Docker 容器默认的 nofile 通常是 1048576
docker run --rm alpine sh -c 'ulimit -n'
# 输出：1048576

# 但是！Kubernetes Pod 如果不显式设置，kubelet 会覆盖为 1024！
kubectl exec -n default pod-name -- ulimit -n
# 输出可能是：1024  ← 这就是坑

# K8s 解决方式：添加 SYS_RESOURCE capability
spec:
  containers:
  - name: app
    securityContext:
      capabilities:
        add: ["SYS_RESOURCE"]  # 允许提升 soft limit
```

---

### 9. journalctl / syslog：日志分析与故障取证

#### 9.1 为什么不只靠应用日志？

系统层面的错误（OOM Killer 杀死进程、网络丢包、磁盘坏道）往往**只存在于系统日志中**。Go 应用无法捕获自己的 OOM，必须借助系统日志取证。

```bash
# 查看 kernel 消息
journalctl -k -n 100

# 查找 OOM Killer 记录（最常见的系统级 crash 原因）
journalctl -k | grep -i oom
# 典型输出：
# Out of memory: Killed process 12345 (your-app) total-vm:16384000kB, anon-rss:12582912kB

# 按时间范围查询
journalctl --since "2 hours ago"

# 查看特定服务的日志
journalctl -u your-app.service -f

# 跨主机排障：切换到上一次启动的 boot
journalctl -b -1 | tail -50

# 搜索所有 error 级别以上的日志
journalctl -p err..emerg -n 200
```

#### 9.2 /var/log/syslog vs journalctl

| 维度 | syslog 系列 | journalctl |
|------|-------------|------------|
| 存储格式 | 纯文本 | 二进制（压缩） |
| 日志轮转 | cronolog / logrotate | journal 自带 rotation |
| 结构化 | 无 | supports structured fields |
| 适用 Go 场景 | 兼容性好（旧系统） | 推荐（现代 Linux） |

---

## 延伸阅读

- Linux man pages: `man tcpdump-filter`
- Brendan Gregg《性能之殇》火焰图章节
- Go pprof 官方文档：https://go.dev/pkg/net/http/pprof/
- [Google Container Best Practices](https://cloud.google.com/architecture/best-practices-for-building-containers)
- [Dockerfile Best Practices](https://docs.docker.com/build/building/best-practices/)
- [gstack — Go-aware stack viewer](https://github.com/chengrenz/gstack)
- [Understanding ulimit and /etc/security/limits.conf](https://linux.die.net/man/1/ulimit)
