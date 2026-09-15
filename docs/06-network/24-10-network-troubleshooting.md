# 网络排查工具箱：tcpdump / ss / mtr / iftop 实战

> 考察频率：★★★☆☆  优先级：P2
> 关键词：tcpdump、ss、netstat、mtr、iftop、nstat、网络排查、丢包、延迟、TIME_WAIT、连接数统计

---

## 🎯 面试官考察意图

这道题常以**场景题**的形式出现：

> "用户反馈服务偶发超时，你的排查步骤是什么？"
> "ping 得通但 HTTP 访问不了，为什么？"
> "怎么确认是网络问题还是应用问题？"

面试官想看的是**方法论**：你是否有"从链路层到应用层逐层排除"的框架，而非只会重启服务。

**核心原则：先分层定位，再深入细节；先用统计数据（`ss`/`nstat`）缩小范围，再用抓包（`tcpdump`）确认细节。**

---

## ⚡ 核心答案（30秒）

**排查分层模型（自下而上）：**

| 层 | 工具 | 回答什么问题 |
|----|------|-------------|
| 链路/路由 | `ping`、`mtr`、`traceroute` | 通不通？在哪一跳丢包？ |
| 传输统计 | `ss`、`nstat`、`netstat -s` | 连接状态、队列溢出、重传率 |
| 流量/带宽 | `iftop`、`nload`、`sar -n DEV` | 带宽打满了吗？谁在传？ |
| 协议细节 | `tcpdump`、`tshark` | 包长什么样？什么时候发的？ |
| 应用 | `curl -v`、`strace`、pprof | 应用层慢了还是网络慢了？ |

**最常用的 5 条命令（背下来）：**

```bash
# 1) 看连接状态分布（瞬间定位 TIME_WAIT/CLOSE_WAIT/队列问题）
ss -s

# 2) 看监听队列（Recv-Q 堆积 = accept 慢）
ss -lnt

# 3) 看内核 TCP 统计（重传、溢出、错误）
nstat -az | grep -Ei "retrans|overflow|drop|error"

# 4) 抓包看三次握手耗时（定位是网络还是应用）
sudo tcpdump -i any -nn -tttt 'tcp port 8080 and tcp[tcpflags] & tcp-syn != 0'

# 5) 一直观察连接数变化
watch -n1 'ss -ant | awk "{print \$1}" | sort | uniq -c'
```

---

## 🔬 深度展开

### 1. `ss`：取代 netstat 的连接观测工具

**为什么用 `ss` 而不是 `netstat`？**
`netstat` 遍历 `/proc/net/tcp` 全表，连接数上万时非常慢（秒级）；`ss` 用 netlink 直接从内核取数据，**快一个数量级**。

```bash
# 连接状态汇总
ss -s
# Total: 1234
# TCP:   891 (estab 600, closed 200, orphaned 0, timewait 90)
#          ↑ 其中 timewait 数量很关键

# 监听端口（重点看 Recv-Q / Send-Q）
ss -lnt
# State   Recv-Q  Send-Q  Local Address:Port
# LISTEN  0       4096    0.0.0.0:8080
#          ↑ 全连接队列中待 accept 的数，接近 Send-Q 就是危险信号

# 查看某个端口的连接状态分布
ss -ant 'sport = :8080' | awk 'NR>1{print $1}' | sort | uniq -c | sort -rn

# 查看已建立的连接详情（含重传、RTT、拥塞窗口）
ss -tio state established '( sport = :8080 )'
# 输出字段解读：
#   rtt:0.15/0.02    当前 RTT / 平均 RTT（ms）
#   rto:200          重传超时时间
#   cwnd:10          拥塞窗口
#   retrans:0/3      当前重传次数 / 累计重传次数  ← 关键指标
#   send 1.5Mbps     发送速率
#   bytes_sent:12345 已发送字节

# 只看重传 > 0 的连接（网络质量差的证据）
ss -tio | grep -v "retrans:0/0" | grep retrans

# 查看处于某个状态的连接
ss -ant state time-wait | wc -l
ss -ant state close-wait          # CLOSE_WAIT 多 = 应用没关连接（代码 bug！）
ss -ant state syn-recv | wc -l    # SYN_RECV 多 = SYN Flood 或队列问题
```

