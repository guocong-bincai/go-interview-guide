# 四层 vs 七层负载均衡：LVS/Nginx、健康检查与算法选型

> 考察频率：★★★★☆  优先级：P1
> 关键词：四层负载均衡、七层负载均衡、LVS、DR 模式、IPVS、Nginx、一致性哈希、最小连接数、健康检查、会话保持、DSR

---

## 🎯 面试官考察意图

负载均衡是**"网络 + 架构"的交叉题**。面试官通常会从"你们线上流量怎么进来的"切入，考察：

1. 你能不能**分清 L4 和 L7 的本质差异**（不是"一个看 IP 一个看 URL"这么浅）；
2. 知不知道 **LVS 的三种模式（NAT/DR/TUN）**，尤其 DR 模式为什么能扛百万并发；
3. 会不会**选负载均衡算法**（轮询够不够？什么时候必须一致性哈希？）；
4. 有没有处理过**会话保持**和**健康检查误判**的实际问题。

**关键区分点**：L4 转发的是"连接"，L7 转发的是"请求"。这个区别决定了它们的性能上限、功能边界和故障表现。

---

## ⚡ 核心答案（30秒）

| 维度 | 四层（L4） | 七层（L7） |
|------|-----------|-----------|
| 工作层次 | 传输层（TCP/UDP） | 应用层（HTTP/HTTPS/gRPC） |
| 转发单位 | **连接** | **请求** |
| 代表 | LVS、F5、DPDK、云 SLB | Nginx、HAProxy、Envoy、云 ALB |
| 性能 | **极高**（10万~百万级 QPS，内核态/DPDK） | 高（万级 QPS，用户态） |
| 功能 | 仅端口/IP 级转发，不看内容 | 可按 URL/Header/Cookie 路由，可改包、可鉴权、可缓存 |
| 后端可见性 | 后端看到的是 **LB 的 IP**（NAT）或 **客户端真实 IP**（DR） | 天然透传真实 IP（通过 `X-Forwarded-For`） |
| 加密 | 不能终止 TLS（只能透传或卸载后重组） | 可终止 TLS、可做证书管理 |
| 典型位置 | 最外层，抗 DDoS、抗大流量 | L4 之后，做业务路由 |

**一句话话术：**
> L4 转发连接，速度快但看不懂内容；L7 转发请求，能按业务路由但性能低一个数量级。生产架构一般是 **L4（LVS/云 SLB）在最外层扛流量 → L7（Nginx/Envoy）做业务路由 → 应用**。

---

## 🔬 深度展开

### 1. 为什么 L4 能比 L7 快这么多

**L4 模式下，请求根本不需要到达用户态。**

```
L7（Nginx）转发路径：
  网卡 → 内核协议栈 → epoll → Nginx 用户态处理 HTTP 解析
       → 重新构造请求 → 内核协议栈 → 网卡
  数据要从内核态拷贝到用户态，再拷贝回去。CPU 消耗在：
  - 用户态/内核态切换（每次请求 4 次以上上下文切换）
  - HTTP 解析（字符串处理）
  - 内存拷贝（body 通常要拷 2~3 次）

L4（LVS DR）转发路径：
  网卡 → 内核协议栈 → IPVS 模块修改 MAC 地址 → 网卡
  全程在内核态完成，数据零拷贝（DR 模式），CPU 只做 MAC 头改写。
```

**量级差异（生产经验值）：**

| 方案 | 单机 QPS | CPU 特征 |
|------|---------|---------|
| LVS DR + 16 核 | 50 万 ~ 100 万 | 网卡中断为主 |
| DPDK 用户态转发 | 100 万+ | 独占核轮询，零中断 |
| Nginx（静态小响应） | 5 万 ~ 10 万 | 用户态 CPU |
| Nginx（带 proxy_pass） | 2 万 ~ 5 万 | 用户态 + 上下文切换 |
| Envoy（带一堆 L7 策略） | 1 万 ~ 3 万 | 用户态 |

> L4 的单机能力比 L7 高 **一到两个数量级**。所以第一层一定要用 L4。

### 2. LVS 三种模式详解（高频考点）

#### （1）NAT 模式（DNAT）

```
客户端 ──► LVS（改目的 IP 为 RS）──► Real Server
                                      │
                                      └─网关必须指向 LVS──► LVS 改源 IP ──► 客户端
```

