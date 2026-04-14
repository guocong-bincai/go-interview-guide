[🏠 首页](../../../README.md) · [🗤️ 数据库](../../README.md) · [🔴 Redis](../README.md)

---

# Redis 缓存三大问题：穿透 / 击穿 / 雪崩

## 面试官考察意图

这是 Redis 最高频的考题，几乎每家公司必问。
初级只能背"布隆过滤器解决穿透"，高级要能讲清楚**每种问题的本质、多种解决方案的 tradeoff、以及在生产中遇到时的具体处理过程**，最后能延伸到热 key、大 key 等衍生问题。

---

## 核心答案（30 秒版）

| 问题 | 本质 | 核心方案 |
|------|------|----------|
| **缓存穿透** | 查询不存在的数据，每次都打到 DB | 布隆过滤器 / 缓存空值 |
| **缓存击穿** | 热点 key 过期，瞬间大量请求打到 DB | 互斥锁 / 逻辑过期 |
| **缓存雪崩** | 大量 key 同时过期 / Redis 宕机 | 随机过期时间 / 熔断降级 / 集群高可用 |

---

## 深度展开

### 1. 缓存穿透（Cache Penetration）

**问题本质：** 请求查询的数据在缓存和数据库中都不存在，每次请求都穿透缓存直达 DB。
**攻击场景：** 恶意用户用不存在的 id（如负数、超大数）轮番请求，击垮数据库。

```
请求 id=-1
    │
    ▼
Redis 查询：MISS（不存在）
    │
    ▼
MySQL 查询：也不存在，返回空
    │
    ▼
不缓存空结果 → 下次同样的请求再次穿透
```

#### 方案一：缓存空值

```go
func GetUser(ctx context.Context, id int64) (*User, error) {
    key := fmt.Sprintf("user:%d", id)

    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        if val == "NULL" {
            return nil, ErrNotFound  // 命中空缓存，直接返回
        }
        var user User
        json.Unmarshal([]byte(val), &user)
        return &user, nil
    }

    // 查 DB
    user, err := db.QueryUser(id)
    if err == sql.ErrNoRows {
        // 缓存空值，设置较短 TTL（防止占用过多内存）
        rdb.Set(ctx, key, "NULL", 5*time.Minute)
        return nil, ErrNotFound
    }

    data, _ := json.Marshal(user)
    rdb.Set(ctx, key, data, 30*time.Minute)
    return user, nil
}
```

**优缺点：**
- 优点：实现简单，适合少量不存在 key 的场景
- 缺点：被恶意遍历时会大量占用 Redis 内存；如果 DB 后来写入了该数据，缓存空值期间数据不一致

#### 方案二：布隆过滤器（Bloom Filter）

```
原理：用多个哈希函数将数据映射到 bit 数组
查询时：若任意一个 bit 为 0 → 数据一定不存在
        若所有 bit 为 1 → 数据可能存在（存在假阳性）

空间极省：1亿数据，误判率 0.1% → 约 180MB
```

```go
// 使用 Redis 的 BF 模块（RedisBloom）或本地布隆过滤器
import "github.com/bits-and-blooms/bloom/v3"

var bf *bloom.BloomFilter

func init() {
    // 预期100万数据，误判率0.01%
    bf = bloom.NewWithEstimates(1_000_000, 0.0001)

    // 启动时从 DB 加载所有合法 id
    ids, _ := db.QueryAllUserIDs()
    for _, id := range ids {
        bf.AddString(strconv.FormatInt(id, 10))
    }
}

func GetUser(ctx context.Context, id int64) (*User, error) {
    key := strconv.FormatInt(id, 10)

    // 布隆过滤器判断：不存在则直接返回，不查 Redis 和 DB
    if !bf.TestString(key) {
        return nil, ErrNotFound
    }

    // 正常走缓存逻辑...
}
```

**优缺点：**
- 优点：内存效率极高，完全阻挡不存在 key 的请求
- 缺点：有误判率（合法 key 被误判为不存在）；数据删除时布隆过滤器不支持删除（需用 Counting Bloom Filter）；数据新增时需同步更新布隆过滤器

**生产选型：** 数据量大、有恶意攻击风险 → 布隆过滤器；数据量小、随机不存在 key → 缓存空值。

---

### 2. 缓存击穿（Cache Breakdown）

**问题本质：** 某个热点 key 过期，在它重建缓存的瞬间，大量并发请求同时穿透到 DB。
区别于穿透：击穿是**存在于 DB 中的数据**，只是缓存恰好过期。

```
热点 key "product:9527" 过期
    │
    ▼
10000 个并发请求同时：Redis MISS → 全部打到 DB
    │
    ▼
DB 被 10000 个查询压垮
```

#### 方案一：互斥锁（Mutex）

