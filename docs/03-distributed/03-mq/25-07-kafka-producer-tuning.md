# Kafka 生产者调优：batch.size、linger.ms 与分区策略

> 考察频率：★★★☆☆  难度：★★★★☆
> 关键词：批量发送、压缩策略、幂等性、分区器、事务消息

## 核心答案（30 秒版）

Kafka 生产者的性能调优核心在于**让 Broker 的批量写入能力最大化**——通过 `batch.size`（单批次最大字节数）、`linger.ms`（等待批处理的时间窗口）、`compression.type`（lz4/zstd 压缩）三个参数协同工作，将零散的小消息合并为大批次顺序写。

**关键原则**：不要追求极低延迟的微批（1~5ms linger），而应根据吞吐需求调整 batch 大小。对于高吞吐场景，建议 `batch.size=65536`(64KB)、`linger.ms=10~50`、`compression=lz4`。

---

## 深度展开

### 1. 生产者的发送流程

```
Producer 发送消息到 Broker 的完整链路：

┌───────────┐    ┌──────────┐    ┌────────────┐    ┌────────────┐
│ Producer  │──▶ │ Batchers │──▶ │ RecordAccumulator │──▶ │ NetworkIO │
│ (应用线程) │    │(组装批次)│    │(内存缓冲池)    │    │(异步发送) │
└───────────┘    └──────────┘    └────────────┘    └─────┬────┘
                                                          │
                                                         ▼
                                                    ┌────────────┐
                                                    │    Broker   │
                                                    │ Leader Partition │
                                                    └────────────┘

核心组件：
- KeySerializer / ValueSerializer：序列化 key/value
- Partitioner：决定消息发到哪个 partition
- RecordAccumulator：内存中的 batch buffer
- Sender（网络线程）：从 accumulator 拉取批次发送到 broker
```

### 2. 关键配置详解

```go
// Go kubernetes/client-go / Segmentio kafka/go-kafka 示例
// 以 Segmentio go-kafka 为例展示核心参数

producer := segmentio.NewProducer(segmentio.WriterConfig{
    Brokers:      []string{"10.0.0.1:9092", "10.0.0.2:9092"},
    Topic:        "order-events",
    
    // ===== 吞吐量调优核心参数 =====
    
    // batch.size: 单批次最大字节数（不是条数！）
    // 默认 1MB，通常不需要改
    BatchSize: 65536, // 64KB — 小 topic 可以调到 64KB 减少尾延迟
    
    // linger.ms: 等待时间窗口（毫秒）
    // 消息不会立即发送，而是先攒成 batch
    // 满足以下任一条件就发送：batch 满 OR 时间到
    LingerMs: 10, // 10ms — 兼顾吞吐和延迟
    
    // compression.type: 压缩算法
    // none = 不压缩，gzip = 强压缩（慢但省带宽），lz4 = 快压缩（推荐），zstd = 新标准
    Compression: segmentio.Lz4Compression,
    
    // ===== 可靠性参数 =====
    
    // acks: 
    // "1" 或 -1 (all) — 生产环境用 all
    Acks: "all",
    
    // retries: 自动重试次数
    Retries: math.MaxInt32, // 无限重试（配合 idempotent 避免重复）
    
    // ===== 幂等性与事务 =====
    
    // enable.idempotence: 开启幂等生产者（默认 true，acks=all 时）
    EnableIdempotence: true, // 每个批次有唯一 sequence number
    
    // transactional.id: 必须设置才能用事务
    // 同一事务 ID 只能由一个生产者使用
    TransactionalID: "order-producer-01",
})
```

### 3. 参数组合的效果矩阵

| batch.size | linger.ms | throughput | latency | 适用场景 |
|------------|-----------|------------|---------|---------|
| 16KB | 0ms | ~3w msg/s | <10ms | 实时事件流，低延迟优先 |
| 64KB | 10ms | ~20w msg/s | ~20ms | **通用推荐**，平衡吞吐和延迟 |
| 256KB | 50ms | ~80w msg/s | ~60ms | 日志采集，纯吞吐优先 |
| 1MB | 100ms | ~100w+ msg/s | >100ms | 离线数据采集 |

### 4. 分区器工作原理

```
Partitioner 决定消息发给哪个 Partition：

情形1：指定了 key → Hash(key) % numPartitions
  key="user-123" → hash("user-123") % 6 = 3 → Partition 3

情形2：没指定 key → RoundRobin（轮询所有 Partition）
  第1条 → P0, 第2条 → P1, ..., 第6条 → P5, 第7条 → P0 ...

情形3：自定义 Partitioner（Go 实现）
type CustomPartitioner struct {
    counter uint64
}

func (p *CustomPartitioner) Partition(message *segmentio.Message, numPartitions int) (int, error) {
    if message.Key != nil {
        // 有 key：hash 路由（有序保证）
        h := crc32.ChecksumIEEE(message.Key)
        return int(h % uint32(numPartitions)), nil
    }
    // 无 key：按时间戳分段，同时间段的消息发同一个分区
    atomic.AddUint64(&p.counter, 1)
    bucket := (atomic.LoadUint64(&p.counter) / 1000000) % uint64(numPartitions)
    return int(bucket), nil
}
```