**状态异常的诊断表：**

| 状态 | 数量异常多意味着 | 排查方向 |
|------|----------------|---------|
| `TIME_WAIT` | 短连接太多 | 启用连接池、`tcp_tw_reuse` |
| `CLOSE_WAIT` | **应用收到 FIN 后没调 Close** | **一定是代码 bug**，查 `defer resp.Body.Close()` |
| `SYN_RECV` | SYN Flood 或半连接队列满 | SYN Cookie、`tcp_max_syn_backlog` |
| `FIN_WAIT_2` | 对端没发 FIN，`tcp_fin_timeout` 生效中 | 检查对端应用行为 |
| `ESTABLISHED` 且 Recv-Q 高 | 应用读得慢 | 查应用 CPU / 处理逻辑 |

> ⚠️ **`CLOSE_WAIT` 是最典型的应用层 bug**：TCP 收到对端的 FIN 后进入 `CLOSE_WAIT`，**只有应用调用 `close()` 才会离开**。如果一直堆积，说明你的 Go 代码里 `resp.Body.Close()` 漏了，或者连接从连接池取出后没归还。

### 2. `nstat` / `netstat -s`：内核 TCP 统计

```bash
nstat -az | grep -Ei "retrans|overflow|drop|error|timeout"
```

**关键计数器含义：**

| 计数器 | 含义 | 健康阈值 |
|--------|------|---------|
| `TcpRetransSegs` | TCP 重传段数 | 重传率 < 1%（`TcpRetransSegs / TcpOutSegs`） |
| `TcpExtListenOverflows` | 全连接队列溢出次数 | **0**（非 0 就要查 accept 循环） |
| `TcpExtListenDrops` | 入队失败丢弃次数 | 0 |
| `TcpExtTCPReqQFullDrop` | 半连接队列溢出丢弃 SYN | 0 |
| `TcpExtTCPTimeouts` | RTO 超时次数 | 低 |
| `TcpExtTCPAbortOnClose` | 因 close 而中止 | 关注趋势 |
| `TcpOutSegs` / `TcpInSegs` | 收发包数 | 用于算重传率 |
| `TcpAttemptFails` | 建连失败次数 | 低 |
| `TcpExtTCPSynRetrans` | SYN 重传次数 | 低（高则网络丢包严重） |

```bash
# 计算重传率（最实用的网络质量指标）
old_out=$(nstat -az | awk '/^TcpOutSegs/{print $2}')
old_ret=$(nstat -az | awk '/^TcpRetransSegs/{print $2}')
sleep 60
new_out=$(nstat -az | awk '/^TcpOutSegs/{print $2}')
new_ret=$(nstat -az | awk '/^TcpRetransSegs/{print $2}')
echo "重传率: $(echo "scale=4; ($new_ret-$old_ret)*100/($new_out-$old_out)" | bc)%"
# > 1% 说明网络有明显丢包，需要找链路问题
```

### 3. `mtr`：网络质量的可视化诊断

`mtr` = `ping` + `traceroute` 的合体，**持续探测每一跳的丢包率和延迟**，是判断"网络抖动在谁家"的最佳工具。

```bash
# TCP 模式（更贴近真实业务，能穿过只管 TCP 的防火墙）
mtr --tcp --port 443 -r -c 100 api.example.com

# 输出示例：
# HOST: localhost            Loss%   Snt   Last   Avg  Best  Wrst StDev
#   1. 192.168.1.1            0.0%   100    1.2   1.3   1.0   2.5   0.3
#   2. 10.0.0.1               0.0%   100    3.4   3.5   3.1   5.0   0.4
#   3. ???                   100.0%   100    0.0   0.0   0.0   0.0   0.0   ← 该跳不响应 ICMP
#   4. 203.0.113.5            2.0%   100   12.0  15.0  11.0  89.0  10.2   ← 从这跳开始丢包
#   5. api.example.com        2.0%   100   13.0  16.0  12.0  90.0  11.0
```

