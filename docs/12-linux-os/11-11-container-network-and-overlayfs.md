[🏠 首页](../../../README.md) · [📦 Linux / 操作系统基础](../README.md)

---

# 容器网络底层（netns / veth / bridge / iptables）与 OverlayFS 镜像分层

> 考察频率：★★★★★  优先级：P0
> 关键词：network namespace、veth pair、Linux bridge、iptables NAT、conntrack、CNI、VXLAN MTU、OverlayFS、copy-up、whiteout、ephemeral storage

---

## 面试官考察意图

只要简历上有 **K8s / 容器 / 微服务**，这两道题必问。
面试官想确认你是"会 `kubectl apply`"还是"知道一个包从 Pod 到 Pod 到底走了哪些内核组件"。排障能力的分水岭就在这里：**跨节点调用偶发超时 / 容器磁盘写满 / 大文件第一次写极慢**，答案全在这两层。

---

# 第一部分：容器网络

## 1. 一个 Pod 的网络栈长什么样

```
K8s 节点（宿主机 netns）                      Pod A（独立 netns）
┌───────────────────────────────────────┐   ┌──────────────────────┐
│  eth0 (物理网卡) 172.16.1.10           │   │  eth0  10.244.1.5    │
│     │                                  │   │   │                  │
│  ┌──┴───────────────┐                  │   │  (veth 的内侧)        │
│  │  cni0 / docker0  │  ← Linux bridge  │   │                      │
│  │  (网桥, 二层交换) │                  │   └──────────────────────┘
│  └──┬───────────────┘                          ▲
│     │  veth1a (10.244.1.5 的对端)               │ ② veth pair 是一对"网线两端"
│     └──────────────────────────────────────────┘
│  
│  iptables (nat 表)  ← 出网 MASQUERADE / 入端口 DNAT
└───────────────────────────────────────┘
```

**三个组件，一句话概括：**
- **network namespace**：给 Pod 一套独立的协议栈（网卡、路由表、iptables、端口空间）
- **veth pair**：一对虚拟网卡，"一端插进容器，一端插在宿主机"，像一根虚拟网线
- **Linux bridge**：虚拟二层交换机，把同一节点上的所有 veth 对端连起来，学习 MAC 转发表

## 2. netns：协议栈隔离

```bash
# 造一个 network namespace（手搓"容器网络"）
ip netns add ns1
ip netns exec ns1 ip addr          # 只有 lo，没有 eth0

# 造一根 veth 网线，两端分别放进 netns 和宿主机
ip link add veth1 type veth peer name veth1-peer
ip link set veth1 netns ns1
ip link set veth1-peer master br0   # 宿主机端插进网桥

# 配置地址 + 路由
ip netns exec ns1 ip addr add 10.244.1.5/24 dev veth1
ip netns exec ns1 ip link set veth1 up
ip netns exec ns1 ip route add default via 10.244.1.1

# 排障三件套
ip netns exec ns1 ss -lntp
ip netns exec ns1 ip route
nsenter -t <pid> -n ip addr          # 进运行中容器的 netns
```

**关键点**：netns 隔离的是 **IP/端口/路由/iptables/conntrack**，所以**同一节点上两个容器可以都用 80 端口**；但注意 **conntrack 表在 netns 内独立计算**（容易忽略：`sysctl net.netfilter.nf_conntrack_max` 需要按 netns 设置）。

## 3. veth pair 与 Linux bridge

```
veth pair 的本质：一对"对接的管道"
   netns A 侧 veth1 发出的帧 ──> 直接出现在宿主机侧 veth1-peer 上（内核内存拷贝，不是网线）

所以：veth 内部流量会消耗 CPU（每包至少一次 skb 拷贝），
     同节点 Pod 互访的吞吐上限由 CPU 决定，不是网卡 —— 
     这也是 Cilium/eBPF 用 sockmap 走"同节点短路"优化性能的原因。
```

```bash
bridge fdb show br br0        # 网桥 MAC 表：哪个 MAC 在哪个 veth 后面
brctl show                    # 或 ip link show master br0
ip neigh show                 # ARP 表（Pod → MAC）

# 网桥也是一个内核实体，可以设 ip / 网关 / 挂 iptables
# ★ 重要坑：net.bridge.bridge-nf-call-iptables
#   若为 1，网桥转发的帧也会走 iptables → 规则写错会导致同节点 Pod 不通
sysctl net.bridge.bridge-nf-call-iptables
```