| 特点 | 说明 |
|------|------|
| 原理 | LVS 改写目的 IP（DNAT），回包时改写源 IP（SNAT） |
| 优点 | 配置简单，Real Server 无感知 |
| 缺点 | **LVS 是双向流量瓶颈**（请求和响应都过 LVS）；RS 的网关必须指向 LVS |
| 适用 | 小规模、内网 |

#### （2）DR 模式（Direct Routing，生产最常用）

```
                          ┌──────────────────────┐
客户端 ──► LVS ──改MAC──► │ Real Server（也配 VIP）│
                          └──────────────────────┘
                                    │
                                    └──► 直接回给客户端（响应不经 LVS）
```

| 特点 | 说明 |
|------|------|
| 原理 | LVS 只改**目的 MAC**（链路层），IP 不变；RS 直接把响应发给网关 |
| 优点 | **响应不经过 LVS**，LVS 只处理请求方向 → 抗压能力是 NAT 的几倍到几十倍 |
| 缺点 | LVS 和 RS 必须在**同一个二层网络**；RS 需要配 VIP 并**抑制 ARP 响应** |
| 关键配置 | RS 上 `arp_ignore=1`、`arp_announce=2`，VIP 配在 **lo 接口**（`lo:0`）上 |

**RS 端配置（必须背下来，面试常问）：**

```bash
# 1) 抑制 ARP：避免 RS 响应 VIP 的 ARP 请求，导致流量不进 LVS
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce

# 2) 在 lo 上配 VIP（注意是 lo，不是 eth0）
ip addr add 192.168.1.100/32 dev lo:0
ip link set lo:0 up

# 3) 屏蔽 RS 的 ARP 广播
# （配合上面的 arp_ignore/arp_announce 已足够）
```

> **为什么 VIP 配在 `lo` 上？** 配在 `eth0` 上 RS 会对外宣告自己也拥有 VIP，导致客户端流量直接打到 RS，绕过 LVS。配在 `lo`（回环）上则只有本机可见，用于接收 LVS 转发过来的包（目的 IP 是 VIP）。

#### （3）TUN 模式（IP 隧道）

| 特点 | 说明 |
|------|------|
| 原理 | LVS 把原始包封装进 IP 隧道发给 RS，RS 解封装后处理，响应直连客户端 |
| 优点 | RS 可以**跨机房**（不要求同一二层网络） |
| 缺点 | 需要 RS 支持 IPIP 隧道；封装开销 |

**三种模式对比：**

| 模式 | 请求路径 | 响应路径 | 跨网段 | 性能 | 生产使用 |
|------|---------|---------|-------|------|---------|
| NAT | 经 LVS | 经 LVS | 否 | 低 | 少 |
| **DR** | 经 LVS | **直连** | 否 | **高** | **最多** |
| TUN | 经 LVS | 直连 | 是 | 中 | 跨机房 |

### 3. 负载均衡算法：什么时候用什么

| 算法 | 原理 | 适用 | 缺点 |
|------|------|------|------|
| **轮询 RR** | 依次分配 | 后端同构、无状态 | 忽略后端实际负载 |
| **加权轮询 WRR** | 按权重分配 | 后端配置不同（新老机器混部） | 静态权重不适应动态变化 |
| **最小连接数 LC** | 分给当前连接最少的 | 请求耗时差异大（长连接、慢接口） | 需要维护连接计数 |
| **最短响应时间** | 分给平均响应最快的 | 对延迟敏感 | 采样窗口问题 |
| **一致性哈希** | 相同 key 固定到同一后端 | **有状态服务、缓存亲和** | 后端增减时流量迁移 |
| **IP Hash** | 按客户端 IP 哈希 | 简单会话保持 | NAT 后大量用户同 IP → 倾斜 |
| **随机** | random | 大集群下近似均匀 | 短期可能不均 |

**面试必答的对比：轮询 vs 最小连接数**

```
轮询的致命问题：
  后端 A 的接口耗时 1s，后端 B 耗时 10ms
  轮询把 50% 流量给了 A → A 的连接堆积 → 雪崩
  最小连接数会自动把更多流量给 B（因为 B 连接数少）

但最小连接数也有问题：
  短连接场景下连接数变化太快，统计不准
  → 所以 Nginx 默认不是 LC，需要显式配置
```

