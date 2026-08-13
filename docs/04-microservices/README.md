# 04 · 微服务工程

> 考察频率：★★★★☆  优先级：P1

## 文章清单

### 01-rpc · RPC 与服务治理
- [✅] [gRPC 原理、Protobuf 编码、流式 RPC](01-rpc/04-01-grpc.md)
- [✅] [服务治理：超时、重试、负载均衡策略](01-rpc/05-03-service-governance.md)
- [✅] [熔断与限流：三态熔断 + 令牌桶/漏桶/滑动窗口](01-rpc/01-02-circuit-breaker.md)
- [✅] [服务降级与舱壁隔离：降级开关、线程池 vs 信号量隔离](01-rpc/06-05-degradation-isolation.md)
- [✅] [接口幂等性设计：唯一约束/去重表/Token/状态机](01-rpc/07-06-api-idempotency.md)
- [✅] [API 设计规范、Protobuf 版本兼容](01-rpc/12-04-idl-design.md)

### 02-api-gateway · API 网关
- [✅] [网关职责：路由、鉴权、限流、灰度](02-api-gateway/06-01-gateway-design.md)
- [✅] [JWT/OAuth2/API Key、Token 刷新、权限模型](02-api-gateway/07-02-auth.md)
- [✅] [网关层限流实现、分布式限流（Redis + Lua）](02-api-gateway/08-03-rate-limit.md)
- [✅] [BFF 模式：Backend for Frontend 设计与实践](02-api-gateway/09-04-bff.md)

### 03-observability · 可观测性
- [✅] [Prometheus + Grafana、RED 指标、SLO/SLA、Go 运行时调优](03-observability/09-01-metrics.md)
- [✅] [Prometheus + Grafana 监控实战：PromQL、Histogram、告警、踩坑](03-observability/11-04-prometheus-grafana-practice.md)
- [✅] [结构化日志、ELK 方案、日志采样](03-observability/14-02-logging.md)
- [✅] [告警规则设计、告警疲劳治理](03-observability/10-03-alerting.md)
- [✅] [可观测性三支柱整合：OpenTelemetry、TraceID 贯穿、eBPF 零侵入](03-observability/15-05-otel-three-pillars.md)

### 04-deployment · 部署与发布
- [✅] [容器化：多阶段构建、镜像优化、资源限制](04-deployment/15-01-docker.md)
- [✅] [K8s 核心概念、Pod 调度、HPA、滚动发布、灰度发布](04-deployment/16-02-kubernetes.md)
- [✅] [K8s 控制面深度原理：API Server/Controller/Scheduler/Informer](04-deployment/02-03-k8s-internals.md)
- [✅] [灰度发布实战：策略、Istio、数据库变更、回滚](04-deployment/03-04-gray-release.md)
- [✅] [无损发布：优雅上下线与流量摘除（readiness/preStop/SIGTERM）](04-deployment/06-06-zero-downtime-release.md)
- [✅] [CI/CD 流水线、蓝绿部署、金丝雀发布](03-cicd/13-03-cicd.md)

### 05-architecture · 架构设计
- [✅] [微服务拆分原则与粒度：DDD 限界上下文、康威定律、绞杀者模式](05-architecture/01-01-microservice-split.md)