**读图要点：**

1. **中间跳 100% 丢包但后面正常** → 该路由器不响应 ICMP，**不是故障**（很常见，别误判）；
2. **某跳开始丢包且持续到终点** → 故障点在那一跳之后；
3. **只有最后一跳丢包** → 对端服务器的问题（可能被限速/防火墙丢包）；
4. **Avg 和 Wrst 差距大（StDev 大）** → 线路抖动（可能拥塞）。

> ⚠️ `mtr` 在**客户端到服务端**跑才有意义。如果要在服务端排查，用 `mtr` 反向探测经常被中间设备限速，结果不可靠。

### 4. `tcpdump`：终极真相工具

**基本用法：**

```bash
# 抓某端口的包（-nn 不做 DNS/端口名解析，-tttt 显示完整时间戳）
sudo tcpdump -i any -nn -tttt 'tcp port 8080'

# 抓包写文件（生产环境必用，避免终端刷屏影响性能）
sudo tcpdump -i any -nn -s 0 -w /tmp/cap.pcap 'tcp port 8080'
# -s 0 = 抓完整包（不截断），默认只抓 262144 字节

# 限制抓包数量，避免磁盘写满
sudo tcpdump -i any -nn -c 1000 -w /tmp/cap.pcap 'tcp port 8080'

# 常用过滤器
'tcp port 8080 and host 10.0.0.5'          # 指定主机+端口
'tcp[tcpflags] & tcp-syn != 0'             # 只看 SYN 包
'tcp[tcpflags] & (tcp-rst) != 0'           # 只看 RST 包（定位连接被拒）
'tcp port 80 and greater 1000'             # 只看大于 1000 字节的包
```

**实战场景 1：三次握手耗时分析**

```bash
# 抓握手包，看时间差
sudo tcpdump -i any -nn -tttt 'tcp port 8080 and (tcp[tcpflags] & tcp-syn != 0)'
# 15:04:05.123456 IP 10.0.0.1.54321 > 10.0.0.2.8080: Flags [S], seq 123
# 15:04:05.124012 IP 10.0.0.2.8080 > 10.0.0.1.54321: Flags [S.], seq 456, ack 124
# 15:04:05.124089 IP 10.0.0.1.54321 > 10.0.0.2.8080: Flags [.], ack 457
#      ↑ SYN→SYN+ACK 用了 0.56ms（正常），若 > 100ms 说明有网络延迟或队列等待
```

**实战场景 2：定位 RST（连接被重置）**

```bash
sudo tcpdump -i any -nn -tttt 'tcp[tcpflags] & (tcp-rst) != 0'
# 15:04:05.2 IP 10.0.0.2.8080 > 10.0.0.1.54321: Flags [R], seq 457
#      ↑ 服务端主动 RST：可能是端口没监听、队列溢出(tcp_abort_on_overflow=1)、
#        或应用用 SetLinger(0) 主动 RST
```

**实战场景 3：40ms 延迟毛刺（Nagle + 延迟确认）**

```bash
sudo tcpdump -i any -nn -tttt -A 'tcp port 8080 and host 10.0.0.5'
# 看图里两个数据包的时间差是否是 40ms 的整数倍
```

**实战场景 4：丢包与重传**

```bash
sudo tcpdump -i any -nn 'tcp port 8080' -w /tmp/cap.pcap
# 用 Wireshark 打开：
#   Analyze → Expert Information  会直接列出"Retransmission"、"Duplicate ACK"
#   或 filter: tcp.analysis.retransmission
```

**生产环境注意事项：**