```nginx
upstream backend {
    # 最小连接数
    least_conn;

    # 或者加权轮询（默认）
    # server 10.0.0.1:8080 weight=3 max_fails=2 fail_timeout=10s;

    server 10.0.0.1:8080 weight=3 max_fails=2 fail_timeout=10s;
    server 10.0.0.2:8080 weight=1 max_fails=2 fail_timeout=10s;

    keepalive 128;  # 与后端保持长连接，减少握手开销
}

# IP Hash（会话保持）
upstream backend_hash {
    ip_hash;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}

# 一致性哈希（推荐，支持 hash key 自定义）
upstream backend_chash {
    hash $arg_user_id consistent;   # 按 user_id 一致性哈希
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

**一致性哈希的坑（必问）：**

```
问题：普通取模 hash(key) % N
  3 台机器时 key="user123" → hash % 3 = 1 → 机器 B
  扩容到 4 台 → hash % 4 = 3 → 机器 D
  后果：几乎所有 key 都换了机器，缓存全部失效（雪崩）

一致性哈希：
  哈希环 0 ~ 2^32，机器和 key 都映射到环上
  扩容时只影响相邻的一段（理论是 1/N 的 key 迁移）
  
但裸一致性哈希有数据倾斜问题（机器少时分布不均）
  → 引入【虚拟节点】：每台机器映射 100~200 个虚拟节点到环上
  → 分布均匀性显著改善
```

> `hash $arg_user_id consistent` 中的 `consistent` 就是启用 ketama 一致性哈希（带虚拟节点）。**不带 `consistent` 是普通取模**，扩容会全量失效。

```go
// Go 里实现一致性哈希（面试常要求手写）
// 生产用 github.com/buraksezer/consistent 或 groupcache/consistenthash
type HashRing struct {
	replicas int
	keys     []int             // 排序后的虚拟节点哈希
	ring     map[int]string    // 虚拟节点哈希 → 真实节点
}

func New(replicas int) *HashRing {
	return &HashRing{replicas: replicas, ring: make(map[int]string)}
}

func (h *HashRing) Add(nodes ...string) {
	for _, node := range nodes {
		for i := 0; i < h.replicas; i++ {
			// 关键：虚拟节点 key 必须包含节点名，保证同一节点稳定
			hash := int(crc32.ChecksumIEEE([]byte(strconv.Itoa(i) + node)))
			h.keys = append(h.keys, hash)
			h.ring[hash] = node
		}
	}
	sort.Ints(h.keys)
}

func (h *HashRing) Get(key string) string {
	if len(h.keys) == 0 {
		return ""
	}
	hash := int(crc32.ChecksumIEEE([]byte(key)))
	// 二分找到第一个 >= hash 的虚拟节点，没有则回到环首
	idx := sort.Search(len(h.keys), func(i int) bool { return h.keys[i] >= hash })
	if idx == len(h.keys) {
		idx = 0
	}
	return h.ring[h.keys[idx]]
}
```

### 4. 健康检查：最容易出事故的地方

**两层健康检查，职责不同：**

| 层级 | 检查方 | 检查方式 | 目的 |
|------|-------|---------|------|
| **LB 层** | Nginx/LVS | 主动探测（HTTP/TCP） | 摘除故障实例 |
| **应用层** | K8s kubelet | `readinessProbe` / `livenessProbe` | 同上，但更贴近应用 |

**Nginx 被动健康检查：**

```nginx
upstream backend {
    server 10.0.0.1:8080 max_fails=2 fail_timeout=10s;
    server 10.0.0.2:8080 max_fails=2 fail_timeout=10s;
    server 10.0.0.3:8080 backup;   # 备用，主节点全挂才启用
}
```

- `max_fails=2`：**10 秒内**失败 2 次即摘除；
- `fail_timeout=10s`：摘除持续 10 秒，之后放一个请求探测；
- 失败判定包括 `error`、`timeout`，以及 `proxy_next_upstream` 里配置的 HTTP 状态码。

> ⚠️ **默认 `proxy_next_upstream` 不含 5xx**！也就是说后端返回 500 不会被计入 `max_fails`。想要按业务状态码摘除，必须显式配置：
> ```nginx
> proxy_next_upstream error timeout http_500 http_502 http_503;
> ```
> 但要小心：**含 500 的重试可能导致重复写操作**。

**主动健康检查（需要 `nginx_upstream_check_module` 或 Nginx Plus）：**

```nginx
upstream backend {
    server 10.0.0.1:8080;
    check interval=3000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "HEAD /healthz HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}
```

**健康检查接口的正确写法（Go）：**

```go
// ✅ 存活检查：只反映"进程是否需要重启"
// 不要检查下游依赖！否则 DB 抖动会导致所有实例同时被重启 → 级联故障
func livez(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("ok"))
}

