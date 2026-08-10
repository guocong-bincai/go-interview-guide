# Kafka 高吞吐原理

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：顺序写、零拷贝、PageCache、分区、批量处理、压缩

## Kafka 为什么这么快

Kafka 的高性能来自**五个核心优化**的叠加效应：

```
性能优化组合拳
=============

1. 顺序写磁盘      → 写入速度接近内存（O(1) vs O(n)）
2. 操作系统 PageCache → 内存缓存热数据，读取命中率高
3. 零拷贝（sendfile）→ 跳过用户态，内核直接传输
4. 批量处理         → 合并小消息，减少网络往返
5. 分区水平扩展     → 并行消费，无单点瓶颈
```

---

## 1. 顺序写磁盘

### 传统随机写的瓶颈

普通磁盘的随机写入速度约 **0.5~2 MB/s**，因为每次写入需要：
1. 机械臂移动到对应磁道（寻道）
2. 等待磁盘旋转到正确位置（旋转延迟）
3. 写入数据

### Kafka 的顺序写优化

Kafka 写消息到磁盘时，**追加（Append）到日志文件末尾**，而不是随机写入：

```bash
# Kafka 的存储结构（每个 Partition 对应一个日志文件）
/var/lib/kafka/data/kafka-topic-0/
  ├── 00000000000000000000.log    # 当前活跃的日志文件
  ├── 00000000000000000001.log    # 旧的日志段（Segment）
  ├── 00000000000000000002.log
  └── index/
      ├── 00000000000000000000.index  # 偏移量索引
      └── 00000000000000000001.index
```

**顺序写的磁盘速度可达 500~600 MB/s**，接近内存。

**原理**：
- 顺序写只需要一次寻道（写入开始时），之后持续写入不移动磁头
- 机械臂在磁盘外圈连续移动，磁盘持续旋转
- 写入变成了"流式"操作

### Go 验证：顺序写 vs 随机写

```go
import (
    "os"
    "time"
)

func writeSequential(filename string, size int) time.Duration {
    f, _ := os.Create(filename)
    defer f.Close()

    start := time.Now()
    buf := make([]byte, 4096) // 4KB 块
    for i := 0; i < size; i += len(buf) {
        f.Write(buf) // 顺序追加
    }
    f.Sync() // 强制刷盘
    return time.Since(start)
}
```

在机械硬盘上，顺序写的速度是随机写的 **100 倍以上**。

---

## 2. 操作系统 PageCache

### Kafka 的读写路径

```
写入：
Producer → Socket Buffer → PageCache（内存） → 磁盘（异步刷盘）

读取：
磁盘 → PageCache（热数据命中）→ Socket Buffer → Consumer
（如果 PageCache 有数据，直接跳过磁盘读取）
```

### PageCache 工作原理

Linux 把空闲内存用作磁盘缓存（PageCache）：
- 写数据时，先写到 PageCache（内存），后台异步刷到磁盘
- 读数据时，先查 PageCache，命中则直接返回（内存速度）

**Kafka 的读写高度依赖 PageCache**：
- Producer 写消息 → 先到 PageCache → 后台线程刷盘
- Consumer 读消息 → 如果数据还在 PageCache 中 → 直接从内存读取（极快）
- 消费过的消息不会立即从 PageCache 清除（LRU 淘汰），**热点消息被反复读取时全部命中内存**

### 内存规划建议

```properties
# Kafka 服务端配置：预留足够内存给 PageCache
# 操作系统建议：预留 30-50% 内存给 PageCache
# 假设机器有 64GB 内存，Kafka JVM 用 8GB，系统预留 30GB 给 PageCache

# JVM 堆内存不要太大（留内存给 PageCache）
-Xms8g -Xmx8g

# Kafka 本身使用堆外内存用于网络和日志读写
max.memory.bytes=1073741824  # 1GB
```

### 生产经验

如果 PageCache 命中率低（内存不够），Kafka 性能会急剧下降：

```bash
# 查看 PageCache 命中率
# cat /proc/meminfo | grep -E "Cached|Buffers"
# Cached 表示 PageCache + 缓冲区大小

# 查看磁盘 I/O 是否饱和（高 I/O wait 表示 PageCache 不足）
iostat -x 1
# %util 接近 100% 说明磁盘是瓶颈（PageCache 不足）
```

---

## 3. 零拷贝（sendfile）

### 传统读取发送路径（4 次拷贝）

```
1. 磁盘 → 内核缓冲区（PageCache）    [第1次拷贝]
2. 内核缓冲区 → 用户态内存           [第2次拷贝]
3. 用户态内存 → Socket 缓冲区        [第3次拷贝]
4. Socket 缓冲区 → 网卡              [第4次拷贝]
```

```go
// 传统方式（用户态-内核态切换 + 4次拷贝）
data, _ := os.ReadFile("kafka.log")  // 磁盘 → 用户内存（2次拷贝）
conn.Write(data)                      // 用户内存 → 网卡（2次拷贝）
```

### 零拷贝路径（2 次拷贝）

Kafka 使用 Linux 的 `sendfile` 系统调用，数据直接从 PageCache 发到网卡：

```
1. 磁盘 → PageCache                    [第1次拷贝：DMA 直接内存访问]
2. PageCache → 网卡（内核直接传输）     [第2次拷贝：DMA]
```

用户态完全不参与，实现"零拷贝"（CPU 不参与数据搬运）。

