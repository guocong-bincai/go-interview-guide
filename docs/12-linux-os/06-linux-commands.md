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

## 延伸阅读

- Linux man pages: `man tcpdump-filter`
- Brendan Gregg《性能之殇》火焰图章节
- Go pprof 官方文档：https://go.dev/pkg/net/http/pprof/