| 注意点 | 说明 |
|-------|------|
| 抓包有性能开销 | 高流量机器上抓全量包可能丢包甚至影响业务；务必加过滤器 + 限包数 |
| 抓包文件含敏感数据 | 可能包含明文密码、Cookie、业务数据，**不要随意上传/外发** |
| 磁盘空间 | `-w` 写文件时监控磁盘，用 `-c` 限包数 |
| 权限 | 需要 root 或 `CAP_NET_RAW` |

### 5. `iftop` / `nload` / `sar`：带宽观测

```bash
# 按连接实时看带宽占用（谁在传数据）
sudo iftop -i eth0 -nNP
# 输出：
#   10.0.0.2:8080  =>  10.0.0.5:54321     1.2Mb   1.5Mb   1.3Mb
#   10.0.0.2:8080  <=  10.0.0.6:12345    800Kb   900Kb   850Kb
#   ↑ => 发送  ↑ <= 接收

# 网卡总带宽
nload eth0

# 历史带宽（sar 需要 sysstat，能看过去的数据）
sar -n DEV 1 10
sar -n DEV -f /var/log/sa/sa15    # 看 15 号的带宽历史

# 网络错误/丢包（网卡层面）
ip -s link show eth0
# RX: bytes packets errors dropped overrun mcast
#               ↑ error/dropped 非 0 说明网卡或驱动有问题
```

### 6. 完整排查案例：接口 P99 从 50ms 涨到 2s

```
现象：接口 P99 突增到 2s，P50 正常（50ms）→ 典型的"长尾"问题
      ↓
步骤 1：确认是网络还是应用
  $ curl -w "@curl-format.txt" -o /dev/null -s https://api.example.com/xxx
  观察 time_connect / time_starttransfer / time_total 的分布
  - time_connect 高 → 建连慢（网络或队列）
  - time_starttransfer 高但 connect 正常 → 服务端处理慢
      ↓
步骤 2：看服务端内核指标
  $ nstat -az | grep -Ei "retrans|overflow"
  TcpExtListenOverflows 0      → 队列没问题
  TcpRetransSegs 增长明显       → 网络丢包
      ↓
步骤 3：看连接状态
  $ ss -s
  TCP: 5000 (estab 4800, closed 100, timewait 90)
  $ ss -tio state established '( sport = :8080 )' | grep -v "retrans:0/0" | head
  发现若干连接 retrans 很高
      ↓
步骤 4：抓包确认
  $ sudo tcpdump -i any -nn -c 500 -w /tmp/cap.pcap 'tcp port 8080 and host <可疑IP>'
  用 Wireshark 打开，filter: tcp.analysis.retransmission
  发现大量重传 + 部分连接的 RTT 从 0.5ms 涨到 200ms
      ↓
步骤 5：向上游/下层定位
  $ mtr --tcp --port 8080 -r -c 100 <上游IP>
  发现在第 4 跳开始丢包
      ↓
结论：中间网络设备拥塞导致丢包 → 重传 → 长尾延迟
      （不是应用问题，找网络团队）
```

**若第 2 步发现 `ListenOverflows` 增长** → 转向"accept 慢"排查（见《TCP 半连接队列、全连接队列与 SYN Flood 防护》）。
**若第 3 步发现 `CLOSE_WAIT` 堆积** → 转向代码 bug 排查（`resp.Body.Close()` 漏调用）。

### 7. Go 应用侧的网络排查工具

```go
// 生产必备：pprof（暴露 goroutine 栈，定位卡在网络 IO 的 goroutine）
import _ "net/http/pprof"

go func() {
    log.Println(http.ListenAndServe("127.0.0.1:6060", nil))
}()
```

```bash
# goroutine 栈：看有多少 goroutine 卡在 net.(*netFD).Read
curl -s http://127.0.0.1:6060/debug/pprof/goroutine?debug=1 | grep -A5 "net.*Read"

# 阻塞分析：看是否卡在网络 IO
curl -s "http://127.0.0.1:6060/debug/pprof/block?seconds=30" -o block.prof
go tool pprof -top block.prof

# 连接跟踪（Go 1.20+）：追踪单个连接的完整生命周期
GODEBUG=http2debug=2 ./app
```