// ✅ 就绪检查：反映"是否可以接收流量"
// 可以检查关键依赖，但要快速、有超时、有缓存
var ready atomic.Bool

func readyz(w http.ResponseWriter, r *http.Request) {
	if !ready.Load() {
		http.Error(w, "not ready", http.StatusServiceUnavailable)
		return
	}
	// 检查关键依赖（带超时）
	ctx, cancel := context.WithTimeout(r.Context(), 500*time.Millisecond)
	defer cancel()
	if err := db.PingContext(ctx); err != nil {
		http.Error(w, "db unavailable", http.StatusServiceUnavailable)
		return
	}
	w.WriteHeader(http.StatusOK)
}
```

**三个必须避开的坑：**

| 坑 | 后果 | 正确做法 |
|----|------|---------|
| `livenessProbe` 里检查 DB/Redis | DB 抖动 → 所有实例被 K8s 重启 → 服务全挂 | liveness 只检查进程自身 |
| 健康检查接口耗时 2s | 健康检查超时 → 误摘除 | 健康检查必须 < 100ms |
| 健康检查无节流 | 高频探测放大下游压力 | 加缓存（如 1 秒内复用上次结果） |

### 5. 会话保持（Session Affinity）方案对比

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **IP Hash** | 按客户端 IP 分配 | 简单 | NAT 后大量用户同 IP → 严重倾斜 |
| **Cookie 插入** | LB 插入 `Set-Cookie: SRV=node1` | 精确 | 需要 LB 支持（Nginx Plus / HAProxy） |
| **一致性哈希 key** | 按业务 key（user_id）哈希 | 精确且可控 | 需要业务传 key |
| **集中式 Session（推荐）** | Session 存 Redis，实例无状态 | **任一实例都能处理** | 多一次 Redis 访问 |

> **最佳实践**：不要用会话保持。把 Session 外置到 Redis，让所有实例无状态。这也是 K8s 环境下唯一可行的方案（Pod 会漂移、会重建）。

### 6. 生产架构示例

```
                    ┌──────────────┐
    用户 ──────────►│  DNS / CDN   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ L4: 云 SLB   │  ← 抗 DDoS、扛流量、四层转发
                    │ (LVS/DPDK)   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Nginx #1 │ │ Nginx #2 │ │ Nginx #3 │  ← L7：路由/鉴权/限流/缓存
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
      ┌──────┴────────────┴────────────┴──────┐
      │            Go 微服务集群                │
      │  (K8s + 服务网格, gRPC/HTTP)            │
      └───────────────────────────────────────┘
```

**为什么需要两层？**

| 只有 L7 | L4 + L7 |
|---------|---------|
| Nginx 直接暴露公网，易被打 | L4 抗 DDoS、隐藏真实拓扑 |
| 单机 QPS 上限低，需要很多机器 | L4 能扛百万 QPS，L7 只需少量 |
| TLS 终止在 Nginx，证书管理分散 | 可以 L4 透传 + L7 终止，灵活 |
| Nginx 挂了入口就挂了 | L4 可做多活、防单点 |

---

## ❓ 高频追问

### Q1：LVS DR 模式下后端服务怎么拿到真实客户端 IP？

DR 模式**不改 IP 头**（只改 MAC），所以后端收到的包的源 IP 就是**客户端真实 IP**。这是 DR 相对于 NAT 的一个额外好处。

NAT 模式下后端看到的是 LVS 的内网 IP，需要靠 `X-Forwarded-For`（L7）或 `TOA`（TCP Option Address，L4）来传递真实 IP。

```go
// Go 服务获取真实 IP 的正确顺序
func ClientIP(r *http.Request) string {
	// 1) 优先 X-Forwarded-For 的最后一个（最靠近服务端的代理追加的）
	if xff := r.Header.Get("X-Forwarded-For"); xff != "" {
		parts := strings.Split(xff, ",")
		// 注意：XFF 可以被客户端伪造，只在可信代理后使用
		// 取第一个非可信代理的 IP（简化：取最后一个）
		ip := strings.TrimSpace(parts[len(parts)-1])
		if net.ParseIP(ip) != nil {
			return ip
		}
	}
	// 2) X-Real-IP（Nginx 设置）
	if rip := r.Header.Get("X-Real-IP"); rip != "" {
		return rip
	}
	// 3) 兜底：RemoteAddr
	host, _, err := net.SplitHostPort(r.RemoteAddr)
	if err != nil {
		return r.RemoteAddr
	}
	return host
}
```

> ⚠️ **安全提示**：`X-Forwarded-For` 是可以伪造的。如果你的服务直接暴露公网，攻击者可以随便写 XFF 来绕过 IP 限流。正确做法是**只在可信代理链后信任 XFF**，并且 Nginx 配置 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for`（追加而非覆盖）。