### 5. Idempotent Producer（幂等生产者）

```go
// 开启 idempotent 后，Kafka 保证同一 partition 内消息不丢不重

producer, err := segmentio.NewProducer(segmentio.WriterConfig{
    Brokers:           []string{"10.0.0.1:9092"},
    Topic:             "events",
    EnableIdempotence: true,  // ← 关键开关
    Acks:              "all",  // 必须为 all
})

// 底层机制：
// 1. 每个 connectionPerPartition 维护一个单调递增的 sequence number
// 2. Broker 收到消息时检查 sequence number，重复的自动丢弃
// 3. 重启后旧连接失效，sequence number 归零（不会重复）

// 限制：idempotent 一次只能对单个 partition 发消息
// 如果需要跨多个 partition 的事务，必须使用 transactional producer
```

### 6. 事务生产者（Transaction Producer）

```go
producer.InitTxn()
producer.BeginTxn()

// 在事务中发送多条消息到不同 topic/partition
for _, topic := range topics {
    producer.SendMessages(ctx, msgs...)
}

// 提交或回滚整个事务
err := producer.CommitTxn()     // 全部成功
// 或
producer.AbortTxn()            // 任何失败则全部回滚
```

**事务消息 vs 普通消息的选择**：

| 场景 | 推荐方案 |
|------|---------|
| 单 topic 生产，允许少量重复 | Idempotent Producer |
| 多 topic/partition 原子发布 | Transaction Producer |
| 本地 DB + MQ 最终一致性 | 本地消息表 / RocketMQ 事务消息 |

### 7. Go 中的常见坑

```go
// ❌ 错误1：每次创建新的 Producer
// 后果：频繁建立 TCP 连接，性能极差
func badProducer() {
    for i := 0; i < 10000; i++ {
        p := segmentio.NewProducer(...)  // 每次循环都新建
        p.WriteMessages(context.Background(), msg)
        p.Close()                         // 关闭连接
    }
}

// ✅ 正确：全局复用 Producer 实例
var globalProducer *segmentio.Producer

func init() {
    var err error
    globalProducer, err = segmentio.NewProducer(segmentio.WriterConfig{
        Brokers:   []string{"10.0.0.1:9092"},
        Topic:     "events",
        BatchSize: 65536,
        LingerMs:  10,
    })
    if err != nil { log.Fatal(err) }
}

// ❌ 错误2：没有指定 partition key
// 后果：RoundRobin 虽然均匀，但失去了 partition 级别的有序性
p.WriteMessages(ctx, &segmentio.Message{
    Value: []byte("order created"),
    // Key: nil ← 这样就没有 order 级别的全局有序保证了
})

// ✅ 正确：按业务 key 路由
p.WriteMessages(ctx, &segmentio.Message{
    Key:   []byte("order-123"),  // 相同 key → 固定 partition
    Value: []byte("order created"),
})
```

---

## 面试高频追问

**Q：batch.size 和 linger.ms 如何配合调优？**

> 它们是互补的：`batch.size` 控制单次发送的最大字节量，`linger.ms` 控制等待多久才发送。**理想状态是**：大多数 batch 因为达到 size 而触发发送（效率高），只有偶尔少数 batch 因为时间到了才强制发送（防延迟）。比如设 batch.size=64KB、linger.ms=10ms，当消息量大时会快速填满批次，消息量小时最多等 10ms 也会发送。

**Q：为什么 lz4 比 gzip 更适合 Kafka？**

> Kafka 追求的是**高性能**，不是极致压缩率。lz4 的压缩速度比 gzip 快 2~10 倍，解压速度更快。虽然压缩率略低（lz4 约 2:1，gzip 约 3:1），但在 K8s/局域网环境下带宽不是瓶颈，CPU 时间才是。所以 lz4 是性价比最高的选择。zstd 正在成为新趋势——它介于两者之间。

**Q：幂等性和事务有什么区别？**

> 幂等只保证**单 partition** 内不丢不重；事务保证**跨 partition/跨 topic** 的原子性。幂等是轻量级的（只需 sequence number），事务需要协调器支持（txn coordinator）。大部分场景幂等就够了，只有在需要跨数据源的原子操作时才需要事务。

---

## 面试话术

**Q：如果让你优化 Kafka 的生产者性能，你会怎么做？**

> 先从三个参数入手：**增大 batch.size（到 64KB~256KB）** 让每次发送更多数据；**加 linger.ms（10~50ms）** 给批次凑够数据的时间窗口；**启用 lz4 压缩** 省带宽又不牺牲 CPU。这三个参数配合能让单机生产者从几千条/秒提升到几十万条/秒。同时保证 acks=all 加上 idempotent 开启，既不丢数据也不重复。

🗣️ **记忆口诀**：**"batch大一点，linger长一点，lz4压一压"**

---

*参考文档：[Apache Kafka Producer Config](https://kafka.apache.org/documentation/#producerconfigs) · [Segmentio Go-Kafka](https://github.com/segmentio/kafka-go)*

[🏠 首页](../../../README.md) · [📦 分布式系统](../README.md) · [💬 消息队列](../../docs/03-distributed/03-mq/README.md)
