<div align="center">

# 🏆 Go 高级工程师面试宝典

> **面向 5~8 年 Go 后端工程师 · 大厂面试核心考点全覆盖**

[![Stars](https://img.shields.io/github/stars/guocong-bincai/go-interview-guide?style=flat-square&logo=github&color=yellow)](https://github.com/guocong-bincai/go-interview-guide/stargazers)
[![Forks](https://img.shields.io/github/forks/guocong-bincai/go-interview-guide?style=flat-square&logo=github&color=blue)](https://github.com/guocong-bincai/go-interview-guide/network/members)
[![License](https://img.shields.io/github/license/guocong-bincai/go-interview-guide?style=flat-square&color=green)](./LICENSE)
[![题目数量](https://img.shields.io/badge/题目-491-orange?style=flat-square)](./docs)
[![版本](https://img.shields.io/badge/版本-v5.26-blue?style=flat-square)](./docs)

[📚 模块导航](#-模块导航) · [🗺️ 学习路线](#️-学习路线) · [📝 更新记录](#-更新记录) · [🤝 贡献指南](#-贡献指南)

</div>

---

## 📚 模块导航

> 共 **491** 道高频面试题 ｜ 12 大核心模块 ｜ 按面试优先级排序

| 序号 | 模块 | 题数 | 频率 | 优先级 | 覆盖内容 |
|:----:|------|:----:|:----:|:------:|----------|
| 01 | [**Go 语言深度**](docs/01-golang/README.md) | **96** | ★★★★★ | P0 | GMP/GC/内存分配/channel/sync/interface/泛型/逃逸分析/pprof/false sharing/block profile/race detector/runtime.Pinner/runtime.metrics/unique 驻留/cgo.Handle/unsafe.Slice/GOTOOLCHAIN/fuzzing
| 02 | [**数据库**](docs/02-database/README.md) | **36** | ★★★★★ | P0 | MySQL 索引/MVCC/锁/双写一致性/Bloom Filter 防穿透/读写分离延迟/乐观悲观锁选型/Redis 数据结构选型 |
| 03 | [**分布式系统**](docs/03-distributed/README.md) | **58** | ★★★★☆ | P1 | CAP/BASE/Raft/2PC/TCC/Saga/Kafka/gRPC/限流熔断/Gossip/ConsistentHash/负载均衡/SentinelCluster/MQ交付语义/指数退避/API Gateway/CQRS
| 04 | [**微服务工程**](docs/04-microservices/README.md) | **31** | ★★★★☆ | P1 | gRPC/Protobuf/网关/限流/服务注册发现/配置中心/服务网格/链路追踪采样/可观测性/K8s/CI-CD/无损发布/服务降级/BFF/框架选型
| 05 | [**系统设计**](docs/05-system-design/README.md) | **34** | ★★★★★ | P0 | 秒杀/短链/IM/Feed流/支付/订单/库存/点赞/消息推送/弹幕/优惠券/缓存一致性/CDN/API网关/WebSocket/深度分页
| 06 | [**网络协议**](docs/06-network/README.md) | **25** | ★★★★☆ | P1 | TCP 三次握手/队列与 SYN Flood/Nagle 与 TCP_NODELAY/HTTP1.1-2-3/HTTP 缓存/CORS/HTTPS/gRPC/WebSocket/502-504-499 排查/L4-L7 负载均衡/Range 断点续传/网络排查工具/安全 |
| 07 | [**高频算法**](docs/07-algorithms/README.md) | **99** | ★★★★☆ | P2 | 滑动窗口/二分/回溯/DP/链表/树/单调栈/堆/TopK/LFU/Trie/最短路/海量数据/排序链表/快速选择/树序列化/逆序对/KMP |
| 08 | [**工程素养**](docs/08-engineering/README.md) | **41** | ★★★★★ | P1 | 技术选型/架构演进/OOM排查/CIDC流水线/可观测性/测试覆盖率/DB迁移/Docker优化/金丝雀发布/依赖管理/告警治理/供应链安全/PGO/可复现构建/Flaky Test/持续性能分析 |
| 09 | [**面试策略**](docs/09-interview-strategy/README.md) | **14** | ★★★☆☆ | P1 | STAR法则/行为面试/简历写法/自我介绍/系统设计面试/Live Coding/薪资谈判/晋升答辩/全流程节奏控制/技术深挖应对/HR面全攻略/反问环节攻略 |
| 10 | [**项目实战问题**](docs/10-real-problems/README.md) | **8** | ★★★★★ | P0 | 业务方案/性能问题/数据一致性/可用性/并发/资源泄漏与安全问题 |
| 11 | [**Go 标准库生产实践**](docs/11-go-std-practice/README.md) | **26** | ★★★★★ | P1 | reflect反射/regexp编译缓存/http中间件链/filepath路径安全/sync.Map + sync单飞/nil-closed channel/defer+named return/context/errors/http.Client + net/url编码陷阱/Request.Body与MaxBytesReader/bufio.Scanner 64KB/os.Root防穿越/OnceFunc家族/netip/crypto-rand/优雅退出/httptest |
| 12 | [**Linux / 操作系统**](docs/12-linux-os/README.md) | **23** | ★★★★★ | P1 | 文件系统/进程线程/虚拟内存/零拷贝/cgroup/IPC/Swap/OOM Killer评分/core dump/TCP调优/proc/sysfs/CPU缓存一致性MESI与内存屏障/内核Buddy与Slab分配器/RCU与自旋锁/容器网络veth-bridge-iptables与conntrack/OverlayFS镜像分层/进程D状态与僵尸进程/fsync落盘与崩溃一致性 |

---

## 🗺️ 学习路线

### 阶段一：基础夯实（1~2 周）

- [📦 01 · Go 语言深度](docs/01-golang/README.md) — GMP、GC、内存、并发原语是必考基础
- [🐧 12 · Linux / 操作系统](docs/12-linux-os/README.md) — 进程线程、虚拟内存、零拷贝
- [🌍 06 · 网络协议](docs/06-network/README.md) — TCP/HTTP 是后端面试地基

### 阶段二：核心进阶（2~3 周）

- [🗄️ 02 · 数据库](docs/02-database/README.md) — MySQL 索引/MVCC + Redis 缓存三兄弟
- [🌐 03 · 分布式系统](docs/03-distributed/README.md) — 一致性、事务、消息队列
- [🔧 04 · 微服务工程](docs/04-microservices/README.md) — RPC、网关、可观测性、K8s

### 阶段三：实战冲刺（2 周）

- [🏗️ 05 · 系统设计](docs/05-system-design/README.md) — 秒杀/短链/IM 高频设计题
- [🔥 10 · 项目实战问题](docs/10-real-problems/README.md) — 线上真实问题的解决思路
- [📐 07 · 高频算法](docs/07-algorithms/README.md) — 面试手撕代码必备
- [🛠️ 08 · 工程素养](docs/08-engineering/README.md) — 5~8 年工程师的区分度
- [💼 09 · 面试策略](docs/09-interview-strategy/README.md) — STAR 法则、薪资谈判

---

## 📝 更新记录

| 日期 | 版本 | 更新内容 |
|------|------|----------|
| 2026-09-23 | v5.26 | Go 语言深度模块新增 8 题（补齐运行时/语言机制/标准库/工具链中此前未覆盖的现代 API）：runtime.Pinner（Go 1.21，cgo 指针传递规则、Pin 后对象为 GC root 不被回收、Unpin 缺失导致永久泄漏、与 cgo.Handle 的分工、cgocheck 校验）、runtime/metrics（Go 1.16+，Sample/Read 无 STW 采集、ReadMemStats 会 STW 造成 P99 毛刺、指标命名稳定性与 KindBad 校验、Float64Histogram 直方图、与 pprof/expvar/Prometheus 的分工）、unique 包值驻留（Go 1.23，unique.Make 返回可比较 Handle、相同值全局一份、分片表 + 弱引用自动回收、低基数高重复值适用而 TraceID 类唯一值不适用、与自建 sync.Map 缓存/Java intern 对比）、cgo.Handle（Go 1.17，C 侧只保存 uintptr 整数句柄不违反指针规则、NewHandle/Value/Delete 生命周期、Delete 缺失导致强引用泄漏、句柄复用竞态、与 Pinner 选型原则）、unsafe.Slice / unsafe.String（Go 1.20，旧写法 `*(*[]byte)(unsafe.Pointer(&s))` 因 string header 无 Cap 字段而越界、手搓 reflect.SliceHeader 的 uintptr 非 GC 引用问题、checkptr 检测方法、sub-slicing 内存泄漏与零拷贝共享生命周期）、Go 1.22 增强 ServeMux（方法前缀匹配、{id} 路径变量与 PathValue、{path...} 尾通配、{$} 精确匹配、注册期冲突 panic 与最具体优先、自动 405 与 Allow 头、与 gin/chi 选型边界）、GOTOOLCHAIN（Go 1.21，go/toolchain 两行指令语义与版本选择算法、auto 自动下载 vs local 固定、生产 CI 用 GOTOOLCHAIN=local 保证可复现构建、供应链与离线考量）、原生 Fuzzing（Go 1.18，FuzzXxx/f.Add/f.Fuzz 写法与参数类型限制、覆盖率引导变异、crasher 最小化后写入 testdata/fuzz 形成回归闭环、不变量断言与避免非确定性、CI 短时 fuzz 与 -race 组合），共新增 8 题，模块题数 88 → 96，全库 483 → 491 |
| 2026-09-22 | v5.25 | Linux / 操作系统模块新增 9 题（补齐此前未覆盖的硬件与内核底层方向）：CPU 缓存一致性与 MESI 协议（cache line 64B、M/E/S/I 四态流转、MESIF/MOESI 工业变体、store buffer + invalidate queue 导致 MESI 只保证最终一致）、内存屏障（LoadLoad/StoreStore/LoadStore/StoreLoad 四类、StoreLoad 最贵、x86 TSO 仅允许 StoreLoad 重排而 ARM64 弱模型几乎全部允许、Go 内存模型 hb 规则与 atomic 强序语义、Double-Checked Locking 在 Go 里的 sync.Once / atomic.Pointer 正确写法、race detector 与 perf c2c 排查）、内核内存分配器（Buddy System 2^order 伙伴拆分合并与 XOR 寻伙伴、内部碎片 vs 外部碎片、/proc/buddyinfo 判读、Slab/SLUB per-CPU freelist、shrinker 回收、dentry/inode slab 计入 cgroup memory.stat 导致「Go 堆很小但容器 OOM」、与 Go mcache/mcentral/mheap 逐层对比、malloc→mmap→缺页→Buddy 全链路）、内核同步机制（spinlock 两条铁律：临界区必须极短且中断上下文需 irqsave、mutex 可睡眠仅进程上下文、RCU 三步 copy-update-publish 与 grace period 静止态判定、RCU 读写扩展性对比 rwlock、Go 中 atomic.Pointer COW 与 sync.Map read/dirty 双层结构作为 RCU 等价物）、容器网络底层（netns 隔离协议栈与 conntrack 按 netns 独立、veth pair 是同节点流量走 CPU 拷贝的吞吐上限来源、Linux bridge MAC 表与 bridge-nf-call-iptables 坑、iptables MASQUERADE/DNAT 与 conntrack 表满导致偶发 SYN 丢包的生产调优基线、Flannel VXLAN/Calico BGP/Cilium eBPF 三路线选型、VXLAN MTU 1500→1450 导致大包静默丢弃的 PMTU 黑洞定位法、五步排障 SOP）、OverlayFS（lower/upper/work/merged 四要素、copy-up 整文件复制导致大文件首写秒级卡顿、whiteout 删除语义、layer 共享 page cache、ephemeral-storage 与 inode 耗尽导致「df -h 有余却写失败」）、进程状态（R/S/D/T/Z 全表、D 状态不可中断睡眠与 kill -9 无效的根因、wchan 与 /proc/PID/stack 定位、NFS 硬挂载与 direct reclaim 两大元凶、load average 统计 R+D、僵尸进程只占 PID 表项不占内存、孤儿进程被 init 收养、Go 中 Cmd.Start 不 Wait 与 Process.Release 误用导致僵尸堆积、容器 PID 1 忽略 SIGTERM 与 tini/subreaper 方案）、fsync 落盘与崩溃一致性（write 只写 page cache、dirty_ratio 20% 触发同步回写造成周期性 P99 毛刺与 dirty_bytes 调优、fsync vs fdatasync vs O_DIRECT、原子替换文件 8 步含 fsync 父目录、ext4 data=journal/ordered/writeback 三模式、delayed allocation 与集中分配停顿、fsync 父目录缺失导致掉电后文件消失、云盘 fsync 5~20ms 的预算控制与 Group Commit、overlayfs 上 fsync 触发 copy-up 的坑），共新增 9 题，模块题数 14 → 23，全库 474 → 483 |
| 2026-09-21 | v5.24 | Go 标准库生产实践模块新增 9 题（聚焦「上线才会炸」的边界条件）：net/url 编码陷阱（QueryEscape 空格转 `+` 而 `+` 不转义导致签名错位、PathEscape 不转义 `/`、跨语言签名必须自实现 RFC3986、url.JoinPath 保留 `/` 的穿越风险、url.URL 结构体拼装优于字符串加法）、http.Request.Body 只能读一次（GetBody 重放与手搓 Request 时 GetBody 为 nil、http.MaxBytesReader + *http.MaxBytesError 防 OOM 而 Content-Length 可伪造、Drain+Close 才归还连接池否则 TIME_WAIT 暴涨、TeeReader 一次读取同时校验与转发）、bufio.Scanner 64KB 陷阱（MaxScanTokenSize、不检查 Err() 会静默截断数据、Buffer/ReadString/ReadSlice 三方案选型、sc.Bytes() 缓冲会被覆盖必须 copy）、os.Root（Go 1.24，fd 级子树隔离根治路径穿越与 TOCTOU、Clean+前缀校验的三个漏洞、服务端生成 UUID 文件名 + 上传目录 nosuid,nodev,noexec）、sync.OnceFunc/OnceValue/OnceValues（Go 1.21，错误也被缓存故不能用于失败重试场景、panic 会被重现、与 SingleFlight 的进程级 vs 并发窗口级辨析）、net/netip（值类型可比较可做 map key、v4-mapped 两种表示导致 IP 白名单绕过、解析零分配快 4~5 倍、SSRF 防护必须含 169.254.169.254 元数据地址与 DNS Rebinding）、crypto/rand vs math/rand（可预测性漏洞、Go 1.20 全局自动随机化与 rand/v2 移除 Seed 改 ChaCha8、Go 1.24 rand.Text()、subtle.ConstantTimeCompare 防时序攻击）、signal.NotifyContext 优雅退出（用已取消 ctx 调 Shutdown 直接失效、摘流量/preStop/terminationGracePeriod 时序、Keep-Alive 空闲连接不由 Shutdown 关闭、后台 goroutine 需 ctx+WaitGroup 收口）、httptest/httputil（NewRecorder 无网络层测不出超时与连接复用、NewServer 测 client 与反向代理、表驱动 + t.Parallel、DumpRequest 会消耗 Body）；并恢复此前未成功推送的工程素养模块 6 篇（告警治理/供应链安全/PGO/可复现构建/Flaky Test/持续性能分析），模块题数 11 模块 17 → 26、08 模块 35 → 41，全库 459 → 474 |
| 2026-09-17 | v5.23 | 高频算法模块新增 9 题（补齐此前未覆盖的进阶方向）：LFU 缓存（哈希 + 频率分桶双向链表 + minFreq O(1) 实现、为什么 minFreq 无需扫描、LFU 缓存污染与 TinyLFU/Redis allkeys-lfu 衰减方案、LFU vs LRU 选型）、前缀树 Trie（208 实现 + 211 通配符 DFS + 敏感词过滤实战、Trie vs 哈希表、双数组/压缩 Trie 内存优化、AC 自动机）、海量数据 TopK（40 亿整数 Hash 分桶 + 桶内小顶堆、2-bit 位图 1GB 精确计数、桶数估算与数据倾斜处理）、排序链表（148 自底向上归并 O(1) 空间完整实现、链表为何不用快排/堆排、稳定性）、数组中第 K 大（215 随机化快速选择平均 O(n) 推导、三路划分应对大量重复、小顶堆 O(n log K) 选型对比）、Dijkstra 单源最短路（743 邻接表 + 小顶堆优化 O((V+E) log V)、懒删除无需 decrease-key、为何不能有负权边、与 BFS/A* 关系、多源与 Floyd 全源）、二叉树序列化与反序列化（297 前序 + nil 占位为何必要、共享游标还原、strings.Builder 防 O(n²)、二进制/Protobuf 生产选型）、数组中的逆序对（剑指 Offer 51 归并分治统计 mid-i、int64 防溢出、树状数组替代解法）、KMP（28 next 前缀函数定义与摊还 O(n+m) 证明、strings.Index 实际用 Rabin-Karp），共新增 9 题，模块题数 90 → 99，全库 450 → 459 |
| 2026-09-16 | v5.22 | 网络协议模块新增 8 篇：HTTP 缓存机制（强缓存 vs 协商缓存、Cache-Control/no-cache 与 no-store 辨析、ETag vs Last-Modified、Vary 头防缓存串味、hash 文件名 + index.html no-cache 失效方案、Go 协商缓存中间件）、CORS 跨域（同源三要素、简单请求 vs 预检、带 Cookie 时不能用 `*`、Expose-Headers、Vary: Origin 踩坑、CORS 中间件必须早于鉴权、Nginx 统一处理）、Nagle 算法与延迟确认（40ms 毛刺完整时序还原、Go 默认 NoDelay=true、net.Buffers 合并写、tcpdump+strace 排查法）、TCP 半连接/全连接队列与 SYN Flood（两队列三参数、somaxconn 才是天花板上限、Go 内部 backlog=somaxconn、队列溢出的两种表现、SYN Cookie 原理与代价、nf_conntrack 表满）、HTTP 502/504/499 排查（三层状态码归因、超时逐层递减预算、WriteTimeout 含 handler 时间的坑、Nginx error log 高频短语对照、499 与幂等、K8s preStop 优雅下线）、四层/七层负载均衡（LVS NAT/DR/TUN 三模式、DR 的 lo 配 VIP + 抑制 ARP、一致性哈希与虚拟节点、健康检查误判、keepalive 三配置缺一不可、X-Forwarded-For 可伪造）、HTTP Range 与断点续传（206/416/If-Range、http.ServeContent、分片上传三阶段、流式合并防 OOM、秒传与归属复制安全、Nginx slice 模块）、网络排查工具箱（ss/nstat/mtr/tcpdump/iftop 完整用法与阈值、CLOSE_WAIT 是应用 bug、curl -w 耗时分解定位网络还是应用），共新增 8 篇；并修正模块 06 题数统计错误（根 README 14 → 25，此前与模块 README 的 17 不一致） |
| 2026-09-13 | v5.21 | 系统设计模块新增 6 篇业务系统设计题：订单系统（状态机 + 版本号 CAS、RocketMQ 延时消息/时间轮/Redis ZSet 四种超时关单方案对比 + 扫表兜底、订单号日期+分片+序列与基因法、user_id 分库分表）、库存系统（条件更新/乐观锁/悲观锁三种 DB 扣减、Redis Lua 原子预扣、热点库存 Key 分段 10 片、幂等回补、库存流水对账）、点赞系统（Set vs Bitmap vs HyperLogLog 去重选型、计数 16 分片 + Pipeline 聚合读、INSERT IGNORE 幂等落库、DB 对账校准）、消息推送系统（Redis 路由表 TTL+心跳续期防幽灵路由、Redis Stream 离线游标拉取、APNs/FCM/厂商通道统一调度与降级、去重/聚合/频控/静默时段 + P0-P3 优先级分级）、弹幕系统（业务取舍：允许丢弃/弱一致/强时效、内存环形缓冲 + 每秒批量落库、按 roomID 一致性哈希、offset 索引历史回放、分级降级）、优惠券系统（券模板/实例模型、Redis Lua + DB 条件更新 + 唯一索引三重防线、设备指纹防刷、核销状态机 CAS、超时回滚取舍、对账 SQL），共新增 6 篇
| 2026-09-12 | v5.20 | 微服务工程模块新增 6 题：服务注册与发现（客户端 vs 服务端发现、CP vs AP 选型、etcd 租约 + KeepAlive + Watch 完整 Go 实现、注册中心故障排查）、分布式配置中心（推 vs 拉 vs 长轮询、Go 长轮询客户端 + atomic 热更新 + 本地快照兜底、灰度发布与回滚、五大常见坑）、服务网格 Service Mesh（数据面 vs 控制面、Envoy/xDS、Mesh vs SDK 选型矩阵、Sidecar 启动竞态与 Ambient 模式）、Go 微服务框架选型（go-zero vs Kratos vs Kitex 六维对比、选型四准则、洋葱模型中间件链顺序）、分布式链路追踪与采样（W3C traceparent / B3、OpenTelemetry Go 实现、头部采样 vs 尾部采样组合策略、goroutine/MQ 断链五大坑），共新增 6 题 |
| 2026-09-10 | v5.18 | 数据库模块新增 6 题：MySQL AUTO_INCREMENT 高并发锁竞争与 innodb_autoinc_lock_mode 优化方案、布隆过滤器防缓存穿透原理与 Go 实现、读写分离延迟补偿策略（Session Stickiness + 强制主库读取 + binlog 延迟监控）、乐观锁 CAS vs 悲观锁 FOR UPDATE Go 实战选型决策树、缓存与数据库双写一致性深度解析（Cache Aside / Delayed Dual Delete / Canal + Binlog）、Redis 数据结构选型指南（String vs Hash vs List vs Set vs ZSet） |
| 2026-09-11 | v5.19 | 分布式系统模块新增 4 篇：端到端消息交付语义（at-most-once/at-least-once/exactly-once 三大语义详解 + Kafka/ RocketMQ/RabbitMQ 实现对比 + 幂等性 Go 代码实战）、重试策略深度剖析（指数退避原理/Full Jitter 防惊群效应/Go backoff 包用法/预算控制设计）、API Gateway 架构设计（五大职责边界 + Chi 手写完整实现 + 动态路由 + Sidecar vs Library 选型矩阵）、CQRS vs Event Sourcing（正交概念辨析/事件重放原理/快照优化/Golang 生态实践 + 何时不用），共新增 4 篇
| 2026-09-08 | v5.17 | Linux / 操作系统模块新增 6 题：Swap 空间与 OOM Killer 评分机制（go memlimit/GOMEMLIMIT 配合策略 + kswapd/direct reclaim）、core dump 排障实战（coredumpctl/core_pattern/gdb+go 符号表解析/信号处理器配置）、TCP 高并发调优（SYN Cookie/SO_REUSEPORT 多进程端口共享/Backlog somaxconn 关系）/proc & /sys 文件系统深度（status/smaps/cpuset cgroup/THP 状态监控），共新增 6 题
| 2026-09-03 | v5.15 | 项目实战问题模块新增 6 题：goroutine 泄漏排查与根治（pprof goroutine profile + select+ctx.Done 退出条件 + Timer 未 Stop 隐蔽泄漏）、HTTP Response.Body 泄漏导致连接耗尽（resp.Body.Close() 标准模板 + http.Client Transport 配置）、文件句柄泄漏导致 EMFILE 崩溃（defer close 规范 + os.ReadFile 替代手动 open/close）、Data Race 竞态条件（go test -race + atomic.RWMutex + sync.Map 选型决策树）、panic/recover 误用分析（初始化 panic vs error 返回规范 + Sentry 监控集成）、TCP TIME_WAIT 过多导致端口耗尽（http.Transport 连接复用 + Keep-Alive + sysctl 调优）
| 2026-08-29 | v5.12 | 系统设计模块新增 6 篇：缓存与数据库一致性策略（Cache-Aside/Write-Through/延迟双删/Canal实时同步）+ CDN 架构设计（PUSH/PULL模式/回源穿透防护/缓存失效）+ API 网关设计（JWT鉴权/动态路由/灰度发布/响应聚合）+ WebSocket 长连接架构（心跳保活/百万并发水平扩展/离线消息）+ 大文件上传系统（分片上传/秒传/断点续传/预签名URL）+ 深度分页优化（游标分页/延迟关联/覆盖索引），共新增 6 题
| 2026-08-25 | v5.9 | Go 语言深度模块新增 4 篇：Race Detector 内部原理（编译期插桩/Clock Vector/Happens-Before 算法实现）、Channel 安全排空模式（for range vs select+default/goroutine 泄漏防护/完整 Worker Pool 示例）、值接收者 vs 指针接收者（Method Set 规则/一致性原则/接口满足条件）、Interface Nil 陷阱（typed nil vs untyped nil/interface 内存布局/返回技巧），共新增 4 题
| 2026-08-21 | v5.8 | Go 语言深度模块新增 7 题：sync.SingleFlight 缓存击穿解决方案、nil vs closed channel 语义差异（goroutine 阻塞根源）、defer + named return 执行顺序细节、context.Context 传播与超时控制最佳实践、errors.Is/As 错误链处理、strings.Builder 零拷贝构建、http.Client 连接池与超时配置
| 2026-08-20 | v5.7 | 面试策略模块新增 4 篇：自我介绍（三段式万能模板+Go社招/校招差异化写法+埋钩子引导提问+反问环节高质量清单）、系统设计面试（五步法框架+短链/限流器/秒杀真题Go实现）、Live Coding应对策略（解题五步法+Go并发常见坑题goroutine泄漏/goroutine闭包/defer顺序）、面试全流程节奏控制（多面连面策略+每日复盘表+目标公司分类管理+Offer评估评分卡），共新增 6 题
| 2026-08-14 | v5.5 | 微服务工程模块新增 6 篇：服务降级与舱壁隔离（降级开关/线程池 vs 信号量隔离）、无损发布（readiness 摘流 + preStop + SIGTERM 优雅关闭 + 长连接摘流）、微服务拆分原则（DDD 限界上下文/康威定律/绞杀者模式）、接口幂等设计（唯一约束/去重表/Token/状态机）、可观测性三支柱整合（OpenTelemetry + TraceID 贯穿 + eBPF 零侵入）、BFF 模式（聚合/裁剪/多端适配）；并修正根 README 总题数统计错误（313 → 328） |
| 2026-08-13 | v5.4 | 分布式系统模块新增 3 篇：限流算法/熔断器/重试策略（令牌桶 vs 漏桶 vs 滑动窗口 + 三态机熔断 + 指数退避）、gRPC 基础（HTTP/2 多路复用 + Protobuf + 四种 RPC 流式模式 + interceptor）、ZooKeeper 原理（ZAB 协议 + Session + Watch + EPHEMERAL 节点），共新增 6 题 |
| 2026-08-12 | v5.3 | 数据库模块新增 5 题：MySQL 三大日志与两阶段提交、一条 SQL 执行流程、InnoDB Buffer Pool、主键选择（自增 vs UUID vs 雪花）、Redis 主从复制 psync 原理 |
| 2026-08-12 | v5.2 | Go 语言深度模块新增 4 题：False Sharing（CPU 缓存行对齐与 padding）、nil vs Closed Channel 行为差异、sync.Pool 最佳实践、Interface 组合 vs 嵌入（Method Set / 隐式实现） |
| 2026-08-12 | v5.1 | Go 语言深度模块新增 4 题：Channel vs Mutex、Panic/Recover、Context Value 陷阱、函数选项模式 |
| 2026-08-10 | v5.0 | 全仓库 README 重构：模块导航表格化、链接全部修复、按高赞项目结构重写 |
| 2026-08-10 | v4.10 | 全仓库 README 格式化：所有题目改为可点击跳转链接（266 个链接），重建 06-network / 10-real-problems / 11-go-std-practice 索引 |
| 2026-08-10 | v4.9 | Linux/IPC 模块大更新 4 题：eventfd、fork COW、seccomp、futex |
| 2026-08-10 | v4.8 | 新增《nil slice vs empty slice》 |
| 2026-05-14 | v4.3 | 新增《GC Pacer》深度解析 |

---

## 🤝 贡献指南

欢迎 PR 补充内容！提交前请确认：

- [ ] 放到正确的子目录，文件名格式 `NN-kebab-case.md`
- [ ] 遵循统一的五段式内容结构（考察意图 → 核心答案 → 深度展开 → 高频追问 → 延伸阅读）
- [ ] 结合生产经验，避免纯理论堆砌
- [ ] 关键结论有数据或源码支撑
- [ ] 代码块注明语言类型，Go 代码可直接运行

**[提 Issue](https://github.com/guocong-bincai/go-interview-guide/issues)** · **[提 PR](https://github.com/guocong-bincai/go-interview-guide/pulls)**

---

## ⭐ Star 趋势

如果本仓库对你有帮助，欢迎点个 Star 🌟

[![Star History Chart](https://api.star-history.com/svg?repos=guocong-bincai/go-interview-guide&type=Date)](https://star-history.com/#guocong-bincai/go-interview-guide&Date)

---

<div align="center">

**📖 持续更新中 · 建议 Watch 仓库获取最新动态**

Made with ❤️ for Go Engineers

</div>