```go
// 自定义指标：把 ss / nstat 里看不到的业务指标暴露出来
var (
	connTotal = promauto.NewGauge(prometheus.GaugeOpts{
		Name: "app_tcp_conn_total", Help: "当前活跃 TCP 连接数",
	})
	connWait = promauto.NewGauge(prometheus.GaugeOpts{
		Name: "app_db_wait_count",
		Help: "数据库连接池等待次数",
	})
)

// 定期采集
func collect() {
	ticker := time.NewTicker(10 * time.Second)
	defer ticker.Stop()
	for range ticker.C {
		stats := db.Stats()
		connWait.Set(float64(stats.WaitCount))
	}
}
```

### 8. 排查工具速查表

| 我要确认 | 用什么 |
|---------|-------|
| 端口在监听吗 | `ss -lntp \| grep :8080` |
| 队列溢出吗 | `nstat -az \| grep ListenOverflow` |
| 连接状态分布 | `ss -s` / `ss -ant \| awk '{print $1}' \| sort \| uniq -c` |
| 哪个连接在重传 | `ss -tio \| grep -v "retrans:0/0"` |
| 重传率高吗 | `nstat` 差分计算 |
| 哪一跳丢包 | `mtr --tcp -r -c 100 <host>` |
| 带宽打满了吗 | `iftop` / `sar -n DEV 1` |
| 建连耗时多少 | `tcpdump` 看 SYN→SYN+ACK 时间差 |
| 连接为什么被拒 | `tcpdump 'tcp[tcpflags] & tcp-rst != 0'` |
| 握手为什么慢 | `tcpdump` + Wireshark 的 TCP Stream Graph |
| 应用卡在哪 | pprof goroutine / block profile |
| 是否有 conntrack 满 | `dmesg \| grep conntrack` |
| 网卡是否丢包 | `ip -s link show eth0` |

---

## ❓ 高频追问

### Q1：`ping` 得通但 HTTP 访问不了，可能是什么原因？

`ping` 用 ICMP，HTTP 用 TCP，两者走不同的协议处理路径。可能原因：

| 原因 | 验证方法 |
|------|---------|
| 防火墙拦了 TCP 端口，但放行了 ICMP | `telnet host 80` / `nc -zv host 80` |
| 服务没监听（进程挂了） | `ss -lntp \| grep :80` |
| SYN 队列满 | `nstat -az \| grep ReqQFull` |
| conntrack 表满 | `dmesg \| grep conntrack` |
| MTU 问题（大包被丢） | `ping -M do -s 1472 host` |
| TCP 被中间设备 RST | `tcpdump` 看是否有 RST |

> 反过来"**ping 不通但 HTTP 能通**"也很常见：很多云主机默认禁 ICMP，或中间设备丢弃 ICMP。

### Q2：怎么判断是网络问题还是应用问题？

**最快的方法：看 `curl -w` 的时间分解。**

```bash
curl -w "@-" -o /dev/null -s https://api.example.com/xxx <<'EOF'
     time_namelookup:  %{time_namelookup}s\n
        time_connect:  %{time_connect}s\n
     time_appconnect:  %{time_appconnect}s\n
    time_pretransfer:  %{time_pretransfer}s\n
       time_redirect:  %{time_redirect}s\n
  time_starttransfer:  %{time_starttransfer}s\n
                     ----------\n
          time_total:  %{time_total}s\n
EOF
```

| 指标 | 高说明什么 |
|------|-----------|
| `time_namelookup` | DNS 解析慢 |
| `time_connect` - `time_namelookup` | TCP 建连慢（网络/队列） |
| `time_appconnect` - `time_connect` | TLS 握手慢（证书链、密钥交换） |
| `time_starttransfer` - `time_appconnect` | **服务端处理慢**（应用问题） |
| `time_total` - `time_starttransfer` | 响应传输慢（带宽/客户端） |

