# 04 · 微服务工程

> 考察频率：★★★★☆  优先级：P1

## 文章清单

### 01-rpc · RPC 与服务治理
- [🟡] `01-grpc.md` — gRPC 原理、Protobuf 编码、流式 RPC
- [🟡] `02-service-governance/02-service-governance.md` — 服务治理：超时、重试、负载均衡策略
- [ ] `03-idl-design.md` — API 设计规范、Protobuf 版本兼容

### 02-api-gateway · API 网关
- [🟡] `01-gateway-design.md` — 网关职责：路由、鉴权、限流、灰度
- [ ] `02-auth.md` — JWT/OAuth2/API Key、Token 刷新、权限模型
- [ ] `03-rate-limit.md` — 网关层限流实现、分布式限流（Redis + Lua）

### 03-observability · 可观测性
- [🟡] `03-observability/01-metrics.md` — Prometheus + Grafana、RED 指标、SLO/SLA、Go 运行时调优
- [ ] `02-logging.md` — 结构化日志、ELK 方案、日志采样
- [ ] `03-alerting.md` — 告警规则设计、告警疲劳治理

### 04-deployment · 部署与发布
- [ ] `01-docker.md` — 容器化：多阶段构建、镜像优化、资源限制
- [ ] `02-kubernetes.md` — K8s 核心概念、Pod 调度、HPA、滚动发布
- [ ] `03-cicd.md` — CI/CD 流水线、蓝绿部署、金丝雀发布
