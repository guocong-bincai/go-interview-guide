<div align="center">

# 🏆 Go 高级工程师面试宝典

**专为 5~8 年 Go 后端工程师打造 · 大厂面试核心考点全覆盖**

[![Stars](https://img.shields.io/github/stars/guocong-bincai/go-interview-guide?style=flat-square&logo=github&color=yellow)](https://github.com/guocong-bincai/go-interview-guide/stargazers)
[![Version](https://img.shields.io/badge/version-v1.9-blue?style=flat-square)](https://github.com/guocong-bincai/go-interview-guide/releases)
[![Forks](https://img.shields.io/github/forks/guocong-bincai/go-interview-guide?style=flat-square&logo=github&color=blue)](https://github.com/guocong-bincai/go-interview-guide/network/members)
[![License](https://img.shields.io/github/license/guocong-bincai/go-interview-guide?style=flat-square&color=green)](./LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](https://github.com/guocong-bincai/go-interview-guide/pulls)
[![文章数量](https://img.shields.io/badge/文章-持续更新中-orange?style=flat-square)](./docs)

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

## 🗺️ 知识地图

### ⭐ 优先级说明

> 备考时间紧张，按 **P0 → P1 → P2** 顺序攻克

| 标记 | 含义 | 建议投入 |
|------|------|----------|
| 🔴 P0 | 大厂必考，逢面必问 | 80% 时间 |
| 🟡 P1 | 高频考点，有一定深度 | 15% 时间 |
| 🟢 P2 | 加分项，有余力再看 | 5% 时间 |

---

### 📦 01 · Go 语言深度 🔴 P0

> GMP / GC / 并发是 Go 岗位必考三件套，任何级别都无法绕过

<details>
<summary><b>01-runtime · 运行时原理</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🔴 GMP 调度模型](./docs/01-golang/01-runtime/01-gmp.md) | G/M/P 角色、Work Stealing、Hand Off、sysmon 抢占 | ✅ 已完成 |
| [🔴 GC 垃圾回收机制](./docs/01-golang/01-runtime/02-gc.md) | 三色标记、混合写屏障、STW 优化历程、GOGC 调优、Green Tea GC | ✅ 已完成 |
| [🟡 goroutine 栈机制](./docs/01-golang/01-runtime/03-stack.md) | 动态栈增长/收缩、连续栈 vs 分段栈、Go 1.25/1.26 栈分配优化 | ✅ 已完成 |
| [🟡 Go 内存模型](./docs/01-golang/03-language-deep/03-memory-model.md) | happens-before、内存对齐、false sharing | ✅ 已完成 |

</details>

<details>
<summary><b>02-concurrency · 并发编程</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🔴 channel 底层原理](./docs/01-golang/02-concurrency/01-channel.md) | hchan 结构、发送/接收流程、select 实现 | ✅ 已完成 |
| [🔴 sync 原语](./docs/01-golang/02-concurrency/02-sync.md) | Mutex/RWMutex 实现、sync.Once、sync.Pool | ✅ 已完成 |
| [🔴 atomic 与无锁](./docs/01-golang/02-concurrency/03-atomic.md) | CAS 原理、atomic 操作、无锁数据结构 | ✅ 已完成 |
| [🟡 并发模式](./docs/01-golang/02-concurrency/04-patterns.md) | Pipeline、Fan-out/Fan-in、errgroup | ✅ 已完成 |
| [🔴 context 原理](./docs/01-golang/02-concurrency/05-context.md) | 取消传播、超时控制、底层实现 | ✅ 已完成 |

</details>

<details>
<summary><b>03-language-deep · 语言机制</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🔴 interface 原理](./docs/01-golang/03-language-deep/01-interface.md) | iface/eface 内存布局、动态分发开销 | ✅ 已完成 |
| [🟡 逃逸分析](./docs/01-golang/03-language-deep/04-escape.md) | 堆 vs 栈分配、如何避免不必要逃逸 | ✅ 已完成 |
| [🟡 slice 与 map 原理](./docs/01-golang/03-language-deep/05-slice-map.md) | 底层结构、扩容策略、Swiss Table（Go 1.24）、并发安全问题 | ✅ 已完成 |
| 🟢 泛型实现 | GCShape stenciling、使用边界 | 📝 待更新 |
| 🟢 reflect 原理 | 性能代价、实际使用场景 | 📝 待更新 |

</details>

<details>
<summary><b>04-performance · 性能调优</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🔴 pprof 实战](./docs/01-golang/04-performance/01-pprof.md) | CPU/内存/goroutine 火焰图分析 | ✅ 已完成 |
| 🟡 内存泄漏排查 | goroutine 泄漏、全局变量、缓存失控 | 📝 待更新 |
| 🟡 基准测试规范 | benchmark 写法、避免编译器优化干扰 | 📝 待更新 |
| 🟢 真实调优案例 | JSON 解析、字符串拼接、sync.Pool 实战 | 📝 待更新 |

</details>

---

### 🗄️ 02 · 数据库 🔴 P0

<details>
<summary><b>01-mysql · MySQL</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🔴 索引原理与优化](./docs/02-database/01-mysql/01-index.md) | B+ 树、聚簇/二级索引、联合索引、索引失效场景、EXPLAIN | ✅ 已完成 |
| [🔴 事务、隔离级别与 MVCC](./docs/02-database/01-mysql/02-transaction.md) | ACID、ReadView、版本链、RC vs RR、间隙锁、死锁 | ✅ 已完成 |
| [🔴 锁机制深度](./docs/02-database/01-mysql/02-transaction.md) | 行锁/表锁/间隙锁/临键锁、死锁检测 | ✅ 已完成 |
| 🟡 慢查询优化 | EXPLAIN 解读、SQL 改写、索引失效 | ✅ [已完成后](docs/02-database/01-mysql/04-slow-query.md) |
| 🟡 分库分表 | ShardingSphere、路由策略、数据迁移 | ✅ [已完成后](docs/02-database/01-mysql/05-sharding.md) |
| 🟡 主从复制 | binlog、半同步复制、延迟处理 | 📝 待更新 |

</details>

<details>
<summary><b>02-redis · Redis</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🔴 数据结构底层实现](./docs/02-database/02-redis/01-data-structures.md) | SDS、ziplist、skiplist、listpack | ✅ 已完成 |
| [🔴 持久化机制](./docs/02-database/02-redis/02-persistence.md) | RDB vs AOF、混合持久化、数据恢复 | ✅ 已完成 |
| [🔴 缓存穿透/击穿/雪崩](./docs/02-database/02-redis/03-cache-problems.md) | 布隆过滤器、互斥锁、逻辑过期、多级缓存 | ✅ 已完成 |
| [🟡 集群方案](./docs/02-database/02-redis/04-cluster.md) | Sentinel vs Cluster、槽位分配、故障转移 | ✅ 已完成 |
| [🟡 分布式锁](./docs/02-database/02-redis/04-distributed-lock.md) | Redlock 算法、Lua 脚本原子性 | ✅ 已完成 |
| [🟡 热 key / 大 key](./docs/02-database/02-redis/06-hot-key.md) | 识别方法、拆分方案、本地缓存 | ✅ 已完成 |

</details>

<details>
<summary><b>03-elasticsearch · Elasticsearch</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| 🟢 倒排索引原理 | 分词、相关性评分、BM25 | 📝 待更新 |
| 🟢 查询优化 | mapping 设计、冷热数据分离 | 📝 待更新 |

</details>

---

### 🌐 03 · 分布式系统 🟡 P1

<details>
<summary><b>01-theory · 理论基础</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🟡 CAP 与 BASE](./docs/03-distributed/01-cap-base/01-cap-base.md) | CAP 定理、BASE 理论、实际系统取舍 | ✅ 已完成 |
| [🟡 Raft 协议](./docs/03-distributed/02-raft/02-raft.md) | Leader 选举、日志复制、成员变更 | ✅ 已完成 |
| 🟢 Paxos 简述 | 与 Raft 的对比 | 📝 待更新 |

</details>

<details>
<summary><b>02-transactions · 分布式事务</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🟡 2PC / 3PC](./docs/03-distributed/02-transactions/01-2pc-3pc.md) | 原理、问题与局限 | ✅ 已完成 |
| [🟡 TCC 模式](./docs/03-distributed/02-transactions/02-tcc.md) | Try/Confirm/Cancel、幂等设计 | ✅ 已完成 |
| [🟡 Saga 模式](./docs/03-distributed/02-transactions/03-saga.md) | 编排 vs 协调、补偿事务设计 | ✅ 已完成 |
| 🟡 消息最终一致性 | 本地消息表、事务消息（RocketMQ）| 📝 待更新 |

</details>

<details>
<summary><b>03-mq · 消息队列</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🟡 Kafka 高吞吐原理](./docs/03-distributed/03-mq-problems/01-kafka-principle.md) | 零拷贝、顺序写、分区 | ✅ 已完成 |
| 🟡 Kafka 消息可靠性 | ACK 机制、ISR、幂等生产者 | 📝 待更新 |
| 🟡 消费者组与 Rebalance | 消费者组、顺序消费 | 📝 待更新 |
| [🟡 消息积压/丢失/重复](./docs/03-distributed/03-mq-problems/05-mq-problems.md) | 常见问题与解决方案 | ✅ 已完成 |

</details>

<details>
<summary><b>04-service-mesh · 服务治理</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| 🟡 注册中心 | Consul/etcd/Nacos 原理与选型 | 📝 待更新 |
| [🟡 熔断与限流](./docs/04-microservices/02-circuit-breaker/02-circuit-breaker.md) | Hystrix/Sentinel、令牌桶/漏桶 | ✅ 已完成 |
| 🟡 链路追踪 | Jaeger/Zipkin、TraceID 传播 | 📝 待更新 |

</details>

---

### 🔧 04 · 微服务工程 🟡 P1

<details>
<summary><b>01-rpc · RPC 与服务治理</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🟡 gRPC 原理](./docs/04-microservices/01-grpc/01-grpc.md) | Protobuf 编码、HTTP/2 多路复用、流式 RPC | ✅ 已完成 |
| [🟡 服务治理](./docs/04-microservices/02-service-governance/02-service-governance.md) | 超时/重试/负载均衡策略 | ✅ 已完成 |

