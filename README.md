<div align="center">

# 🏆 Go 高级工程师面试宝典

> **面向 5~8 年 Go 后端工程师 · 大厂面试核心考点全覆盖**

[![Stars](https://img.shields.io/github/stars/guocong-bincai/go-interview-guide?style=flat-square&logo=github&color=yellow)](https://github.com/guocong-bincai/go-interview-guide/stargazers)
[![Forks](https://img.shields.io/github/forks/guocong-bincai/go-interview-guide?style=flat-square&logo=github&color=blue)](https://github.com/guocong-bincai/go-interview-guide/network/members)
[![License](https://img.shields.io/github/license/guocong-bincai/go-interview-guide?style=flat-square&color=green)](./LICENSE)
[![题目数量](https://img.shields.io/badge/题目-364-orange?style=flat-square)](./docs)
[![版本](https://img.shields.io/badge/版本-v5.10-blue?style=flat-square)](./docs)

[📚 模块导航](#-模块导航) · [🗺️ 学习路线](#️-学习路线) · [📝 更新记录](#-更新记录) · [🤝 贡献指南](#-贡献指南)

</div>

---

## 📚 模块导航

> 共 **356** 道高频面试题 ｜ 12 大核心模块 ｜ 按面试优先级排序

| 序号 | 模块 | 题数 | 频率 | 优先级 | 覆盖内容 |
|:----:|------|:----:|:----:|:------:|----------|
| 01 | [**Go 语言深度**](docs/01-golang/README.md) | **88** | ★★★★★ | P0 | GMP/GC/内存分配/channel/sync/interface/泛型/逃逸分析/pprof/false sharing/block profile/race detector
| 02 | [**数据库**](docs/02-database/README.md) | **30** | ★★★★★ | P0 | MySQL 索引/MVCC/锁/Redis 持久化/缓存三大问题/分库分表/主从复制/Buffer Pool |
| 03 | [**分布式系统**](docs/03-distributed/README.md) | **30** | ★★★★☆ | P1 | CAP/BASE/Raft/2PC/TCC/Saga/Kafka/gRPC/限流熔断/ZK/分布式锁 |
| 04 | [**微服务工程**](docs/04-microservices/README.md) | **22** | ★★★★☆ | P1 | gRPC/Protobuf/网关/限流/可观测性/K8s/CI-CD/无损发布/服务降级/BFF |
| 05 | [**系统设计**](docs/05-system-design/README.md) | **19** | ★★★★★ | P0 | 秒杀/短链/IM/Feed流/支付/分布式ID/限流/CQRS |
| 06 | [**网络协议**](docs/06-network/README.md) | **14** | ★★★★☆ | P1 | TCP 三次握手/HTTP1.1-2-3/HTTPS/gRPC/WebSocket/安全 |
| 07 | [**高频算法**](docs/07-algorithms/README.md) | **90** | ★★★★☆ | P2 | 滑动窗口/二分/回溯/DP/链表/树/单调栈/堆/TopK |
| 08 | [**工程素养**](docs/08-engineering/README.md) | **28** | ★★★★★ | P1 | 技术选型/架构演进/OOM排查/Race Detector/Benchmark/内存分析/trace/error handling/技术领导力 |
| 09 | [**面试策略**](docs/09-interview-strategy/README.md) | **11** | ★★★☆☆ | P1 | STAR法则/行为面试/简历写法/自我介绍/系统设计面试/Live Coding/薪资谈判/晋升答辩/全流程节奏控制 |
| 10 | [**项目实战问题**](docs/10-real-problems/README.md) | **7** | ★★★★★ | P0 | 业务方案/性能问题/数据一致性/可用性/并发 |
| 11 | [**Go 标准库生产实践**](docs/11-go-std-practice/README.md) | **9** | ★★★★☆ | P1 | io/encoding/json/time/testing + sync单飞/Context/net/http 进阶 |
| 12 | [**Linux / 操作系统**](docs/12-linux-os/README.md) | **8** | ★★★★☆ | P1 | 文件系统/进程线程/虚拟内存/零拷贝/cgroup/IPC |

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
| 2026-08-26 | v5.10 | 数据库模块新增 2 篇：MySQL 字符集 utf8mb4 + COLLATION / 事务回滚机制（UNDO LOG）+ 强制索引优化器 HINT；Redis 内存管理专题（碎片率监控与 Active Defrag / MULTI 不支持回滚 / PubSub vs Stream 可靠性对比 / Pipeline 性能优化），共新增 10 题
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
