# 缓存与数据库一致性策略 + 多级缓存架构

> 考察频率：★★★★★  优先级：P0  阿里/字节必考
> 关键词：Cache-Aside、Write-Through、延迟双删、延时队列、本地缓存、Caffeine/Guava、一致性问题

---

## 核心答案（30 秒版）

| 方案 | 适用场景 | 一致性 | 性能 | 复杂度 |
|------|----------|--------|------|--------|
| **Cache-Aside** | 读多写少（最常见） | ⭐⭐⭐ 最终一致 | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Read-Through** | 客户端封装缓存层 | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Write-Through** | 写入一致性要求高 | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Write-Behind** | 允许短暂不一致 | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **延迟双删** | Cache-Aside 的兜底方案 | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |

**生产最佳实践**：Cache-Aside + 延迟双删作为兜底 + 本地二级缓存抗热点。

---

## 深度展开

### 1. 四种缓存更新策略对比

#### 1.1 Cache-Aside Pattern（旁路缓存）—— 面试重点

```
读：先查缓存 → 命中则返回 → 未命中查 DB → 写入缓存 → 返回
写：先删缓存 → 更新 DB → （可选）删除旧缓存
```

```go
// Cache-Aside 读操作
func (s *UserService) GetUser(id int64) (*User, error) {
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", id)

    // 1. 先查缓存
    cached, err := s.cache.Get(ctx, key).Result()
    if err == nil && cached != "" {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    // 2. 缓存未命中，查数据库
    user, err := s.db.GetUser(id)
    if err != nil || user == nil {
        return nil, ErrNotFound
    }

    // 3. 写入缓存
    data, _ := json.Marshal(user)
    s.cache.Set(ctx, key, data, 30*time.Minute)

    return user, nil
}

// Cache-Aside 写操作 —— 先删缓存再更新DB
func (s *UserService) UpdateUser(id int64, fields map[string]interface{}) error {
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", id)

    // 1. 先删除缓存（不是更新！）
    s.cache.Del(ctx, key)

    // 2. 更新数据库
    err := s.db.UpdateUser(id, fields)
    if err != nil {
        return err
    }

    // 3. 可选：再删一次旧缓存（兜底，防止并发问题）
    time.Sleep(50 * time.Millisecond)
    s.cache.Del(ctx, key)

    return nil
}
```

**为什么 Cache-Aside 是"先删缓存"而不是"更新缓存"？**

```
如果先更新缓存，可能出现以下竞争：

线程A更新DB ───────────────────────────→ DB已更新
         ↓
线程B读缓存 ──→ 读到旧值（还没被删）
         ↓
线程A删缓存 ───────────────────────────→ 缓存没了
         ↓
线程C读缓存 ──→ 未命中，从DB读到新值并写入缓存 ✓

但是反过来（先更新缓存）：

线程A更新DB ───────────────────────────→ DB已更新为V2
         ↓
线程B写缓存 ──→ 将旧值V1写入缓存 ✗（覆盖了V2之前的值！）
         ↓
线程C读缓存 ──→ 读到V1（脏数据！）
```

**关键结论**：Cache-Aside 必须"先删缓存，后更新DB"，因为删除是幂等的，而更新会覆盖其他线程写入的新值。

#### 1.2 Write-Through（同步写回）

```
写：应用 → 缓存（由缓存负责同步到DB）→ 返回成功
读：同Cache-Aside
```

适用场景：缓存本身就是持久化存储（如 Redis），且对一致性要求高。

```go
// 使用 Redis Cluster 的 Pipeline 实现 Write-Through
func updateUserThroughPipeline(cache *redis.Client, userID int64, user *User) error {
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", userID)
    data, _ := json.Marshal(user)

    pipe := cache.Pipeline()
    pipe.Set(ctx, key, data, 30*time.Minute)
    // pipeline 执行时，缓存和DB的更新在同一RTT内完成
    _, err := pipe.Exec(ctx)
    return err
}
```

#### 1.3 Write-Behind（异步写回）

```
写：应用 → 缓存 → 立即返回 ←─ 缓存后台定时批量刷到DB
```

适用场景：消息日志、监控指标等允许短暂丢失的场景。**不适合金融/订单类数据**。

