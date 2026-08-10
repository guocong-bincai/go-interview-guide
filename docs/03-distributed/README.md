# 03 · 分布式系统

> 考察频率：★★★★☆  优先级：P1

## 文章清单

### 01-theory · 理论基础
- [✅] [CAP 定理、BASE 理论、实际系统取舍 + PACELC 模型](01-theory/06-01-cap-base.md)
- [✅] [Raft 协议：Leader 选举、日志复制、成员变更](01-theory/01-02-raft.md)
- [✅] [Paxos 简述、与 Raft 的对比](01-theory/02-03-paxos.md)
- [✅] [强一致 / 最终一致 / 线性一致性、实际场景选型](01-theory/07-04-consistency.md)

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
- [✅] [RocketMQ vs Kafka 对比、事务消息、延迟消息](03-mq/21-04-rocketmq.md)
- [✅] [消息积压、重复消费、消息丢失处理方案](03-mq/13-05-mq-problems.md)

### 04-service-mesh · 服务治理
- [✅] [注册中心（Consul/etcd/Nacos）原理与选型](04-service-mesh/23-01-service-discovery.md)
- [✅] [链路追踪（Jaeger/Zipkin）、TraceID 传播、采样策略](04-service-mesh/16-03-tracing.md)