```go
var mu sync.Map  // 每个 key 一把锁

func GetProduct(ctx context.Context, id int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", id)

    // 先查缓存
    if val, err := rdb.Get(ctx, key).Result(); err == nil {
        var p Product
        json.Unmarshal([]byte(val), &p)
        return &p, nil
    }

    // 缓存 MISS，用 Redis SET NX 实现分布式互斥锁
    lockKey := key + ":lock"
    locked, err := rdb.SetNX(ctx, lockKey, 1, 10*time.Second).Result()
    if err != nil {
        return nil, err
    }

    if !locked {
        // 没拿到锁，短暂等待后重试（等待持锁者重建缓存）
        time.Sleep(50 * time.Millisecond)
        return GetProduct(ctx, id)  // 递归重试
    }
    defer rdb.Del(ctx, lockKey)

    // 拿到锁：double check 缓存（防止其他协程已重建）
    if val, err := rdb.Get(ctx, key).Result(); err == nil {
        var p Product
        json.Unmarshal([]byte(val), &p)
        return &p, nil
    }

    // 查 DB，重建缓存
    product, err := db.QueryProduct(id)
    if err != nil {
        return nil, err
    }
    data, _ := json.Marshal(product)
    rdb.Set(ctx, key, data, 30*time.Minute)
    return product, nil
}
```

**优缺点：**
- 优点：强一致性，DB 只被查询一次
- 缺点：等待锁期间性能下降；锁超时设置不当可能导致死锁或重复查询

#### 方案二：逻辑过期（不设 TTL，异步更新）

```go
type CacheItem struct {
    Data      interface{} `json:"data"`
    ExpireAt  int64       `json:"expire_at"`  // 逻辑过期时间（Unix 时间戳）
}

func GetProduct(ctx context.Context, id int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", id)

    val, err := rdb.Get(ctx, key).Result()
    if err != nil {
        // 缓存不存在，同步查 DB 并写入（首次加载）
        return loadAndCache(ctx, id)
    }

    var item CacheItem
    json.Unmarshal([]byte(val), &item)
    product := item.Data.(*Product)

    // 逻辑过期：返回旧数据，异步更新
    if time.Now().Unix() > item.ExpireAt {
        go func() {
            lockKey := key + ":lock"
            locked, _ := rdb.SetNX(ctx, lockKey, 1, 10*time.Second).Result()
            if !locked {
                return  // 已有其他协程在更新
            }
            defer rdb.Del(ctx, lockKey)
            loadAndCache(context.Background(), id)
        }()
    }

    return product, nil  // 立即返回旧数据，不等待更新
}
```

**优缺点：**
- 优点：请求永远不等待，性能最好（热点场景推荐）
- 缺点：有短暂数据不一致（返回旧数据）；需要预热（启动时提前加载数据）

---

### 3. 缓存雪崩（Cache Avalanche）

**问题本质：** 分两种情况：
1. **大量 key 同时过期**：批量导入时设置了相同 TTL
2. **Redis 实例宕机**：所有请求直接打到 DB

#### 方案一：随机过期时间（解决批量过期）

```go
func SetWithJitter(ctx context.Context, key string, val interface{}, baseTTL time.Duration) error {
    // 在基础 TTL 上加 ±20% 的随机抖动
    jitter := time.Duration(rand.Int63n(int64(baseTTL / 5)))
    if rand.Intn(2) == 0 {
        jitter = -jitter
    }
    ttl := baseTTL + jitter
    data, _ := json.Marshal(val)
    return rdb.Set(ctx, key, data, ttl).Err()
}

// 批量导入时，每个 key 的 TTL 在 24h ± 4.8h 范围内随机分布
// 过期时间离散，不会集中过期
```

#### 方案二：永不过期 + 后台定时刷新

```go
// 对于计算成本极高、更新不频繁的数据（如首页推荐列表）
// 不设过期时间，由定时任务周期性更新

func StartCacheRefresher() {
    ticker := time.NewTicker(10 * time.Minute)
    go func() {
        for range ticker.C {
            refreshHotData()
        }
    }()
}
```

#### 方案三：Redis 高可用（解决 Redis 宕机）

```
生产部署方案选型：

Sentinel（哨兵模式）：
  - 适合：读写分离，主从切换自动化
  - 切换时间：10~30秒，期间请求可能失败
  - 规模：数十 GB

Cluster（集群模式）：
  - 适合：数据量大（TB 级），高并发写
  - 数据分片：16384 个槽位
  - 单节点故障自动 failover

多级缓存：
  本地缓存（进程内）→ Redis → DB
  Redis 宕机时，本地缓存承接流量（需注意一致性）
```