```go
// 异步写回示例
type WriteBehind struct {
    cache     *redis.Client
    db        *sql.DB
    queue     chan cacheEntry
    flushMs   int64
}

type cacheEntry struct {
    key      string
    value    []byte
    updatedAt time.Time
}

func (w *WriteBehind) StartFlushLoop() {
    ticker := time.NewTicker(time.Duration(w.flushMs) * time.Millisecond)
    go func() {
        for range ticker.C {
            entries := w.drainQueue()
            w.batchWriteToDB(entries)
        }
    }()
}

func (w *WriteBehind) drainQueue() []cacheEntry {
    // 非阻塞方式取所有待刷新条目
    var entries []cacheEntry
    for {
        select {
        case e := <-w.queue:
            entries = append(entries, e)
        default:
            return entries
        }
    }
}
```

#### 1.4 Read-Through（自动读取）

缓存组件自动处理"缓存未命中时查DB"的逻辑，应用代码无感知。Go 生态中可用 `bigcache`/`freecache` 结合中间件实现。

### 2. Cache-Aside 的一致性问题分析（高频面试题）

#### 2.1 经典竞态问题

```
时间线：
T1: 线程A 删除缓存
T2: 线程B 读缓存未命中 → 开始读DB
T3: 线程A 更新DB（新值 V2）
T4: 线程B 从DB读到旧值 V1 → 写入缓存 ✗（缓存又被旧值污染了！）
T5: 线程C 读缓存 → 拿到 V1（脏数据）
```

这就是为什么需要"延迟双删"作为兜底：

```go
// 延迟双删
func UpdateUserWithDelayDelete(userID int64, fields map[string]interface{}) error {
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", userID)

    // 第1次删除
    s.cache.Del(ctx, key)

    // 更新数据库
    err := s.db.UpdateUser(userID, fields)
    if err != nil {
        return err
    }

    // 休眠一个极短的间隔，让读请求完成
    time.Sleep(50 * time.Millisecond)

    // 第2次删除 —— 清理可能已经被写入的旧缓存
    s.cache.Del(ctx, key)

    // 可选：通过MQ发送延迟任务再次删除（最终保证）
    s.publishDelayedDelete(key, 1*time.Second)

    return nil
}
```

#### 2.2 用 Canal 做实时同步（终极方案）

```
┌──────┐   binlog   ┌─────────┐   同步   ┌────────┐
│ MySQL ├───────────→│ Canal   ├─────────→│ Redis  │
│      │            │ (Binlog │          │        │
│      │            │ 订阅者) │          │        │
└──────┘            └─────────┘          └────────┘
                              ↑              ↓
                          监听变更       自动更新缓存
                          发送到 MQ/Kafka
```

Canal 模拟 MySQL slave，解析 binlog 后推送变更。配合 MQ 可以实现可靠的数据同步：

```go
// Canal + Kafka 方案
type CanaSyncHandler struct {
    kafkaProducer *sarama.AsyncProducer
    cacheClient   *redis.Client
}

func (h *CanaSyncHandler) OnChange(event canal.DataChangeEvent) {
    key := fmt.Sprintf("%s:%d", event.Table, event.PrimaryKeyValue)
    message := &KafkaMessage{
        Table:   event.Table,
        Op:      event.Op.String(),    // INSERT/UPDATE/DELETE
        PK:      event.PrimaryKeyValue,
        Timestamp: time.Now().UnixNano(),
    }

    payload, _ := json.Marshal(message)
    h.kafkaProducer.Input() <- &sarama.ProducerMessage{
        Topic: "canal-changes",
        Value: sarama.ByteEncoder(payload),
    }
}

// 消费者端：根据binlog类型更新Redis
func syncFromKafka(msg *sarama.ConsumerMessage) {
    var change KafkaMessage
    json.Unmarshal(msg.Value, &change)

    switch change.Op {
    case "UPDATE":
        // 重新查询最新值写入缓存
        newValue := queryLatest(change.Table, change.PK)
        cache.Set(ctx, makeKey(change.Table, change.PK), newValue, TTL)
    case "DELETE":
        cache.Del(ctx, makeKey(change.Table, change.PK))
    }
}
```

### 3. 多级缓存架构

#### 3.1 三层缓存模型

```
                    ┌─────────────┐
                    │  CDN        │  静态资源/页面级缓存
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  本地缓存    │  Caffeine/Guava/SyncMap
                    │  L1 ~几十ms  │  无网络开销
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Redis集群   │  L2 ~1ms
                    │  分布式的   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  MySQL/TiDB │  最终数据源
                    └─────────────┘
```