### Q2：为什么不用 DNS 轮询做负载均衡？

| 问题 | 说明 |
|------|------|
| 无法感知后端故障 | DNS 不知道后端挂了，会继续返回挂掉的 IP |
| TTL 缓存 | 客户端/本地 DNS 缓存导致变更生效慢（分钟到小时级） |
| 无法做健康检查和流量控制 | 没有摘流、没有权重、没有灰度 |
| 分配不均 | 客户端缓存策略不同，流量分布不可控 |

DNS 适合做**机房级/区域级**的流量调度（GSLB），不适合做实例级 LB。

### Q3：一致性哈希扩容时怎么减少影响？

1. **虚拟节点**：每台机器映射 100~200 个虚拟节点，减少数据倾斜；
2. **逐步扩容**：一次只加少量节点，观察缓存命中率和 DB 压力；
3. **双读/预热**：新节点加入后先预热（把热点数据提前加载进去）再放流量；
4. **分层哈希**：用 `hash(user_id) % bucket_count` 先映射到虚拟桶，桶再映射到物理机，扩容时只移动桶。

### Q4：Nginx 的 `keepalive` 指令为什么必须配合 `proxy_http_version 1.1` 和 `proxy_set_header Connection ""`？

因为 **HTTP/1.0 默认是短连接**。Nginx 默认用 HTTP/1.0 与 upstream 通信，所以：

```nginx
upstream backend {
    server 10.0.0.1:8080;
    keepalive 128;          # ① 维持 128 个空闲长连接
}

location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;             # ② 必须！否则 upstream 用 1.0，长连接不生效
    proxy_set_header Connection "";     # ③ 必须！清掉客户端传来的 Connection: close
}
```

三个配置缺一不可。漏掉 ② 或 ③，`keepalive` 完全不生效，每次请求都新建 TCP 连接，后端 TIME_WAIT 暴涨。

### Q5：后端实例的权重怎么定？

| 依据 | 说明 |
|------|------|
| CPU 核数 | 最常见，8 核给 8，4 核给 4 |
| 内存 | 内存密集型服务（如缓存） |
| 实测压测 QPS | 最准确，但需定期校准 |
| 上线时间 | 新实例权重渐进提升（如 10% → 50% → 100%），避免冷启动问题 |

**注意**：权重是**静态**的，无法应对"某台机器被其他进程抢了 CPU"的情况。所以**权重 + 最小连接数**的组合比纯权重轮询更稳。

---

## 📋 总结 checklist

- [ ] 能说出 L4 转发"连接"、L7 转发"请求"的本质区别
- [ ] 能解释 L4 为什么比 L7 快一到两个数量级
- [ ] 能说清 LVS 三种模式的请求/响应路径差异
- [ ] 知道 DR 模式 RS 要配 VIP 到 `lo` + 抑制 ARP
- [ ] 会选负载均衡算法（什么时候必须一致性哈希）
- [ ] 知道裸一致性哈希的倾斜问题和虚拟节点解决法
- [ ] 知道 Nginx `keepalive` 必须配合 `proxy_http_version 1.1` + `Connection ""`
- [ ] 知道 `livenessProbe` 里不能检查 DB
- [ ] 知道 `X-Forwarded-For` 可伪造，要配合可信代理使用

---

## 📚 延伸阅读

- [LVS 官方文档: 三种转发模式](http://www.linuxvirtualserver.org/software/ipvs.html)
- [Nginx upstream 模块文档](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [Consistent Hashing and Random Trees (原始论文)](https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf)
- [Kubernetes: 配置存活/就绪/启动探针](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