</details>

<details>
<summary><b>02~04 · 网关 / 可观测性 / 部署</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| 🟡 API 网关设计 | 路由/鉴权/限流/灰度 | 📝 待更新 |
| 🟡 Prometheus + Grafana | RED 指标、SLO/SLA | 📝 待更新 |
| 🟢 Kubernetes 核心 | Pod 调度、HPA、滚动发布 | 📝 待更新 |

</details>

---

### 🏗️ 05 · 系统设计 🔴 P0

<details>
<summary><b>02-scenarios · 高频设计题</b>（点击展开）</summary>

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🔴 秒杀系统设计](./docs/05-system-design/01-seckill/01-seckill.md) | 预减库存、异步下单、防超卖 | ✅ 已完成 |
| [🔴 分布式 ID 方案](./docs/05-system-design/05-distributed-id/05-distributed-id.md) | Snowflake、Leaf、UUIDv7 对比 | ✅ 已完成 |
| [🔴 限流系统设计](./docs/05-system-design/06-rate-limiter/06-rate-limiter.md) | 令牌桶、滑动窗口、分布式限流 | ✅ 已完成 |
| [🟡 短链系统设计](./docs/05-system-design/02-short-url/02-short-url.md) | 发号器、跳转、高可用 | ✅ 已完成 |
| [🟡 Feed 流设计](./docs/05-system-design/04-feed/04-feed.md) | 推模式 vs 拉模式 vs 推拉结合 | ✅ 已完成 |
| 🟡 IM 系统设计 | 消息投递、离线消息、已读未读 | 📝 待更新 |