### Q3：`tcpdump` 抓包会不会影响线上性能？

会，但可控：

| 因素 | 影响 |
|------|------|
| 抓包量 | 全量抓包在高 PPS 场景下 CPU 开销明显（可达 5~20%） |
| 写磁盘 | `-w` 写文件受磁盘 IO 限制，高流量下可能丢包 |
| 缓冲区 | 默认缓冲小，高流量下内核缓冲区溢出导致丢包（看 `tcpdump` 结束时的 `dropped by kernel`） |

**生产实践：**
1. **一定要加过滤器**（只抓特定端口/IP）；
2. **限制包数和每包字节**（`-c 1000`，`-s 96` 只抓头部）；
3. **提高缓冲区**（`-B 4096`）减少内核丢包；
4. **优先用 `ss`/`nstat` 做初步定位**，只在确有必要时抓包；
5. 用 `ebpf` 方案（如 Cilium Hubble、Pixie）做持续可观测性，比临时抓包更友好。

### Q4：没有 root 权限怎么排查网络问题？

| 需求 | 无 root 的替代方案 |
|------|------------------|
| 看连接状态 | `ss -ant`（普通用户可看自己进程的） |
| 看监听端口 | `ss -lnt`（可以，但看不到进程名，需要 root） |
| 抓包 | `setcap cap_net_raw,cap_net_admin+eip /usr/bin/tcpdump` 授权 |
| 看内核统计 | `nstat -az`（多数系统普通用户可读 `/proc/net/snmp`） |
| ping/mtr | 通常可用（`ping` 有 setuid 或 capabilities） |
| 应用层 | Go 的 pprof、应用日志、Trace |

### Q5：如何做持续的网络可观测性（而不是临时抓包）？

| 方案 | 特点 |
|------|------|
| **eBPF**（Cilium/Hubble、Pixie、bcc-tools） | 零侵入，内核态采集，可看每个请求的 TCP 重传/RTT |
| **Prometheus + node_exporter** | 采集 `node_netstat_*` 指标，配 Grafana 看趋势和告警 |
| **服务网格** | Envoy 侧的 `upstream_rq_time`、`upstream_cx_*` 指标 |
| **OpenTelemetry** | 应用层 span + 网络层指标的关联 |
| **日志聚合** | Nginx access log 的 `upstream_response_time` 分布 |

**关键指标要持续监控（而非出事了才看）：**

```
- TCP 重传率（> 1% 告警）
- ListenOverflows / ListenDrops（> 0 告警）
- conntrack 使用率（> 80% 告警）
- TIME_WAIT / CLOSE_WAIT 数量趋势
- 接口 P99 延迟（分位数而非平均）
- 各依赖的调用耗时分布
```

---

## 📋 总结 checklist

- [ ] 能说出排查的 5 层框架（链路/统计/带宽/协议/应用）
- [ ] 知道 `ss` 比 `netstat` 快的原理（netlink vs 遍历 proc）
- [ ] 知道 `ss -lnt` 里 `Recv-Q`/`Send-Q` 在 LISTEN 状态的含义
- [ ] 知道 `CLOSE_WAIT` 堆积是应用 bug（漏 `Close()`）
- [ ] 会用 `nstat` 算重传率，知道 1% 是健康阈值
- [ ] 会用 `mtr --tcp`，知道"中间跳 100% 丢包不一定是故障"
- [ ] 会写 `tcpdump` 过滤器和抓 RST/SYN 的技巧
- [ ] 知道生产抓包必须加过滤器和限包数
- [ ] 会用 `curl -w` 分解耗时，快速区分网络慢还是应用慢

---

## 📚 延伸阅读

- [Linux 内核文档: IP Sysctl](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)
- [tcpdump 手册页](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [ss 与 iproute2 使用指南](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Wireshark: TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html)
- [Cilium: eBPF-based Networking and Observability](https://cilium.io/)