## 4. iptables / NAT：跨节点与外部访问的大门

### 4.1 出网：SNAT / MASQUERADE

```
Pod (10.244.1.5) → 访问 8.8.8.8

  nat 表 POSTROUTING 链（K8s 自动生成）：
  -A POSTROUTING -s 10.244.0.0/16 ! -o cni0 -j MASQUERADE

  效果：源地址被改写为节点 IP (172.16.1.10)，回包再由 conntrack 反查改回
```

### 4.2 入站：DNAT 端口映射

```
外部访问 节点IP:30080 → Service ClusterIP:80 → Pod 10.244.1.5:8080

  nat 表：
  -A PREROUTING -j DNAT --to-destination 10.244.1.5:8080
  + 节点上的 kube-proxy 规则（iptables 模式）
```

### 4.3 conntrack：高并发下最容易炸的一环（必考）

```bash
# 查看连接跟踪表使用量与溢出统计
conntrack -S
# entries            12345   ← 当前表项
# insert_failed       1234   ← 插入失败数量（命中上限！）
# drop                1234   ← 丢包数
# invalid             12

sysctl net.netfilter.nf_conntrack_max          # 常见 65536（太小！）
sysctl net.netfilter.nf_conntrack_count
dmesg | grep -i "nf_conntrack: table full"
```

**故障形态：** 流量一上来，**偶发随机超时/丢包，且丢的是 SYN 包**（新连接失败，老连接没事），CPU/带宽都不忙。
**根因：** conntrack 表满 → 新连接无法建立跟踪 → 丢包。NAT 场景下**每个连接 2 条记录**（正/反），所以 `max` 实际只够 3 万连接。
**处置：** 调大 `nf_conntrack_max`（同时调 `hashsize`）；缩短 `nf_conntrack_tcp_timeout_established`（默认 5 天！静态场景可降到 1 小时）；开启 `tcp_loose`；更彻底的做法是用 **IPVS 模式**或 **eBPF（Cilium）绕过 conntrack 的部分开销**。

```bash
# 生产推荐基线（按内存权衡，每条约 300 字节）
sysctl -w net.netfilter.nf_conntrack_max=1048576
sysctl -w net.netfilter.nf_conntrack_buckets=262144     # ≈ max/4
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=3600
```

## 5. 跨节点：CNI 的三种实现路线

| 方案 | 封装 | 性能 | 特点 | 排障关键 |
|------|------|------|------|---------|
| **Flannel VXLAN** | 用户态/内核 UDF 封装（默认内核态） | 中 | 简单、跨网段可用 | **MTU 必须 −50**；VXLAN 端口 8472/UDP |
| **Calico BGP** | 无封装（underlay 路由） | 高 | 三层路由，Pod IP 直接可路由 | 依赖网络设备允许 BGP / 或 IPIP 兜底 |
| **Cilium eBPF** | 无/可选 | 最高 | 绕过 iptables + conntrack，XDP 加速 | 需要较新内核（5.x+） |

### 5.1 MTU：跨节点最隐蔽的"偶发超时"

```
物理网卡 MTU 1500
  └─ VXLAN 封装头 = 8(VXLAN) + 8(UDP) + 20(IP) + 14(以太) ≈ 50 字节
        ↓
  Pod 内网卡 MTU 应为 1450（Flannel 默认会设置）

⚠️ 若 Pod 内仍是 1500：
   小包（如 ping、API 小请求）正常 → 大包（如上传、gRPC 大批量）被丢且无响应
   现象：接口"偶发超时"，重试有时成功（分片后变小）→ 极难定位
```

```bash
# 定位：用"不分片 + 指定大小"实测
ping -M do -s 1472 <pod-ip>     # 1472 + 28 = 1500
ping -M do -s 1422 <pod-ip>     # 1422 + 28 = 1450 ← 跨 VXLAN 应在此通过

# 抓包看是否有 ICMP "Fragmentation needed"（PMTU 黑洞时不会有）
tcpdump -i any -n -v 'icmp[icmptype] == 3'
```