</details>

---

### 🌍 06 · 网络协议 🟡 P1

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🟡 TCP 三次握手 / 四次挥手](./docs/06-network/01-tcp-ip/01-tcp-handshake.md) | TIME_WAIT 问题、连接状态机 | ✅ 已完成 |
| [🟡 HTTP/1.1 vs HTTP/2 vs HTTP/3](./docs/06-network/02-http/01-http-versions.md) | 多路复用、头部压缩、QUIC | ✅ 已完成 |
| 🟡 HTTPS 握手流程 | TLS 握手、证书链、性能优化 | 📝 待更新 |

---

### 📐 07 · 高频算法 🟢 P2

> 目标：30 分钟内写出可运行的 Go 代码

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🟡 滑动窗口](./docs/07-algorithms/01-sliding-window/01-sliding-window.md) | 固定窗口 / 可变窗口模板 | ✅ 已完成 |
| [🟡 二分查找](./docs/07-algorithms/02-binary-search/02-binary-search.md) | 左闭右开模板、变种题型 | ✅ 已完成 |
| [🟡 动态规划](./docs/07-algorithms/04-dp/04-dp.md) | 背包、最长子序列、状态压缩 | ✅ 已完成 |
| [🟡 链表](./docs/07-algorithms/05-linked-list/05-linked-list.md) | 反转、合并、环检测、LRU | ✅ 已完成 |
| [🟢 回溯](./docs/07-algorithms/03-backtracking/03-backtracking.md) | 全排列、组合、子集、剪枝 | ✅ 已完成 |
| [🟢 单调栈](./docs/07-algorithms/07-monotonic-stack/07-monotonic-stack.md) | 下一个更大元素、柱状图 | ✅ 已完成 |
| [🟢 堆](./docs/07-algorithms/08-heap/08-heap.md) | TopK、合并 K 个有序列表 | ✅ 已完成 |
| [🟢 树](./docs/07-algorithms/06-tree/06-tree.md) | DFS/BFS、层序遍历、公共祖先 | ✅ 已完成 |