```go
// 本地缓存兜底：使用 ristretto 或 groupcache
import "github.com/dgraph-io/ristretto"

var localCache *ristretto.Cache

func init() {
    localCache, _ = ristretto.NewCache(&ristretto.Config{
        NumCounters: 1e7,   // 1000万计数器
        MaxCost:     1<<30, // 1GB 本地缓存
        BufferItems: 64,
    })
}

func GetProductWithFallback(ctx context.Context, id int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", id)

    // L1: 本地缓存
    if val, ok := localCache.Get(key); ok {
        return val.(*Product), nil
    }

    // L2: Redis
    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        var p Product
        json.Unmarshal([]byte(val), &p)
        localCache.SetWithTTL(key, &p, 1, 1*time.Minute)
        return &p, nil
    }

    // L3: DB（Redis 宕机时兜底）
    if err != redis.Nil {
        // Redis 连接错误，直接查 DB
        return db.QueryProduct(id)
    }

    return nil, ErrNotFound
}
```

#### 方案四：熔断降级

```go
// 使用 hystrix 或 sentinel 在 DB 被打爆时熔断
import "github.com/afex/hystrix-go/hystrix"

hystrix.ConfigureCommand("db-query", hystrix.CommandConfig{
    Timeout:               1000,  // 1秒超时
    MaxConcurrentRequests: 100,   // 最大并发
    ErrorPercentThreshold: 50,    // 错误率超50%触发熔断
})

func GetProduct(ctx context.Context, id int64) (*Product, error) {
    var product *Product
    err := hystrix.Do("db-query", func() error {
        var err error
        product, err = db.QueryProduct(id)
        return err
    }, func(err error) error {
        // fallback：返回降级数据（如空数据或缓存中的旧数据）
        product = &Product{ID: id, Name: "服务繁忙，请稍后"}
        return nil
    })
    return product, err
}
```

---

## 三种问题对比总结

| | 穿透 | 击穿 | 雪崩 |
|---|---|---|---|
| **触发条件** | 查询不存在的 key | 热点 key 过期 | 大量 key 同时过期 / Redis 宕机 |
| **影响范围** | 特定不存在的 key | 特定热点 key | 大面积缓存失效 |
| **DB 压力** | 持续高压 | 瞬间高峰 | 大面积高压 |
| **核心方案** | 布隆过滤器 | 互斥锁 / 逻辑过期 | 随机 TTL / 高可用 / 多级缓存 |

---

## 高频追问

**Q：缓存穿透和缓存击穿的区别？**

- 穿透：key 在 DB 中**不存在**，每次都打 DB
- 击穿：key 在 DB 中**存在**，只是缓存过期，瞬间大量请求穿透

**Q：布隆过滤器的假阳性率如何控制？**

```
误判率 ≈ (1 - e^(-kn/m))^k
k = 哈希函数数量 = (m/n) * ln2
m = bit 数组大小
n = 元素数量

实践：n=1亿，误判率0.1% → m ≈ 1.44GB，k ≈ 10
    n=1亿，误判率1%   → m ≈ 958MB，k ≈ 7
```

**Q：缓存更新策略有哪些？哪种最推荐？**

| 策略 | 说明 | 推荐度 |
|------|------|--------|
| Cache Aside（旁路缓存） | 先更新 DB，再删缓存（不更新）| ⭐⭐⭐⭐⭐ 推荐 |
| Write Through | 写时同步更新缓存和 DB | ⭐⭐⭐ 一致性好但有开销 |
| Write Behind | 写缓存，异步刷 DB | ⭐⭐ 性能好但有丢数据风险 |

**为什么 Cache Aside 是删缓存而不是更新缓存？**
更新缓存有并发问题：A 先更新 DB→B 后更新 DB→B 先更新缓存→A 后更新缓存，导致缓存是旧值。
删缓存后下次读时重建，避免此问题。

**Q：删缓存后写 DB 前系统崩溃怎么办？**

这属于 Cache Aside 的"先删缓存再写 DB"的问题。
推荐顺序是**先写 DB 再删缓存**（数据库写成功才能算成功），即使删缓存失败，也只是短暂不一致，下次读会重建正确数据。
极端情况用**延迟双删**兜底：

```go
// 延迟双删：写 DB 前后各删一次缓存
rdb.Del(ctx, key)          // 第一次删
db.Update(...)             // 写 DB
time.AfterFunc(500*time.Millisecond, func() {
    rdb.Del(ctx, key)      // 延迟第二次删，清除写 DB 期间的脏缓存
})
```

---

## 延伸阅读

- [Redis 官方文档：Patterns](https://redis.io/docs/manual/patterns/)
- [缓存更新的套路](https://coolshell.cn/articles/17416.html)（陈皓）
- [布隆过滤器原理与实践](https://llimllib.github.io/bloomfilter-tutorial/)

---

**[← 上一篇：Redis 持久化机制](./02-persistence.md)** · **[下一篇：Redis 集群方案 →](./04-cluster.md)**
