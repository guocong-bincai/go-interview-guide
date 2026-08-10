# 缓存三大问题：穿透/击穿/雪崩 + 代码实现

> 面试频率：★★★★★  考察角度：缓存原理、Go 实现、解决方案对比

---

## 1. 缓存穿透

### 1.1 原理

```
查询一个根本不存在的数据（如 id=-1）
→ 缓存查不到
→ 数据库也查不到
→ 每次请求都打到数据库
→ 数据库压力大甚至被打挂
```

### 1.2 解决方案

**方案 A：缓存空值**

```go
func GetUserCache(client *redis.Client, userID int64) (*User, error) {
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", userID)

    // 先查缓存
    cached, err := client.Get(ctx, key).Result()
    if err == nil {
        if cached == "NULL" {
            return nil, nil // 缓存的空值
        }
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    // 缓存没有，查数据库
    user, err := db.QueryUser(userID)
    if err != nil {
        return nil, err
    }

    if user == nil {
        // 缓存空值，防止穿透
        client.Set(ctx, key, "NULL", 5*time.Minute)
        return nil, nil
    }

    // 缓存结果
    data, _ := json.Marshal(user)
    client.Set(ctx, key, data, 30*time.Minute)
    return user, nil
}
```

**方案 B：BloomFilter（布隆过滤器）**

```go
package cache

import (
    "github.com/bits-and-blooms/bloom/v3"
)

type CacheWithBloom struct {
    redis  *redis.Client
    bloom  *bloom.BloomFilter
    prefix string
}

func NewCacheWithBloom(redisAddr string, expectedItems int64) *CacheWithBloom {
    c := &CacheWithBloom{
        redis:  redis.NewClient(&redis.Options{Addr: redisAddr}),
        bloom:  bloom.NewWithEstimates(uint(expectedItems), 0.01), // 1% 误判率
        prefix: "cache:bloom:",
    }
    return c
}

func (c *CacheWithBloom) Set(key string) {
    c.bloom.Add([]byte(key))
}

func (c *CacheWithBloom) MayExist(key string) bool {
    return c.bloom.Test([]byte(key))
}

func (c *CacheWithBloom) Get(key string) (string, error) {
    // BloomFilter 判断存在 → 可能存在，尝试查缓存/DB
    if !c.MayExist(key) {
        return "", nil // 一定不存在，直接返回
    }
    return c.redis.Get(context.Background(), c.prefix+key).Result()
}
```

---

## 2. 缓存击穿

### 2.1 原理

```
热点 key 过期瞬间
→ 大量并发请求同时涌入
→ 缓存查不到，都去查数据库
→ 数据库被打挂
```

### 2.2 解决方案

**方案 A：互斥锁（Mutex）**

```go
func GetUserWithLock(client *redis.Client, db *sql.DB, userID int64) (*User, error) {
    ctx := context.Background()
    key := fmt.Sprintf("user:%d", userID)

    // 先查缓存
    data, err := client.Get(ctx, key).Result()
    if err == nil && data != "NULL" {
        var user User
        json.Unmarshal([]byte(data), &user)
        return &user, nil
    }

    // 互斥锁：只有一个请求去查数据库
    lockKey := fmt.Sprintf("lock:user:%d", userID)
    lock, err := client.SetNX(ctx, lockKey, "1", 10*time.Second).Result()
    if !lock {
        // 没拿到锁，等一会儿再试缓存
        time.Sleep(50 * time.Millisecond)
        return GetUserWithLock(client, db, userID) // 递归重试
    }
    defer client.Del(ctx, lockKey)

    // 拿到锁，查数据库
    user, err := db.QueryUser(userID)
    if user != nil {
        d, _ := json.Marshal(user)
        client.Set(ctx, key, d, 30*time.Minute)
    } else {
        client.Set(ctx, key, "NULL", 5*time.Minute)
    }
    return user, nil
}
```

**方案 B：热点数据永不过期 + 异步更新**

```go
// 用 background refresh 替代过期
type AsyncCache struct {
    redis *redis.Client
}

func (ac *AsyncCache) Get(key string) (string, error) {
    val, err := ac.redis.Get(context.Background(), key).Result()
    if err == redis.Nil {
        // 缓存不存在，异步加载
        go ac.refreshAsync(key)
        return "", nil
    }
    return val, err
}

func (ac *AsyncCache) refreshAsync(key string) {
    // 从 DB 加载数据，更新缓存
    data := loadFromDB(key)
    ac.redis.Set(context.Background(), key, data, 0) // TTL=0 表示永不过期
}
```

---

## 3. 缓存雪崩

### 3.1 原理

```
大量 key 同时过期
→ 同一时刻大量请求都查缓存（miss）
→ 全部打到数据库
→ 数据库扛不住
```

### 3.2 解决方案

**方案 A：随机过期时间**