---

### 🛠️ 08 · 工程素养 🟡 P1

> 5-8 年工程师的核心竞争力，区分度最高的模块

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| [🔴 线上问题排查：OOM](./docs/08-engineering/01-oom/01-oom.md) | heap dump 分析、内存泄漏定位 | ✅ 已完成 |
| [🔴 线上问题排查：CPU 飙升](./docs/08-engineering/02-cpu-spike/02-cpu-spike.md) | pprof 分析、goroutine 死循环 | ✅ 已完成 |
| [🟡 goroutine 泄漏](./docs/08-engineering/05-goroutine-leak/05-goroutine-leak.md) | 识别、定位、修复模式 | ✅ 已完成 |
| 🟡 架构演进复盘 | 从单体到微服务的决策过程 | 📝 待更新 |
| 🟢 Code Review 规范 | 什么值得 block，什么只是建议 | 📝 待更新 |

---

### 💼 09 · 面试策略

| 文章 | 核心考点 | 状态 |
|------|----------|------|
| 简历写法 | 量化成果、STAR 法则、关键词 | 📝 待更新 |
| 行为面试 | 冲突处理、失败经历、晋升理由 | 📝 待更新 |
| 薪资谈判 | 锚点设置、竞争 Offer 利用 | 📝 待更新 |

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

- [ ] 遵循统一的五段式内容结构
- [ ] 结合生产经验，避免纯理论堆砌
- [ ] 关键结论有数据或源码支撑
- [ ] 代码块注明语言类型，Go 代码可直接运行

**Issue 和 PR 均欢迎** 👉 [提 Issue](https://github.com/guocong-bincai/go-interview-guide/issues) | [提 PR](https://github.com/guocong-bincai/go-interview-guide/pulls)

---

## ⭐ Star 趋势

如果本仓库对你有帮助，欢迎点个 Star 🌟，这是对作者最大的鼓励！

[![Star History Chart](https://api.star-history.com/svg?repos=guocong-bincai/go-interview-guide&type=Date)](https://star-history.com/#guocong-bincai/go-interview-guide&Date)

---

<div align="center">

**📖 持续更新中 · 建议 Watch 仓库获取最新动态**

Made with ❤️ for Go Engineers

</div>