**Go 侧配套**：`http.Server` 无需改；但 gRPC 的 `MaxRecvMsgSize`（默认 4MB）与 TCP MSS 无关，别混淆。真正要做的是**确保 CNI 设对 MTU**，并在服务网格/网关层考虑 MSS clamp（`iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu`）。

## 6. 容器网络排障 SOP

```bash
# ① Pod 内到宿主机不通 → 查 netns 内地址/路由
nsenter -t <pid> -n ip addr show && nsenter -t <pid> -n ip route

# ② 同节点 Pod 互访不通 → 查网桥 MAC 表 + iptables FORWARD 链计数
bridge fdb show br cni0
iptables -t filter -L FORWARD -n -v | grep -v " 0 "

# ③ 跨节点不通 → 查 VXLAN 隧道 + MTU
ip -d link show flannel.1
tcpdump -i flannel.1 -n udp port 8472

# ④ 出网不通 → 查 nat 表 MASQUERADE 计数 + conntrack
iptables -t nat -L POSTROUTING -n -v

# ⑤ DNS 不通 → CoreDNS / 网络策略（NetworkPolicy）常被遗忘
kubectl exec -it <pod> -- nslookup kubernetes.default
```

---

# 第二部分：OverlayFS 与容器镜像分层

## 7. OverlayFS 的工作原理

### 7.1 联合挂载（union mount）

```
OverlayFS 四要素：

  lowerdir  （只读层，可多层 : 分隔）   /var/lib/docker/overlay2/<id>/diff
  upperdir  （可写层，容器独有）         /var/lib/docker/overlay2/<id>/diff
  workdir   （内部工作目录，必须与 upper 同文件系统）
  merged    （合并后的挂载点，容器看到的 /）

  读文件：先找 upper，没有则按 lower 从新到旧找
  写文件：文件在 lower → 先 copy-up 到 upper，再改
  删文件：在 upper 创建 whiteout（字符设备 0/0）遮住下层同名文件
```

```bash
mount | grep overlay
# overlay on / type overlay (rw,lowerdir=...,upperdir=...,workdir=...)

# 查看层数（层越多，查找路径越长，性能越差）
docker inspect <container> | jq '.[0].GraphDriver.Data'
```

### 7.2 copy-up：为什么"大文件第一次写很慢"

```
容器内修改 base 镜像里的一个 1GB 文件：

  ① overlayfs 发现该文件只在 lowerdir
  ② 把整个 1GB 文件复制到 upperdir   ← 磁盘占用瞬间 +1GB，耗时秒级
  ③ 在 upperdir 副本上应用修改
  ④ 以后所有写都直接落 upperdir

⚠️ 所以：
  - "容器里改了个 2GB 日志文件" → 磁盘双倍占用 + 长阻塞
  - 数据库/日志目录必须挂 volume，否则全是 overlay，性能差 + 层膨胀
```

```bash
# 查看每层实际占用的差异（定位层膨胀）
du -sh /var/lib/docker/overlay2/*/diff | sort -h | tail
```

### 7.3 稀疏文件与 inode 黑洞

```bash
df -h   # 显示空间充足
df -i   # ❗ inode 耗尽 → "no space left on device" 但还有空间
```
**故障形态：** 服务报 `write: no space left on device`，但 `df -h` 还有 50% 空间。原因：overlay 层里堆了海量小文件（如未清理的临时文件、Go 编译缓存），**inode 用尽**。`df -i` 一查就露馅。

## 8. 镜像分层与启动性能

```
镜像分层设计（一份 Dockerfile）：

  Layer 1: FROM golang:1.24-alpine       ← 共享（所有 Go 项目复用）
  Layer 2: COPY go.mod go.sum            ← 缓存命中率高
  Layer 3: RUN go mod download
  Layer 4: COPY . .
  Layer 5: RUN go build -o /app          ← 层内 500MB 编译产物
  ──────────── 多阶段构建分界线 ────────────
  Layer 6: FROM alpine:3.20  + COPY --from=builder /app

关键收益：
  ① 最终镜像只含运行产物（几十 MB 而非 1GB+）
  ② 拉取时各层并行 + 跨节点复用（基础层已在本地）
  ③ 层不可变 → 可被多个容器共享，page cache 也共享
```

**Page cache 共享这一条常被追问：** lowerdir 是只读的，多个容器挂同一层时**读同一份镜像文件共享 page cache**，所以"启动 100 个容器"不会导致 100 份文件内存。但 **copy-up 之后各自的 upperdir 副本是独立 page cache**，写多的文件会让内存占用陡增。

