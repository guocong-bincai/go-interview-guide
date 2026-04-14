# 03 · 分布式系统

> 考察频率：★★★★☆  优先级：P1

## 文章清单

### 01-theory · 理论基础
- [🟡] `01-cap-base/01-cap-base.md` — CAP 定理、BASE 理论、实际系统中的取舍
- [🟡] `02-raft/02-raft.md` — Raft 协议：Leader 选举、日志复制、成员变更
- [ ] `03-paxos.md` — Paxos 简述、与 Raft 的对比
- [ ] `04-consistency.md` — 强一致 / 最终一致 / 线性一致性、实际场景选型

### 02-transactions · 分布式事务
- [🟡] `01-2pc-3pc.md` — 2PC/3PC 原理、问题与局限
- [ ] `02-tcc.md` — TCC 模式：Try/Confirm/Cancel 实现与幂等设计
- [ ] `03-saga.md` — Saga 模式：编排 vs 协调、补偿事务设计
- [ ] `04-msg-eventual.md` — 消息最终一致性：本地消息表、事务消息（RocketMQ）
- [ ] `05-seata.md` — Seata AT/TCC/Saga 模式实战

### 03-mq · 消息队列
- [ ] `01-kafka-principle.md` — Kafka 高吞吐原理：零拷贝、顺序写、分区
- [ ] `02-kafka-reliability.md` — 消息可靠性：ACK 机制、ISR、幂等生产者
- [ ] `03-kafka-consumer.md` — 消费者组、Rebalance、顺序消费
- [ ] `04-rocketmq.md` — RocketMQ vs Kafka 对比、事务消息、延迟消息
- [🟡] `03-mq-problems/05-mq-problems.md` — 消息积压、重复消费、消息丢失处理方案

### 04-service-mesh · 服务治理
- [ ] `01-service-discovery.md` — 注册中心（Consul/etcd/Nacos）原理与选型
- [ ] `02-circuit-breaker.md` — 熔断（Hystrix/Sentinel）、限流算法（令牌桶/漏桶）
- [ ] `03-tracing.md` — 链路追踪（Jaeger/Zipkin）、TraceID 传播、采样策略
