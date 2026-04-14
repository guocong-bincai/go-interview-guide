# 分布式 ID

> 考察频率：★★★★☆  难度：★★★☆☆

## 核心需求

| 需求 | 说明 |
|------|------|
| 全局唯一 | 不同机器生成的 ID 不能重复 |
| 趋势递增 | 插入性能友好（B+ 树） |
| 高可用 | 不能有单点故障 |
| 高性能 | QPS 要高 |

---

## 方案对比

| 方案 | 趋势递增 | 高可用 | 性能 | 存储 |
|------|---------|-------|------|------|
| UUID | ❌ | ✅ | ✅ | ❌（无序） |
| 数据库自增 | ✅ | ❌（单点） | ❌ | ✅ |
| Snowflake | ✅ | ✅ | ✅ | ✅ |
| 百度 UidGenerator | ✅ | ✅ | ✅ | ✅ |
| 美团 Leaf | ✅ | ✅ | ✅ | ✅ |

---

## UUID

```go
import "github.com/google/uuid"

// 最简单，但无序、字符串、存储大
id := uuid.New().String() // "550e8400-e29b-41d4-a716-446655440000"
```

**问题**：
- 字符串，存储大（36 字节 vs 8 字节）
- 无序，索引不友好
- 可猜测，不安全

---

## 数据库自增

```sql
CREATE TABLE sequence (
    id BIGINT AUTO_INCREMENT PRIMARY KEY
);
```

```go
func nextID() (int64, error) {
    var id int64
    err := db.QueryRow("INSERT INTO sequence VALUES (NULL)").Scan(&id)
    return id, err
}
```

**问题**：
- 单点故障
- 写入是随机 I/O（页分裂）

---

## Snowflake（雪花算法）

### 原理

64 位结构：

```
+--------------------------------------------------------------------------+
| 1 bit  |   41 bit timestamp    |   10 bit machine   |    12 bit sequence   |
+--------+----------------------+--------------------+-----------------------+
|  符号   |      毫秒时间戳        |    机器 ID          |     每毫秒序列号        |
+--------+----------------------+--------------------+-----------------------+
```

- **时间戳**：41 位，约可用 69 年
- **机器 ID**：10 位，最多 1024 台机器
- **序列号**：12 位，每毫秒最多 4096 个 ID

### Go 实现

```go
type Snowflake struct {
    mu        sync.Mutex
    timestamp int64
    machineID int64
    sequence  int64
    epoch     int64 // 起始时间
}

func NewSnowflake(machineID int64) *Snowflake {
    return &Snowflake{
        machineID: machineID,
        epoch:     1609459200000, // 2021-01-01 毫秒时间戳
    }
}

func (sf *Snowflake) NextID() int64 {
    sf.mu.Lock()
    defer sf.mu.Unlock()

    now := time.Now().UnixMilli()

    if now == sf.timestamp {
        sf.sequence++
        if sf.sequence > 4095 {
            // 等待下一毫秒
            for now == sf.timestamp {
                now = time.Now().UnixMilli()
            }
            sf.sequence = 0
        }
    } else {
        sf.sequence = 0
        sf.timestamp = now
    }

    // 关键：ID = 时间戳差值左移 22 位 | 机器 ID 左移 12 位 | 序列号
    id := (now-sf.epoch)<<22 | sf.machineID<<12 | sf.sequence
    return id
}
```

### 问题

1. **时间回拨**：服务器时间回拨会导致 ID 重复
2. **单机依赖**：依赖本机时钟，不是真正的分布式

### 解决方案：时间回拨处理

```go
func (sf *Snowflake) NextID() int64 {
    sf.mu.Lock()
    defer sf.mu.Unlock()

    now := time.Now().UnixMilli()

    if now < sf.timestamp {
        // 时间回拨，使用备份方案
        sf.sequence++
        if sf.sequence > 4095 {
            return 0 // 或返回错误
        }
        // 使用上次时间 + sequence
        now = sf.timestamp
    } else if now == sf.timestamp {
        sf.sequence++
        if sf.sequence > 4095 {
            for now == sf.timestamp {
                now = time.Now().UnixMilli()
            }
            sf.sequence = 0
        }
    } else {
        sf.sequence = 0
    }

    sf.timestamp = now
    return (now-sf.epoch)<<22 | sf.machineID<<12 | sf.sequence
}
```

---

## 百度 UidGenerator

基于 Snowflake 的优化：
- 使用未来时间（5 秒内）避免等待
- 双 Buffer：异步预生成 ID

```go
// 关键思想：RingBuffer 预生成
type UidGenerator struct {
    ringBuffer *RingBuffer
    // ...
}

func (g *UidGenerator) Get() int64 {
    // 从 RingBuffer 取，异步补充
    return g.ringBuffer.Pop()
}
```

---

## 美团 Leaf

两种模式：
1. **Leaf-segment**：数据库号段模式，批量获取 ID 段
2. **Leaf-snowflake**：ZK 获取 workerID，解决时钟回拨

### Leaf-segment 原理

```go
// 服务端维护号段
type LeafSegment struct {
    maxID    int64
    step     int64
    current  int64
}

// 获取一批 ID
func (s *LeafSegment) GetBatch() []int64 {
    if s.current >= s.maxID {
        // 重新从数据库加载
        s.Refill()
    }
    ids := make([]int64, 0, 1000)
    for ; s.current < s.maxID; s.current++ {
        ids = append(ids, s.current)
    }
    return ids
}
```

**优势**：一次从数据库拿 1000 个 ID，数据库压力降低 1000 倍。

---

## 面试话术

**Q：Snowflake 时间回拨怎么解决？**

> 两种方案：一是使用 Leaf-snowflake，通过 ZK 分配 workerID，并记录上次时间戳，回拨时使用"回拨时间 + 序列号"生成；二是 UidGenerator 的双 Buffer 方案，异步预生成，不依赖实时时间。

**Q：为什么分布式 ID 要趋势递增？**

> B+ 树的插入性能。如果 ID 随机，每次插入可能触发页分裂和随机 I/O；如果 ID 递增，新数据追加到尾部，顺序写入，索引友好。

**Q：Snowflake 的 41 位时间戳为什么是 69 年？**

> 2^41 毫秒 = 2199023255552 毫秒 ≈ 2199023255 秒 ≈ 69.7 年。从 2021 年起可用到 2091 年。

---

## 总结

| 方案 | 适用场景 |
|------|---------|
| UUID | 不需要排序、本地生成 |
| 数据库自增 | 单机、低 QPS |
| Snowflake | 通用场景，推荐使用 |
| UidGenerator | 高并发，预生成优化 |
| Leaf | 需要强一致、高可用 |
