# 03 · 分布式系统

> 考察频率：★★★★☆  优先级：P1

## 文章清单

### 01-theory · 理论基础
- [✅] [CAP 定理、BASE 理论、实际系统取舍 + PACELC 模型](01-theory/06-01-cap-base.md)
- [✅] [Raft 协议：Leader 选举、日志复制、成员变更](01-theory/01-02-raft.md)
- [✅] [Paxos 简述、与 Raft 的对比](01-theory/02-03-paxos.md)
- [✅] [强一致 / 最终一致 / 线性一致性、实际场景选型](01-theory/07-04-consistency.md)
- [✅] [一致性哈希（Consistent Hashing）：hash 环、虚拟节点与 Go 实现](01-theory/08-07-consistent-hashing.md)
- [✅] [Gossip 协议：原理、Anti-Entropy、故障检测与 Go 实战](01-theory/09-08-gossip-protocol.md)

### 02-transactions · 分布式事务
- [✅] [2PC/3PC 原理、问题与局限](02-transactions/08-01-2pc-3pc.md)
- [✅] [TCC 模式：Try/Confirm/Cancel 实现与幂等设计](02-transactions/09-02-tcc.md)
- [✅] [Saga 模式：编排 vs 协调、补偿事务设计](02-transactions/10-03-saga.md)
- [✅] [消息最终一致性：本地消息表、事务消息（RocketMQ）](02-transactions/18-04-msg-eventual.md)
- [✅] [Seata AT/TCC/Saga 模式实战](02-transactions/11-05-seata.md)

### 03-mq · 消息队列
- [✅] [Kafka 高吞吐原理：零拷贝、顺序写、分区](03-mq/12-01-kafka-principle.md)
- [✅] [消息可靠性：ACK 机制、ISR、幂等生产者](03-mq/19-02-kafka-reliability.md)
- [✅] [消费者组、Rebalance、顺序消费](03-mq/20-03-kafka-consumer.md)
- [✅] [Kafka 生产者调优：batch.size / linger.ms / 压缩策略 / 分区器](03-mq/25-07-kafka-producer-tuning.md)
- [✅] [RocketMQ vs Kafka 对比、事务消息、延迟消息](03-mq/21-04-rocketmq.md)
- [✅] [消息积压、重复消费、消息丢失处理方案](03-mq/13-05-mq-problems.md)

### 04-service-mesh · 服务治理
- [✅] [注册中心（Consul/etcd/Nacos）原理与选型](04-service-mesh/23-01-service-discovery.md)
- [✅] [链路追踪（Jaeger/Zipkin）、TraceID 传播、采样策略](04-service-mesh/16-03-tracing.md)
- [✅] [限流算法 / 熔断器 / 重试策略：令牌桶 vs 漏桶 vs 滑动窗口 + 三态机熔断 + 指数退避](04-service-mesh/17-04-rate-limiter-circuit-breaker.md)
- [✅] [gRPC 基础与原理：HTTP/2 多路复用、Protobuf、四种流式 RPC、拦截器、负载均衡](04-service-mesh/18-05-grpc-fundamentals.md)
- [✅] [负载均衡选型：Nginx vs HAProxy vs Envoy vs Traefik](04-service-mesh/19-06-load-balancer-selection.md)

### 05-coordination · 协调服务
- [✅] [Redis vs etcd vs ZooKeeper 分布式锁横向对比](05-coordination/05-03-distributed-lock-comparison.md)
- [✅] [etcd 原理与实战](05-coordination/17-01-etcd.md)
- [✅] [配置中心选型与实践](05-coordination/24-02-config-center.md)
- [✅] [ZooKeeper 原理与实战：ZAB 协议、Session、EPHEMERAL 节点、Watch 机制](05-coordination/06-01-zookeeper-fundamentals.md)
- [✅] [Redis Sentinel vs Cluster：架构对比、故障转移与分片哈希槽](05-coordination/07-01-sentinel-vs-cluster.md)
