<div align="center">

# 🏆 Go 高级工程师面试宝典

**专为 5~8 年 Go 后端工程师打造 · 大厂面试核心考点全覆盖**

[![Stars](https://img.shields.io/github/stars/guocong-bincai/go-interview-guide?style=flat-square&logo=github&color=yellow)](https://github.com/guocong-bincai/go-interview-guide/stargazers)
[![Forks](https://img.shields.io/github/forks/guocong-bincai/go-interview-guide?style=flat-square&logo=github&color=blue)](https://github.com/guocong-bincai/go-interview-guide/network/members)
[![License](https://img.shields.io/github/license/guocong-bincai/go-interview-guide?style=flat-square&color=green)](./LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](https://github.com/guocong-bincai/go-interview-guide/pulls)
[![文章数量](https://img.shields.io/badge/文章-136-orange?style=flat-square)](./docs)
[![版本](https://img.shields.io/badge/版本-v2.28-blue?style=flat-square)](./docs)

<br/>

> 不是入门教程，不是题目合集。
> 每个知识点直击面试官考察意图，给出**能拿到 Offer 的回答**。

<br/>

[📖 开始阅读](#-知识地图) · [⭐ 收藏仓库](https://github.com/guocong-bincai/go-interview-guide) · [🤝 参与贡献](#-参与贡献)

</div>

---

## 🎯 这个仓库适合你吗？

✅ **适合：**
- 工作 **5~8 年**，准备跳槽大厂 / 独角兽的 Go 后端工程师
- 已有经验，想**系统梳理**知识体系、查漏补缺
- 准备**技术晋升答辩**，需要深度输出

❌ **不适合：**
- Go 初学者（请先完成语言基础学习）
- 纯刷题备战（配合 LeetCode 使用，本仓库不是题目合集）

---

## 💡 和其他面经的区别

| | 本仓库 | 普通面经 |
|---|---|---|
| 内容深度 | 讲清楚「为什么这样设计」| 只告诉你「是什么」|
| 生产视角 | 结合真实线上场景和踩坑 | 纯理论堆砌 |
| 数据支撑 | 给出可量化指标（QPS / 延迟 / 内存）| 模糊描述 |
| 代码示例 | 可运行的 Go 代码 + 生产级写法 | 伪代码或无代码 |
| 追问覆盖 | 覆盖面试官的下一个问题 | 点到为止 |

---

## 🚨 贡献规范（新增内容必读）

> 这部分写给所有参与补充内容的同学。**不符合规范的 PR 会被打回。**

### 内容定位

这个仓库面向两类读者：

- **Go 后端工程师（5~8 年）**：深度原理 + 生产经验 + 面试追问，Go 代码为主
- **算法模块（所有人）**：通俗易懂讲解，覆盖高频题（简单/中等），Go + Python 双语实现

写每篇文章前先问自己：**"这是高频考点吗？读者能直接用上吗？"**

### 语言规则

| 模块 | 代码语言 |
|------|---------|
| 01 Go 语言 / 02 数据库 / 03 分布式 / 04 微服务 / 05 系统设计 / 06 网络 / 08 工程素养 | **仅 Go** |
| **07 算法** | **Go + Python 双语**（每道题都要给两种实现） |
| 09 面试策略 | 无代码 |

### 内容质量要求

**Go 后端模块（01~06, 08）必须做到：**
- 讲清楚底层原理，不只是罗列功能点
- 结合生产经验：踩过哪些坑、线上出过什么问题、怎么排查
- 给出可量化数据：QPS、延迟 P99、内存占用、优化前后对比
- Go 代码示例可直接运行，生产级写法
- 覆盖追问：面试官的下一个问题提前写进去

**算法模块（07）必须做到：**
- **通俗讲解**：用大白话说清楚思路，类比生活场景，让没见过这类题的人也能看懂
- **图示或步骤拆解**：复杂题必须有逐步推导，不能上来就贴代码
- **Go + Python 双语**：两种语言都要有完整可运行的代码
- **复杂度分析**：时间复杂度 + 空间复杂度必写，说明为什么
- **高频优先**：优先补充高频题（★★★★以上），简单题也可以写，不用每题都很深

**禁止出现：**

| 禁止内容 | 原因 |
|---------|------|
| 未发布的 Go 版本特性（如 Go 1.26+） | 面试考不到，写了是误导 |
| 算法题只贴代码、没有思路讲解 | 对读者没有帮助 |
| 复制粘贴官方文档 | 没有附加价值 |
| 没有代码示例的纯文字理论 | 不可信，面试时说不清楚 |
| 与 Go 后端完全无关的内容（前端、移动端） | 定位不符 |

### 文章结构模板

**每篇文章必须遵循以下五段式结构：**

```markdown
## 面试官考察意图
这道题想考什么，考到什么深度，高级工程师的回答和初级工程师的回答有什么差距。

## 核心答案（30 秒版）
开门见山的简短回答，适合面试时的第一句话。可以是一张对比表格或几行结论。

## 深度展开
原理 → 实现细节（源码/汇编层面） → 生产经验/踩坑案例 → 与竞品的横向对比

## 高频追问
面试官的下一个问题，逐一给出参考答案。

## 延伸阅读
官方文档链接、论文、知名博客（链接必须能打开）。
```

### 文件存放规则

- 文件放到正确的目录，不要直接放在模块根目录下（有子目录就放子目录）
- 文件名格式：`NN-kebab-case.md`，编号从 01 开始，不能与现有文件编号冲突
- 图片放在同级 `assets/` 目录，引用用相对路径

### 优先补充的内容（P0 空缺）

以下是当前仍需补充的高优先级内容，按需认领：

| 模块 | 缺少的文章 | 优先级 | 状态 |
|------|----------|--------|------|
| 01-golang/01-runtime | Go 调度器源码走读（runtime/proc.go） | P0 | ✅ 已完成（2026-04）|
| 02-database/01-mysql | EXPLAIN 输出字段逐一解读 + 实战案例 | P0 | ✅ 已完成（2026-04）|
| 03-distributed/04-service-mesh | Service Mesh（Istio/Envoy）原理 | P1 | ✅ 已完成（v2.28）|
| 01-golang/05-stdlib | net/http 深度解析 | P1 | ✅ 已完成（2026-04）|
| 01-golang/05-stdlib | sync.Map 与并发安全 Map | P1 | ✅ 已完成（2026-04）|
| 01-golang/05-stdlib | 错误处理最佳实践 | P1 | ✅ 已完成（2026-04）|
| 03-distributed/05-coordination | etcd 原理与实战 | P1 | ✅ 已完成（v2.27）|
| 03-distributed/05-coordination | 配置中心选型与实践 | P2 | ✅ 已完成（v2.27）|
| 02-database/04-tidb | TiDB 架构与适用场景 | P2 | ✅ 已完成（v2.27）|
| 06-network/03-security | Web 安全：常见攻击与防御 | P1 | ✅ 已完成（2026-04）|
| 07-algorithms/09-graph | 图论高频题（DFS/BFS/并查集） | P2 | ✅ 已完成（v2.28）|
| 10-real-problems/05 | 数据迁移与重构问题 | P1 | ✅ 已完成（2026-04）|
| 10-real-problems/06 | 并发编程实战问题 | P0 | ✅ 已完成（2026-04）|
| 10-real-problems/07 | 面试高频场景题（开放性） | P0 | ✅ 已完成（2026-04）|
| ~~08-engineering/02-troubleshooting~~ | ~~goroutine 泄漏排查 SOP~~ | ~~P0~~ | ✅ 已完成 |

---

## 🗺️ 知识地图

### ⭐ 优先级说明

> 备考时间紧张，按 **P0 → P1 → P2** 顺序攻克

| 标记 | 含义 | 建议投入 |
|------|------|----------|
| 🔴 P0 | 大厂必考，逢面必问 | 优先搞定 |
| 🟡 P1 | 高频考点，有一定深度 | 次优先 |
| 🟢 P2 | 加分项，有余力再看 | 拔高用 |

---

### 📦 01 · Go 语言深度 🔴 P0

<details>
<summary><b>01-runtime · 运行时原理</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🔴 [GMP 调度模型](./docs/01-golang/01-runtime/01-gmp.md) | G/M/P 角色、Work Stealing、Hand Off、sysmon 抢占 |
| 🔴 [GC 垃圾回收机制](./docs/01-golang/01-runtime/02-gc.md) | 三色标记、混合写屏障、STW 优化历程、GOGC 调优 |
| 🔴 [Go 调度器源码走读](./docs/01-golang/01-runtime/05-scheduler-source-code.md) | schedule/findRunnable/execute 源码解读、g0 栈、Hand Off 细节、sysmon、netpoller |
| 🟡 [内存分配器](./docs/01-golang/01-runtime/03-memory-alloc.md) | tcmalloc、mspan、mcache/mcentral/mheap |
| 🟡 [goroutine 栈机制](./docs/01-golang/01-runtime/04-stack.md) | 动态栈增长/收缩、连续栈 vs 分段栈 |

</details>

<details>
<summary><b>02-concurrency · 并发编程</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🔴 [channel 底层原理](./docs/01-golang/02-concurrency/01-channel.md) | hchan 结构、发送/接收流程、select 实现、死锁排查 |
| 🔴 [sync 原语](./docs/01-golang/02-concurrency/02-sync.md) | Mutex/RWMutex 实现、sync.Once、sync.Pool |
| 🟡 [atomic 与无锁](./docs/01-golang/02-concurrency/03-atomic.md) | CAS 原理、atomic 操作、无锁数据结构 |
| 🟡 [并发模式](./docs/01-golang/02-concurrency/04-patterns.md) | Pipeline、Fan-out/Fan-in、errgroup、Worker Pool |
| 🟡 [context 原理](./docs/01-golang/02-concurrency/05-context.md) | 取消传播、超时控制、底层实现 |

</details>

<details>
<summary><b>03-language-deep · 语言机制</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🔴 [interface 原理](./docs/01-golang/03-language-deep/01-interface.md) | iface/eface 内存布局、动态分发、nil 陷阱 |
| 🟡 [reflect 原理](./docs/01-golang/03-language-deep/02-reflect.md) | 性能代价、实际使用场景 |
| 🟡 [泛型实现](./docs/01-golang/03-language-deep/03-generics.md) | GCShape stenciling、使用边界 |
| 🟡 [逃逸分析](./docs/01-golang/03-language-deep/04-escape.md) | 堆 vs 栈分配、如何避免不必要逃逸 |
| 🟡 [slice 与 map](./docs/01-golang/03-language-deep/05-slice-map.md) | 底层结构、扩容策略、并发安全问题 |
| 🟡 [内存模型](./docs/01-golang/03-language-deep/06-memory-model.md) | happens-before、内存对齐、false sharing |
| 🟢 [循环与迭代器新特性](./docs/01-golang/03-language-deep/07-loop-iterators.md) | Go 1.22 循环变量语义变更、range-over-func |

</details>

<details>
<summary><b>04-performance · 性能调优</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🔴 [pprof 实战](./docs/01-golang/04-performance/01-pprof.md) | CPU/内存/goroutine 火焰图分析 |
| 🟡 [内存泄漏排查](./docs/01-golang/04-performance/02-memory-leak.md) | goroutine 泄漏、全局变量、缓存失控 |
| 🟡 [基准测试规范](./docs/01-golang/04-performance/03-benchmark.md) | benchmark 写法、避免编译器优化干扰 |
| 🟡 [真实调优案例](./docs/01-golang/04-performance/04-tuning-cases.md) | JSON 解析、字符串拼接、sync.Pool 实战 |

</details>

<details>
<summary><b>05-stdlib · 标准库与工程实践</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [net/http 深度解析](./docs/01-golang/05-stdlib/01-net-http.md) | Server 底层结构、Transport 连接池、优雅关闭、中间件链 |
| 🟡 [sync.Map 与并发安全 Map](./docs/01-golang/05-stdlib/02-sync-map.md) | 双 map 设计、读写流程、与 RWMutex+map 性能对比 |
| 🟡 [错误处理最佳实践](./docs/01-golang/05-stdlib/03-errors.md) | errors.Is/As/Unwrap、%w 包装、panic/recover 边界 |

</details>

---

### 🗄️ 02 · 数据库 🔴 P0

<details>
<summary><b>01-mysql · MySQL</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🔴 [索引原理与优化](./docs/02-database/01-mysql/01-index.md) | B+ 树、聚簇/二级索引、联合索引、索引失效全场景、EXPLAIN |
| 🔴 [事务、隔离级别与 MVCC](./docs/02-database/01-mysql/02-transaction.md) | ACID、ReadView、版本链、RC vs RR、间隙锁、死锁 |
| 🔴 [锁机制深度](./docs/02-database/01-mysql/03-lock.md) | 行锁/表锁/间隙锁/临键锁、死锁检测与避免 |
| 🟡 [慢查询优化](./docs/02-database/01-mysql/04-slow-query.md) | EXPLAIN 解读、SQL 改写、深分页优化 |
| 🔴 [EXPLAIN 输出字段逐一解读](./docs/02-database/01-mysql/07-explain.md) | 12 字段详解、Using filesort/temporary 优化、实战案例、EXPLAIN ANALYZE |
| 🟡 [分库分表](./docs/02-database/01-mysql/05-sharding.md) | ShardingSphere、路由策略、数据迁移 |
| 🟡 [主从复制](./docs/02-database/01-mysql/06-replication.md) | binlog、半同步复制、主从延迟处理 |

</details>

<details>
<summary><b>02-redis · Redis</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🔴 [数据结构底层实现](./docs/02-database/02-redis/01-data-structures.md) | SDS、ziplist、skiplist、listpack、各类型应用场景 |
| 🔴 [持久化机制](./docs/02-database/02-redis/02-persistence.md) | RDB vs AOF、混合持久化、数据恢复流程 |
| 🔴 [缓存穿透/击穿/雪崩](./docs/02-database/02-redis/03-cache-problems.md) | 布隆过滤器、互斥锁、逻辑过期、多级缓存 |
| 🟡 [集群方案](./docs/02-database/02-redis/04-cluster.md) | Sentinel vs Cluster、槽位分配、故障转移 |
| 🟡 [分布式锁](./docs/02-database/02-redis/05-distributed-lock.md) | Redlock 算法、Lua 脚本原子性、锁续期 |
| 🟡 [热 key / 大 key](./docs/02-database/02-redis/06-hot-key.md) | 识别方法、拆分方案、本地缓存兜底 |

</details>

<details>
<summary><b>03-elasticsearch · Elasticsearch</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟢 [倒排索引原理](./docs/02-database/03-elasticsearch/01-inverted-index.md) | 分词、相关性评分、BM25 |
| 🟢 [查询优化](./docs/02-database/03-elasticsearch/02-query-optimization.md) | mapping 设计、冷热数据分离 |

</details>

<details>
<summary><b>04-tidb · TiDB</b>（点击展开）⚠️ 待补充</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟢 [TiDB 架构与适用场景](./docs/02-database/04-tidb/01-tidb-architecture.md) | TiDB/TiKV/PD 三层架构、HTAP、分布式事务 Percolator、与 MySQL 分库分表对比 |

</details>

---

### 🌐 03 · 分布式系统 🟡 P1

<details>
<summary><b>01-theory · 理论基础</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [CAP 与 BASE](./docs/03-distributed/01-theory/01-cap-base.md) | CAP 定理、BASE 理论、实际系统取舍 |
| 🟡 [Raft 协议](./docs/03-distributed/01-theory/02-raft.md) | Leader 选举、日志复制、成员变更 |
| 🟢 [Paxos 简述](./docs/03-distributed/01-theory/03-paxos.md) | 与 Raft 的对比、应用场景 |
| 🟡 [一致性模型](./docs/03-distributed/01-theory/04-consistency.md) | 强一致 / 最终一致 / 线性一致性 |

</details>

<details>
<summary><b>02-transactions · 分布式事务</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [2PC / 3PC](./docs/03-distributed/02-transactions/01-2pc-3pc.md) | 原理、问题与局限 |
| 🟡 [TCC 模式](./docs/03-distributed/02-transactions/02-tcc.md) | Try/Confirm/Cancel、幂等设计、空回滚、悬挂问题 |
| 🟡 [Saga 模式](./docs/03-distributed/02-transactions/03-saga.md) | 编排 vs 协调、补偿事务设计 |
| 🟡 [消息最终一致性](./docs/03-distributed/02-transactions/04-msg-eventual.md) | 本地消息表、事务消息（RocketMQ）|
| 🟢 [Seata 实战](./docs/03-distributed/02-transactions/05-seata.md) | AT/TCC/Saga 模式选型 |

</details>

<details>
<summary><b>03-mq · 消息队列</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [Kafka 高吞吐原理](./docs/03-distributed/03-mq/01-kafka-principle.md) | 零拷贝、顺序写、分区、Page Cache |
| 🟡 [Kafka 消息可靠性](./docs/03-distributed/03-mq/02-kafka-reliability.md) | ACK 机制、ISR、幂等生产者 |
| 🟡 [消费者组与 Rebalance](./docs/03-distributed/03-mq/03-kafka-consumer.md) | 消费者组、Rebalance 流程、顺序消费 |
| 🟡 [RocketMQ vs Kafka](./docs/03-distributed/03-mq/04-rocketmq.md) | 对比、事务消息、延迟消息 |
| 🔴 [消息积压/丢失/重复](./docs/03-distributed/03-mq/05-mq-problems.md) | 消息积压处理、exactly-once 语义 |
| 🟢 [RabbitMQ](./docs/03-distributed/03-mq/06-rabbitmq.md) | AMQP、Exchange 类型、死信队列 |

</details>

<details>
<summary><b>04-service-mesh · 服务治理</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [注册中心](./docs/03-distributed/04-service-mesh/01-service-discovery.md) | Consul/etcd/Nacos 原理与选型 |
| 🟡 [Service Mesh 原理](./docs/03-distributed/04-service-mesh/02-service-mesh.md) | Istio/Envoy、Sidecar 注入、流量管理、mTLS、熔断 |
| 🟡 [链路追踪](./docs/03-distributed/04-service-mesh/03-tracing.md) | Jaeger/Zipkin、TraceID 传播、采样策略 |

</details>

<details>
<summary><b>05-coordination · 协调服务</b>（点击展开）⚠️ 待补充</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [etcd 原理与实战](./docs/03-distributed/05-coordination/01-etcd.md) | Raft + boltdb + MVCC、Watch 机制、分布式锁、Leader 选举 |
| 🟢 [配置中心选型与实践](./docs/03-distributed/05-coordination/02-config-center.md) | Apollo vs Nacos vs etcd、热更新原理、配置灰度、Go 接入 |

</details>

---

### 🔧 04 · 微服务工程 🟡 P1

<details>
<summary><b>01-rpc · RPC 与服务治理</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [gRPC 原理](./docs/04-microservices/01-rpc/01-grpc.md) | Protobuf 编码、HTTP/2 多路复用、流式 RPC |
| 🟡 [熔断与限流](./docs/04-microservices/01-rpc/02-circuit-breaker.md) | Hystrix/Sentinel、令牌桶/漏桶算法 |
| 🟡 [服务治理](./docs/04-microservices/01-rpc/03-service-governance.md) | 超时/重试/负载均衡策略 |
| 🟢 [IDL 设计规范](./docs/04-microservices/01-rpc/04-idl-design.md) | Protobuf 版本兼容、API 设计规范 |

</details>

<details>
<summary><b>02-api-gateway · API 网关</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [网关设计](./docs/04-microservices/02-api-gateway/01-gateway-design.md) | 路由/鉴权/限流/灰度发布 |
| 🟡 [认证鉴权](./docs/04-microservices/02-api-gateway/02-auth.md) | JWT/OAuth2/API Key、Token 刷新 |
| 🟡 [限流实现](./docs/04-microservices/02-api-gateway/03-rate-limit.md) | 网关层限流、分布式限流（Redis + Lua）|

</details>

<details>
<summary><b>03-observability · 可观测性</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [Metrics 监控](./docs/04-microservices/03-observability/01-metrics.md) | Prometheus + Grafana、RED 指标、SLO/SLA |
| 🟡 [日志体系](./docs/04-microservices/03-observability/02-logging.md) | 结构化日志、ELK 方案、日志采样 |
| 🟡 [告警设计](./docs/04-microservices/03-observability/03-alerting.md) | 告警规则、告警疲劳治理 |

</details>

<details>
<summary><b>04-deployment · 部署与发布</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟢 [Docker 实践](./docs/04-microservices/04-deployment/01-docker.md) | 多阶段构建、镜像优化、资源限制 |
| 🟢 [Kubernetes 核心](./docs/04-microservices/04-deployment/02-kubernetes.md) | Pod 调度、HPA、滚动发布、Service Mesh |
| 🟢 [CI/CD 流水线](./docs/04-microservices/03-cicd/03-cicd.md) | 蓝绿部署、金丝雀发布 |

</details>

---

### 🏗️ 05 · 系统设计 🔴 P0

<details>
<summary><b>架构模式</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [CQRS 模式](./docs/05-system-design/01-patterns/01-cqrs.md) | 读写分离、Event Sourcing 结合 |
| 🟡 [事件驱动架构](./docs/05-system-design/01-patterns/02-event-driven.md) | Outbox Pattern、事件溯源 |
| 🟢 [DDD 战略设计](./docs/05-system-design/01-patterns/03-ddd.md) | 领域、限界上下文、聚合根 |
| 🟡 [Saga 落地](./docs/05-system-design/02-scenarios/01-saga.md) | 编排型 vs 协调型、补偿事务设计 |
| 🟡 [IM 系统](./docs/05-system-design/02-scenarios/02-im.md) | 消息投递、离线消息、已读未读 |

</details>

<details>
<summary><b>高频设计题</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🔴 [秒杀系统](./docs/05-system-design/01-seckill/01-seckill.md) | 预减库存、异步下单、防超卖、流量漏斗 |
| 🔴 [短链系统](./docs/05-system-design/02-short-url/02-short-url.md) | 发号器、跳转、高可用 |
| 🟡 [Feed 流](./docs/05-system-design/04-feed/04-feed.md) | 推模式 vs 拉模式 vs 推拉结合 |
| 🔴 [分布式 ID](./docs/05-system-design/05-distributed-id/05-distributed-id.md) | Snowflake、Leaf、UUIDv7 对比 |
| 🔴 [限流系统](./docs/05-system-design/06-rate-limiter/06-rate-limiter.md) | 令牌桶、滑动窗口、分布式限流 |
| 🟡 [搜索系统](./docs/05-system-design/07-search/07-search.md) | 倒排索引、分词、搜索建议 |
| 🟡 [支付系统](./docs/05-system-design/08-payment/08-payment.md) | 幂等、对账、资金安全 |

</details>

<details>
<summary><b>容量估算</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [信封估算法](./docs/05-system-design/03-capacity/01-back-of-envelope.md) | QPS / 存储 / 带宽快速估算 |
| 🟡 [性能指标体系](./docs/05-system-design/03-capacity/02-performance-indicators.md) | P99/P999、可用性 SLA |

</details>

---

### 🌍 06 · 网络协议 🟡 P1

<details>
<summary><b>01-tcp-ip · TCP/IP</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [TCP 三次握手/四次挥手](./docs/06-network/01-tcp-ip/01-tcp-handshake.md) | 状态机、TIME_WAIT 问题、连接复用 |
| 🟡 [流量控制与拥塞控制](./docs/06-network/01-tcp-ip/02-tcp-flow.md) | 滑动窗口、CUBIC、BBR |
| 🟡 [粘包与拆包](./docs/06-network/01-tcp-ip/03-tcp-sticky.md) | 原因与解决方案 |
| 🟢 [TCP Keepalive](./docs/06-network/01-tcp-ip/04-tcp-keepalive.md) | TCP Keepalive vs 应用层心跳 |

</details>

<details>
<summary><b>02-http · HTTP / HTTPS</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [HTTP 版本演进](./docs/06-network/02-http/01-http-versions.md) | HTTP/1.1 vs HTTP/2 vs HTTP/3 核心差异 |
| 🟡 [HTTPS 握手流程](./docs/06-network/02-http/02-https.md) | TLS 握手、证书链、性能优化 |
| 🟡 [WebSocket](./docs/06-network/02-http/03-websocket.md) | 升级握手、与 HTTP 长轮询对比 |
| 🟡 [gRPC 与 HTTP/2](./docs/06-network/02-http/04-grpc-http2.md) | 多路复用、流控 |

</details>

<details>
<summary><b>03-security · Web 安全</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [常见攻击与防御](./docs/06-network/03-security/01-common-attacks.md) | SQL 注入、XSS、CSRF、SSRF、JWT 安全、敏感数据处理 |

</details>

---

### 📐 07 · 高频算法 🟡 P2

> 所有题目提供 **Go + Python 双语实现**，通俗讲解思路，适合所有阶段工程师

<details>
<summary><b>00-foundation · 高频基础题通俗讲解</b>（点击展开）</summary>

| 文章 | 包含题目 |
|------|----------|
| 🔴 [数组与哈希高频题](./docs/07-algorithms/00-foundation/01-array-hash.md) | 两数之和、三数之和、最长不重复子串、有效括号、字母异位词 |
| 🔴 [字符串高频题](./docs/07-algorithms/00-foundation/02-string.md) | 反转字符串、回文判断、最长公共前缀、字符串转整数 |
| 🟡 [双指针高频题](./docs/07-algorithms/00-foundation/03-two-pointers.md) | 移除元素、合并有序数组、接雨水、盛水最多容器 |
| 🟡 [排序与查找](./docs/07-algorithms/00-foundation/04-sort-search.md) | 快排、归并、堆排序原理，在旋转数组中搜索 |

</details>

<details>
<summary><b>核心模板（中高频进阶）</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [滑动窗口](./docs/07-algorithms/01-sliding-window/01-sliding-window.md) | 固定窗口 / 可变窗口模板 |
| 🟡 [二分查找](./docs/07-algorithms/02-binary-search/02-binary-search.md) | 左闭右开模板、变种题型 |
| 🟡 [回溯](./docs/07-algorithms/03-backtracking/03-backtracking.md) | 全排列、组合、子集、剪枝 |
| 🔴 [动态规划](./docs/07-algorithms/04-dp/04-dp.md) | 背包、LCS、LIS、编辑距离、打家劫舍系列 |
| 🟡 [链表](./docs/07-algorithms/05-linked-list/05-linked-list.md) | 反转、合并、环检测、LRU |
| 🟡 [树 DFS/BFS](./docs/07-algorithms/06-tree/06-tree.md) | 层序遍历、公共祖先、路径问题 |
| 🟡 [单调栈](./docs/07-algorithms/07-monotonic-stack/07-monotonic-stack.md) | 下一个更大元素、柱状图 |
| 🟡 [堆 / TopK](./docs/07-algorithms/08-heap/08-heap.md) | 优先队列、TopK、合并 K 个有序列表 |
| 🟢 [图论高频题](./docs/07-algorithms/09-graph/01-graph.md) | DFS/BFS 模板、拓扑排序、并查集（岛屿数量/课程表） |

</details>

<details>
<summary><b>Go 实现技巧</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟢 [Go 刷题技巧](./docs/07-algorithms/02-golang-impl/01-go-tips.md) | 排序、字符串处理、math 库 |
| 🟢 [并发算法题](./docs/07-algorithms/02-golang-impl/02-concurrency-problems.md) | 生产者消费者、交替打印 |

</details>

---

### 🛠️ 08 · 工程素养 🟡 P1

> 5-8 年工程师的核心竞争力，**区分度最高**的模块

<details>
<summary><b>线上问题排查</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🔴 [OOM 排查](./docs/08-engineering/01-oom/01-oom.md) | heap dump 分析、内存泄漏定位 |
| 🔴 [CPU 飙升排查](./docs/08-engineering/02-cpu-spike/02-cpu-spike.md) | pprof 分析、goroutine 死循环定位 |
| 🔴 [死锁排查](./docs/08-engineering/02-troubleshooting/03-deadlock.md) | 数据库死锁、Go 并发死锁、pprof 定位 |
| 🔴 [goroutine 泄漏排查 SOP](./docs/08-engineering/02-troubleshooting/06-goroutine-leak-sop.md) | 泄漏告警→profile采集→定位根因→修复验证完整闭环 |
| 🟡 [高延迟排查](./docs/08-engineering/02-troubleshooting/04-high-latency.md) | 链路追踪、GC 停顿、连接池 |
| 🟡 [goroutine 泄漏](./docs/08-engineering/05-goroutine-leak/05-goroutine-leak.md) | 识别、定位、修复模式 |

</details>

<details>
<summary><b>项目设计与演进</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [技术选型方法论](./docs/08-engineering/01-project-design/01-tech-selection.md) | 如何在面试中讲清楚为什么选 X |
| 🟡 [架构演进复盘](./docs/08-engineering/01-project-design/02-architecture-evolution.md) | 从单体到微服务的决策过程 |
| 🟡 [项目复盘模板](./docs/08-engineering/03-project-review/03-project-review.md) | 背景 / 方案 / 结果 / 反思 |

</details>

<details>
<summary><b>技术领导力</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [技术规划](./docs/08-engineering/02-tech-planning/02-tech-planning.md) | 季度/年度技术 OKR 制定 |
| 🟢 [带新人](./docs/08-engineering/03-mentoring/03-mentoring.md) | 技术传承、文档文化 |
| 🟢 [Code Review 规范](./docs/08-engineering/03-leadership/01-code-review.md) | 什么值得 block，什么只是建议 |

</details>

---

### 🔥 10 · 项目实战问题 🔴 P0

> **面试官最爱问的环节**："你在项目里遇到过什么难题？怎么解决的？"
> 这个模块专门收录生产环境中的高频问题，每个场景都给出**完整的排查思路 + 解决代码 + 面试话术**。

<details>
<summary><b>业务层高频问题</b>（点击展开）</summary>

| 文章 | 包含问题 |
|------|----------|
| 🔴 [业务层高频问题](./docs/10-real-problems/01-business-problems.md) | 超卖问题（Redis Lua/乐观锁）、重复下单/支付（幂等Token/唯一索引）、订单状态不一致（事务消息/对账）、大促流量突增（限流/预热/降级/异步）、数据库慢查询、第三方接口超时（熔断/重试） |

</details>

<details>
<summary><b>性能问题排查</b>（点击展开）</summary>

| 文章 | 包含问题 |
|------|----------|
| 🔴 [性能问题排查与优化](./docs/10-real-problems/02-performance-problems.md) | 接口 P99 突然升高（pprof/GC/链路追踪排查 SOP）、服务内存持续增长（goroutine泄漏/全局缓存）、CPU 突然打满（正则预编译/JSON优化/strings.Builder）、流量突增连接池打满、Redis 变慢（bigkey/hotkey/slowlog） |

</details>

<details>
<summary><b>数据一致性问题</b>（点击展开）</summary>

| 文章 | 包含问题 |
|------|----------|
| 🔴 [数据一致性问题](./docs/10-real-problems/03-data-consistency.md) | 缓存与DB不一致（Cache-Aside/延迟双删/binlog监听）、分布式事务部分失败（TCC实现/本地消息表）、MQ消息丢失/重复消费（ACK/幂等消费）、主从延迟读到旧数据、并发写数据覆盖（乐观锁/CAS） |

</details>

<details>
<summary><b>可用性与稳定性问题</b>（点击展开）</summary>

| 文章 | 包含问题 |
|------|----------|
| 🔴 [可用性与稳定性问题](./docs/10-real-problems/04-availability-problems.md) | 缓存雪崩（随机TTL/多级缓存/预热）、缓存击穿（互斥锁/逻辑过期）、依赖服务宕机（熔断器状态机/降级策略）、服务雪崩（超时隔离/goroutine信号量）、流量热点（一致性哈希/热点路由） |
| 🔴 [数据迁移与重构问题](./docs/10-real-problems/05-migration-problems.md) | 不停服数据库迁移（双写+灰度）、单体拆微服务（绞杀者模式）、MySQL 迁移分库分表 |
| 🔴 [并发编程实战问题](./docs/10-real-problems/06-concurrency-problems.md) | goroutine 池设计、并发安全本地缓存（分片锁/TTL）、优雅关闭（WaitGroup+信号）、errgroup/semaphore/pipeline |
| 🔴 [面试高频场景题](./docs/10-real-problems/07-interview-scenarios.md) | 高可用设计框架、技术债处理、重新设计系统、P0 故障处理 SOP、QPS 从千到万的扩容路径 |

</details>

---

### 💼 09 · 面试策略

<details>
<summary><b>行为面试与谈判</b>（点击展开）</summary>

| 文章 | 核心考点 |
|------|----------|
| 🟡 [STAR 法则](./docs/09-interview-strategy/02-behavioral/01-star-method.md) | Situation/Task/Action/Result |
| 🟡 [高频行为题](./docs/09-interview-strategy/02-behavioral/02-common-questions.md) | 冲突处理、失败经历、晋升理由 |
| 🟡 [晋升答辩](./docs/09-interview-strategy/03-promotion-interview/03-promotion-interview.md) | 结构化表达、影响力量化 |
| 🟡 [薪资谈判](./docs/09-interview-strategy/03-negotiation/01-salary-negotiation.md) | 锚点设置、竞争 Offer 利用 |
| 🟢 [Offer 对比](./docs/09-interview-strategy/02-offer-comparison/02-offer-comparison.md) | 薪资 / 成长 / 平台 / 稳定性 |

</details>

---

## 📖 每篇文章的结构

本仓库所有文章遵循统一规范，确保面试时能**结构化输出**：

```
1. 🎯 面试官考察意图   — 这道题想考什么，考到什么深度
2. ⚡ 核心答案（30秒）— 开门见山，电梯演讲式简短回答
3. 🔬 深度展开        — 原理 → 源码/实现细节 → 生产经验/踩坑 → 竞品对比
4. ❓ 高频追问        — 覆盖面试官的下一个问题，不被追问打乱节奏
5. 📚 延伸阅读        — 参考资料与源码链接
```

---

## 🤝 参与贡献

欢迎 PR 补充内容！提交前请确认：

- [ ] 放到正确的子目录，文件名格式 `NN-kebab-case.md`
- [ ] 遵循统一的五段式内容结构
- [ ] 结合生产经验，避免纯理论堆砌
- [ ] 关键结论有数据或源码支撑
- [ ] 代码块注明语言类型，Go 代码可直接运行
- [ ] 不包含未发布 Go 版本的特性

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