## 9. 容器存储相关的生产坑

| 现象 | 根因 | 处置 |
|------|------|------|
| `no space left on device` 但 `df -h` 有余 | inode 耗尽 / 已被删除但 fd 未释放的文件 | `df -i`；`lsof +L1` 找 deleted 文件 |
| 容器磁盘写满导致 Pod 被驱逐 | ephemeral-storage limit | 设置 `resources.limits.ephemeral-storage` + 日志轮转 |
| 大文件写慢（秒级卡顿） | overlay copy-up 整文件复制 | 挂 volume（hostPath/CSI）写数据 |
| 镜像拉取慢 | 层数过多 / 层顺序不合理 | 多阶段构建 + 合并 RUN + 大层放最前 |
| 启动时 `mount` 报 `workdir` 错误 | workdir 与 upperdir 不同文件系统 | 确保都在 `/var/lib/docker` 同一 FS |
| Pod 内 `du` 和宿主机对不上 | 统计的是 merged 视图 | 直接看 `upperdir` |

**推荐文件系统**：`overlay2`（默认，K8s 推荐）配合 **XFS**（`ftype=1`，支持 `d_type`，overlay2 必需）或 ext4。**ext4 + overlay2 在老内核上有兼容问题**，新内核已修复。

## 10. 高频追问

**Q1：为什么容器里的 `ps` 看不到宿主机进程？**
PID namespace 隔离。容器内 PID 1 映射到宿主机某个 PID，容器内看不到宿主机其他进程。注意：**PID 1 在内核中有特殊语义（默认忽略未注册信号的默认动作）**，所以容器里的 Go 程序作为 PID 1 时 `SIGTERM` 可能被忽略 → 需要 `tini` / `docker run --init` 或应用自己 `signal.Notify`。

**Q2：为什么 `docker run --net=host` 能让服务"性能更好"？**
跳过 veth/bridge/iptables 全链路，直接用宿主机协议栈，省掉每次收发的 veth 拷贝与 NAT 查表。代价是没有网络隔离、端口冲突、Service 发现失效。

**Q3：同节点 Pod 互访走不走 iptables？**
取决于 `bridge-nf-call-iptables`。若开启，网桥转发也会经过 iptables FORWARD 链 → 规则写错会影响同节点通信，且每包多一次查表。Cilium 的 eBPF 方案会把这部分短路。

**Q4：OverlayFS 支持硬链接吗？copy-up 后硬链接会怎样？**
早期不支持跨层硬链接（copy-up 会打断硬链接关系）。较新内核加入 **index 特性**（`index=on`），在 upperdir 维护硬链接索引，保证 copy-up 后硬链接关系仍成立。**这也是"某些镜像在旧内核上行为不一致"的原因。**

**Q5：VXLAN 和 IPIP 的区别？**
VXLAN 是 L2 over UDP（封装整个以太帧，可跨三层网络构建大二层，端口 4789/8472）；IPIP 是 L3 隧道（封装 IP 包，无 UDP 头，开销更小但不支持 L2 广播）。Calico 支持两者，IPIP 性能更好但依赖底层网络允许 IP 协议 4。

**Q6：容器 OOM 和宿主机 OOM 的关系？**
cgroup memory.max 是容器自己的上限，超限杀容器内进程（`OOMKilled`）；宿主机内存不足时由内核 OOM Killer 按 `oom_score` 杀进程（**容器内进程的 score 含 cgroup 因素**）。**dentry/slab 计入容器内存但不计入 Go 的 heap**，所以会出现"Go 程序堆很小，容器却 OOM"。

---

## 延伸阅读

- [Linux Network Namespace 实操（ip netns）](https://man7.org/linux/man-pages/man8/ip-netns.8.html)
- [Kubernetes 网络模型官方文档](https://kubernetes.io/docs/concepts/services-networking/)
- [OverlayFS 内核文档](https://docs.kernel.org/filesystems/overlayfs.html)
- [nf_conntrack 调优（Cloudflare）](https://blog.cloudflare.com/how-to-drop-10-million-packets/)
- [Cilium eBPF 数据路径](https://docs.cilium.io/en/stable/network/ebpf/)