```go
type MultiLevelCache struct {
    local  *sync.Map       // Go sync.Map 做 L1 本地缓存
    redis  *redis.Client  // Redis 做 L2 远程缓存
    db     *sql.DB        // MySQL 做数据源
}

// 三级读操作流程
func (m *MultiLevelCache) Get(key string) (string, error) {
    ctx := context.Background()

    // L1: 本地缓存（最快，零网络开销）
    if val, ok := m.local.Load(key); ok {
        return val.(string), nil
    }

    // L2: Redis 缓存
    val, err := m.redis.Get(ctx, key).Result()
    if err == nil {
        // 同时回填到 L1 本地缓存
        m.local.Store(key, val)
        return val, nil
    }

    // L3: 数据库
    row := m.db.QueryRowContext(ctx, "SELECT value FROM cache_table WHERE key=$1", key)
    var result string
    if err := row.Scan(&result); err != nil {
        return "", err
    }

    // 回填 L2
    m.redis.Set(ctx, key, result, 30*time.Minute)
    // 回填 L1
    m.local.Store(key, result)

    return result, nil
}

// 写操作：只更新 DB + Redis，不主动清 L1（等 L1 TTL 自然过期）
func (m *MultiLevelCache) Set(key, value string, ttl time.Duration) error {
    ctx := context.Background()
    err := m.db.ExecContext(ctx, "INSERT INTO cache_table (key, value) VALUES ($1, $2) ON CONFLICT(key) DO UPDATE SET value=$2", key, value)
    if err != nil {
        return err
    }
    m.redis.Set(ctx, key, value, ttl+5*time.Second) // L2 TTL 略长于逻辑TTL
    return nil
}
```

#### 3.2 多级缓存的一致性问题

```
问题：L1 和本地进程绑定，无法像 Redis 那样全局失效

解决方案 A：被动过期（推荐）
  - L1 TTL 设置较短（如 5~30 秒）
  - 写操作只删 Redis，L1 等过期自然淘汰
  - 代价：短时间内可能有 5~30 秒的不一致窗口

解决方案 B：发布-订阅通知
  - Redis SetEx 触发 Pub/Sub 事件
  - 各节点收到通知后清除本地缓存对应 key
  - 缺点：Pub/Sub 不可靠，有丢消息可能

解决方案 C：版本号/时间戳比较
  - 每个值带版本号，L1 存储时记录版本
  - 每次读取时检查 Redis 中的版本是否更高
  - 如果高了就替换本地缓存
  - 缺点：额外 RTT
```

### 4. 缓存穿透/击穿/雪崩 vs 一致性问题的区别

很多候选人会把"缓存三大问题"和"一致性策略"搞混：

| 概念 | 本质 | 解决方案 |
|------|------|----------|
| **缓存穿透** | 永远查不到的 key 打到 DB | 布隆过滤器 / 缓存空值 |
| **缓存击穿** | 热点 key 过期瞬间大量请求打到 DB | 互斥锁 / 逻辑过期 |
| **缓存雪崩** | 大量 key 同时过期或 Redis 挂了 | 随机 TTL / 集群部署 |
| **一致性冲突** | 缓存和 DB 数据不一致 | 延迟双删 / Canal / 重试 |

### 5. 面试话术

**Q：缓存和数据库如何保证一致性？**

> 最常用的是 Cache-Aside：读的时候先查缓存，没有就查 DB 然后写回缓存；写的时候先更新 DB 再删除缓存。但这样在并发情况下，可能有一个线程还没读完旧值，另一个线程已经把缓存删了然后写了新值，导致旧的被写回去。所以生产上一般加一层"延迟双删"兜底——更新完 DB 后 sleep 50ms 再删一次缓存，或者通过 MQ 发一条延迟消息在几秒后再删。对于强一致性场景就用 Canal 同步 binlog。

**Q：什么时候用多级缓存？**

> 当单台 Redis 扛不住读 QPS 时用。通常用 sync.Map 做应用侧 L1，TTL 设得短一些（5~30秒）。Redis 作为 L2。写操作只更新 L2 和数据源，L1 等过期自然淘汰。这样可以在不引入复杂分布式缓存协调机制的情况下，把热点数据的 P99 延迟压到微秒级。代价是有短暂的 stale read 窗口，但对大多数业务来说可接受。
