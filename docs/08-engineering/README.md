# 08 · 工程素养

> 考察频率：★★★★★  优先级：P1
> 5-8 年工程师的核心竞争力，区分度最高的模块

## 文章清单

### 01-project-design · 项目设计与复盘
- [✅] [技术选型方法论：如何在面试中讲清楚为什么选 X](01-project-design/16-01-tech-selection.md)
- [✅] [架构演进：从单体到微服务的决策过程](01-project-design/07-02-architecture-evolution.md)
- [✅] [项目复盘模板：背景 / 方案 / 结果 / 反思](03-project-review/23-03-project-review.md)
- [⏳] [故障复盘（Postmortem）方法论：RCA、时间线、改进项闭环](01-project-design/01-04-incident-postmortem.md)
- [⏳] [重构与技术债治理：重构 vs 重写、灰度重构、技术债偿还](01-project-design/02-05-refactoring-and-tech-debt.md)
- [⏳] [Go 服务架构设计：分层、Clean Architecture、Hexagonal](01-project-design/08-06-code-architecture.md)

### 02-troubleshooting · 线上问题排查
- [✅] [OOM 排查：heap dump 分析、内存泄漏定位](01-oom/06-01-oom.md)
- [✅] [CPU 飙升排查：pprof、goroutine 死循环](02-cpu-spike/09-02-cpu-spike.md)
- [✅] [死锁排查：数据库死锁、Go 并发死锁](02-troubleshooting/11-03-deadlock.md)
- [✅] [接口高延迟排查：链路追踪、GC 停顿、连接池](02-troubleshooting/03-04-high-latency.md)
- [✅] [goroutine 泄漏：识别、定位、修复模式](05-goroutine-leak/15-05-goroutine-leak.md)
- [⏳] [混沌工程与故障演练：故障注入、演练机制、回滚控制](02-troubleshooting/19-08-chaos-engineering.md)

### 03-leadership · 技术领导力
- [✅] [Code Review 规范：Block vs Suggestion、Go 项目检查清单、Review 流程](03-leadership/20-01-code-review.md)
- [✅] [技术规划：季度 / 年度技术 OKR 制定](02-tech-planning/10-02-tech-planning.md)
- [✅] [带新人：技术传承、文档文化、知识管理](03-mentoring/22-03-mentoring.md)
- [⏳] [工程质量体系落地：质量门禁、lint、测试覆盖率、CI](03-leadership/12-04-engineering-quality.md)
- [⏳] [跨团队协作：目标对齐、资源冲突、复杂项目推进](03-leadership/13-05-cross-team-collaboration.md)
- [⏳] [招聘与面试：高级工程师如何识别候选人](03-leadership/21-06-hiring-interviewing.md)

### 03-leadership · 技术领导力（工程落地）
- [✅] [Code Review 规范：Block vs Suggestion、Go 项目检查清单、Review 流程](03-leadership/20-01-code-review.md)
- [✅] [技术规划：季度 / 年度技术 OKR 制定](02-tech-planning/10-02-tech-planning.md)
- [✅] [带新人：技术传承、文档文化、知识管理](03-mentoring/22-03-mentoring.md)
- [✅] [CI/CD 流水线设计：GitLab CI/Jenkins + 质量门禁 + 灰度发布](03-leadership/25-01-ci-cd-pipeline.md)
- [✅] [可观测性：日志/指标/链路追踪落地实践](03-leadership/25-02-observability.md)
- [✅] [Module Workspace 与依赖管理：Monorepo 大项目最佳实践](03-leadership/25-03-dependency-management.md)
- [⏳] [工程质量体系落地：质量门禁、lint、测试覆盖率、CI](03-leadership/12-04-engineering-quality.md)
- [⏳] [跨团队协作：目标对齐、资源冲突、复杂项目推进](03-leadership/13-05-cross-team-collaboration.md)
- [⏳] [招聘与面试：高级工程师如何识别候选人](03-leadership/21-06-hiring-interviewing.md)

### 04-performance-governance · 质量治理与交付
- [✅] [压测方法论：从单接口到全链路](04-performance-governance/04-01-load-testing.md)
- [✅] [全链路压测实战：影子流量与数据隔离](04-performance-governance/05-02-full-link-stress-testing.md)
- [✅] [测试策略：单测、集成测试、契约测试](04-performance-governance/14-03-testing-strategy.md)
- [✅] [测试覆盖率提升：Table-Driven Test + Mock + 覆盖率盲区消除](04-performance-governance/25-02-test-coverage-strategy.md)
- [✅] [数据库迁移：零停机 Schema 变更五步法](04-performance-governance/25-03-db-migration-strategy.md)
- [✅] [容器镜像优化：多阶段构建 + scratch + BuildKit 缓存](04-performance-governance/25-04-docker-image-optimization.md)
- [✅] [金丝雀发布：K8s 灰度部署 + 优雅关闭 + 快速回滚](04-performance-governance/25-05-canary-release.md)

### 06-race-detector · 并发安全与性能诊断
- [✅] [Race Detector：go test -race 原理与 CI 集成](06-race-detector/24-01-race-detector.md)
- [✅] [内存深度分析：alloc_space vs inuse_space](06-race-detector/24-02-memory-profiling.md)
- [✅] [Benchmark 最佳实践：基准测试方法论](06-race-detector/24-03-benchmark-best-practices.md)
- [✅] [错误处理策略：errors.Is/As/Wrap 体系设计](06-race-detector/24-04-error-handling-strategy.md)
- [✅] [Go tool trace：运行时事件追踪与性能分析](06-race-detector/24-05-go-tool-trace.md)