```go
// Linux sendfile（零拷贝）
import "golang.org/x/sys/unix"

func sendFileZeroCopy(src *os.File, dst int) error {
    // 数据从 src 文件的 PageCache 直接通过 DMA 发到 dst socket
    _, err := unix.Sendfile(dst, src.Fd(), nil, 0)
    return err
}
```

**效果**：单次传输吞吐量提升 **2-3 倍**，CPU 开销大幅降低。

### Kafka 中的零拷贝使用

Kafka 客户端读取消息时的核心路径：

```java
// Kafka 源码（Kafka 是 Java，但原理一样）
// org.apache.kafka.common.network.NetworkIO:
//
// 1. 使用 Selector 读取网络数据
// 2. 使用 sendfile 将日志文件数据直接发往 socket
//    FileChannel.transferTo() → 调用 Linux sendfile
```

---

## 4. 批量处理（Batch）

### 生产者批量发送

Kafka Producer 不会每条消息发一次，而是**积累一批后再发送**：

```go
// Kafka Go Producer 批量配置
producer, _ := sarama.NewProducer([]string{"localhost:9092"}, &sarama.Config{
    // 批量积累消息数量
    BatchSize: 200,        // 积累 200 条消息
    // 批量等待时间（任一条件满足就发送）
    BatchTimeout: 10 * time.Millisecond,
    // 压缩
    Compression: sarama.CompressionSnappy,
})

// 消息先放入本地批次，满足条件后一次性发送
// 网络往返从 1次/消息 → 1次/批次
```

**对比**：
- 无批量：10000 条消息 = 10000 次网络往返
- 有批量（Batch=200）：10000 条消息 = 50 次网络往返

### 批量压缩

批次中的消息会被**一起压缩**（而不是逐条压缩）：

| 压缩算法 | 压缩率 | CPU 消耗 | 适用场景 |
|---------|--------|---------|---------|
| gzip | 高 | 高 | 带宽敏感场景 |
| snappy | 中 | 低 | Kafka 默认，推荐 |
| lz4 | 中高 | 低 | 低延迟场景 |
| zstd | 最高 | 中 | Kafka 2.1+，最新 |

压缩在批次层面进行，压缩后整个批次作为一条消息传输，进一步减少网络开销。

### 消费者批量拉取

消费者也是批量拉取，而不是逐条拉取：

```go
config := &sarama.Config{
    // 消费者每次从 Broker 拉取的最大消息数
    Consumer.MaxWaitTime = 500 * time.Millisecond
    // 实际拉取量由 Broker 和 Consumer 的配置共同决定
}
```

---

## 5. 分区（Partition）水平扩展

### 分区是 Kafka 并行性的基础

```
Topic: orders（3 个分区）

Partition 0: [msg1, msg2, msg3, msg7, msg10]  → Consumer A（负责）
Partition 1: [msg4, msg5, msg8, msg11]        → Consumer B（负责）
Partition 2: [msg6, msg9, msg12]              → Consumer C（负责）
```

- **写入**：Producer 根据 key 路由到对应 Partition（默认 hash(key) % n）
- **消费**：每个 Partition 只能被一个 Consumer 实例消费（保证顺序）
- **并行**：分区数 = 最大并发消费数

### 生产者分区策略

```go
// Kafka Go Producer 分区路由
func partitioner(key []byte, numPartitions int32) int32 {
    if key == nil {
        // 无 key：轮询发送到各分区（保证负载均衡）
        return atomic.AddInt64(&counter, 1) % int64(numPartitions)
    }
    // 有 key：哈希路由（相同 key 一定到同一分区，保证顺序）
    return murmur2(key) % numPartitions
}
```

### 分区数与消费者数的关系

```
分区数 = 消费者数（最大并行度）

场景：
- 3 个分区，2 个消费者：1 个消费者处理 2 个分区，另 1 个处理 1 个分区
- 3 个分区，6 个消费者：最多 3 个消费者同时工作，剩余 3 个空闲
- 3 个分区，1 个消费者：所有消息被单个消费者处理（无并行）
```

**生产经验**：
- 分区数设置建议：**Broker 数量 × 消费者数量 × 2**（留余量）
- 分区数只能增加不能减少
- 分区数过多会增加元数据开销（ZooKeeper/Controller 压力）

```bash
# 查看 Topic 分区数
kafka-topics.sh --describe --topic orders --zookeeper localhost:2181
```

---

## 综合性能数据

| 操作 | 传统方式 | Kafka（优化后） |
|------|---------|----------------|
| 顺序写磁盘 | 0.5~2 MB/s | 500~600 MB/s |
| 读取发送 | 4次拷贝，用户态参与 | 2次拷贝，零拷贝 |
| 网络效率 | 每消息一次往返 | 每批次一次往返 |
| 消息吞吐 | ~10 万条/秒 | ~**100 万+条/秒** |

---

## 面试高频追问

**Q1: Kafka 和 RocketMQ 的区别？**
→ Kafka 追求高吞吐，顺序写 + 零拷贝 + 批量是其核心，适合日志和大数据场景。RocketMQ 用事务消息和延迟消息弥补了 Kafka 的不足，对电商订单场景更友好。

**Q2: PageCache 满了怎么办？**
→ Linux 使用 LRU 算法淘汰冷数据。如果热点数据占满 PageCache，磁盘 I/O 会急剧上升，此时增加机器内存或减少 Kafka JVM 堆大小（给系统留更多 PageCache）。

**Q3: Kafka 分区数是不是越多越好？**
→ 不是。分区数越多：① Broker 元数据压力增大（Controller 压力）；② 选举时间变长；③ 打开文件句柄增加。建议评估实际消费并发需求后合理设置，并留一定余量。