```go
// 过期时间加随机抖动，避免同时过期
func CacheWithJitter(key string, data interface{}, baseTTL time.Duration) {
    jitter := time.Duration(rand.Intn(int(baseTTL/2)))
    ttl := baseTTL + jitter
    client.Set(ctx, key, data, ttl)
}

// 使用
CacheWithJitter("user:123", user, 30*time.Minute)  // 实际 TTL: 30~45分钟
```

**方案 B：多级缓存**

```go
// L1 (本地缓存) → L2 (Redis) → DB
type TwoLevelCache struct {
    local *ccache.Cache    // localcache（进程内）
    redis *redis.Client
    ttl   time.Duration
}

func (tl *TwoLevelCache) Get(key string) (string, error) {
    // L1 查本地
    if val, ok := tl.local.Get(key); ok {
        return val.(string), nil
    }

    // L2 查 Redis
    val, err := tl.redis.Get(context.Background(), "cache:"+key).Result()
    if err == nil {
        tl.local.Set(key, val, tl.ttl) // 回填 L1
        return val, nil
    }

    return "", redis.Nil
}
```

**方案 C：Redis 高可用 + 限流**

```go
// Redis Cluster + 应用层限流兜底
func GetWithRateLimit(key string) (string, error) {
    // 令牌桶限流
    if !limiter.Allow() {
        // 限流触发，返回降级数据或友好错误
        return "", errors.New("service overloaded")
    }
    return getFromCache(key)
}
```

---

## 4. 缓存与数据库双写一致性

### 4.1 Cache Aside（最常用）

```go
// 读：Cache Aside
func ReadUser(userID int64) (*User, error) {
    // 先读缓存
    data, _ := redis.Get(ctx, "user:"+strconv.FormatInt(userID, 10)).Result()
    if data != "" {
        var u User
        json.Unmarshal([]byte(data), &u)
        return &u, nil
    }
    // 缓存没有，读数据库
    u, err := db.QueryUser(userID)
    if err != nil {
        return nil, err
    }
    // 回填缓存
    d, _ := json.Marshal(u)
    redis.Set(ctx, "user:"+strconv.FormatInt(userID, 10), d, 30*time.Minute)
    return u, nil
}

// 写：先写数据库，再删缓存（不是更新缓存）
func WriteUser(userID int64, name string) error {
    // 1. 写数据库
    err := db.UpdateUser(userID, name)
    if err != nil {
        return err
    }
    // 2. 删除缓存（不是更新），下次读自然会填充
    redis.Del(ctx, "user:"+strconv.FormatInt(userID, 10))
    return nil
}
```

**为什么是删除缓存而不是更新？**
> 更新缓存可能产生脏数据（如两个请求同时更新 A 和 B，各自先更新缓存再写 DB）；删除缓存是幂等的，更安全。

### 4.2 延时双删（解决删除缓存后、DB写入前 的脏读）

```go
func WriteUserWithDoubleDelete(userID int64, name string) error {
    // 1. 先删缓存
    redis.Del(ctx, "user:"+strconv.FormatInt(userID, 10))

    // 2. 写数据库
    db.UpdateUser(userID, name)

    // 3. 延迟 500ms 再删一次（等缓存被其他请求重新填充后再删）
    go func() {
        time.Sleep(500 * time.Millisecond)
        redis.Del(ctx, "user:"+strconv.FormatInt(userID, 10))
    }()

    return nil
}
```

---

## 5. 三大问题对比总结

| 问题 | 现象 | 原因 | 核心解决 |
|------|------|------|---------|
| 缓存穿透 | 查询不存在的数据，打穿 DB | 恶意/错误查询 | BloomFilter / 缓存空值 |
| 缓存击穿 | 热点 key 过期瞬间大量打到 DB | 单 key 过期的瞬间并发 | 互斥锁 / 永不过期 |
| 缓存雪崩 | 大量 key 同时过期 | 统一 TTL / Redis 宕机 | 随机 TTL / 多级缓存 / 高可用 |

---

## 6. 面试高频追问

**Q：为什么写数据库时删除缓存而不是更新缓存？**
> 更新缓存可能导致并发场景下的数据不一致。例如：线程 A 更新 DB（name=Tom），线程 B 更新 DB（name=Jerry），如果各自更新缓存，可能因为时序问题导致最终缓存是 Tom 而不是 Jerry。删除缓存是幂等的，更安全。

**Q：如何保证 Redis 和 MySQL 的最终一致性？**
> 1. Cache Aside + 延时双删（延迟 500ms 再删一次）；2. Canal 订阅 MySQL binlog 异步更新缓存；3. 强一致场景直接不用缓存，或用分布式事务。

**Q：布隆过滤器误判率怎么算？**
> m/n = -ln(ε) / ln(2)^2，其中 m 为 bit 位数，n 为元素个数，ε 为误判率。10 亿数据、1% 误判率只需要约 1GB 内存。

**Q：Redis 挂了怎么办？**
> 1. 主从自动切换（哨兵/集群）；2. 限流保护数据库；3. 本地缓存兜底（热点数据）；4. 监控告警第一时间响应。
