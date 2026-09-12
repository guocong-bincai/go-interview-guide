# 点赞系统设计（计数分片 + 去重 + 异步落库）

> 考察频率：★★★★☆  难度：★★★★☆  优先级：P1
> 关键词：计数分片、热点 Key、Bitmap 去重、MQ 削峰、缓存对账、大 Key 拆分

## 核心问题

| 问题 | 描述 |
|------|------|
| 高频写 | 爆款内容一秒钟几万次点赞，DB 直接被打穿 |
| 重复点赞 | 同一用户反复点赞/取消，如何保证计数准确且幂等 |
| 热点 Key | 明星微博的点赞计数 Key 集中在单个 Redis 分片 |
| 读取快 | 首页要展示点赞数，必须毫秒级返回 |
| 一致性 | Redis 计数与 DB 计数长期不一致怎么办 |

---

## 一、整体架构

```
用户点赞 → 点赞服务 ─┬─ 1. Redis 去重（SET / Bitmap）  ── 失败直接返回
                     ├─ 2. 发送 MQ 消息（异步落库）
                     └─ 3. Redis INCR 计数（读走缓存，毫秒返回）

                                              ▼
                                       消费 MQ → MySQL 点赞明细表
                                              ▼
                                       定时对账 → 修正 Redis 计数
```

**核心思想：写路径走"Redis 快速判断 + MQ 异步落库"，读路径只走 Redis 计数。**

---

## 二、去重设计

### 方案 A：Redis Set（关系简单）

```go
// like:{contentID} 存点赞用户集合
func LikeWithSet(ctx context.Context, rdb *redis.Client, cid, uid string) (bool, error) {
    added, err := rdb.SAdd(ctx, "like:u:"+cid, uid).Result()
    if err != nil {
        return false, err
    }
    if added == 0 {
        return false, nil // 已经点过赞，幂等返回
    }
    rdb.Incr(ctx, "like:c:"+cid)
    publishLikeEvent(ctx, cid, uid) // 异步落库
    return true, nil
}
```

缺点：Set 内存开销大（每个用户 ID 都要存完整字符串），用户量大时很占内存。

### 方案 B：Bitmap（高效，适合 ID 连续的场景）

```go
// 每个用户一个位偏移：offset = userID
// SETBIT like:bitmap:{contentID} {userID} 1
func LikeWithBitmap(ctx context.Context, rdb *redis.Client, cid string, uid int64) (bool, error) {
    key := "like:bitmap:" + cid
    prev, err := rdb.SetBit(ctx, key, uid, 1).Result()
    if err != nil {
        return false, err
    }
    if prev == 1 {
        return false, nil // 该位本来就是 1，重复点赞
    }
    rdb.Incr(ctx, "like:c:"+cid)
    publishLikeEvent(ctx, cid, uid)
    return true, nil
}

// 统计点赞数（也天然支持 BITCOUNT 做聚合/对账）
func CountBitmap(ctx context.Context, rdb *redis.Client, cid string) int64 {
    n, _ := rdb.BitCount(ctx, "like:bitmap:"+cid, nil).Result()
    return n
}
```

| 方案 | 内存 | 适用 |
|------|------|------|
| Set | 大（~50 字节/人） | 用户量小、ID 稀疏 |
| Bitmap | 小（1 bit/人，1 亿用户约 12MB） | userID 位数固定且范围可控 |
| **HyperLogLog** | 极小（12KB） | 只关心总量 UV，允许 0.81% 误差 |

> 面试加分：**HyperLogLog 不能用于"是否点赞过"的判断**（它只能估基数，不能判断成员是否存在），去重必须用 Set 或 Bitmap。

---

## 三、计数分片（解决热点 Key）

明星内容点赞时，`like:c:{cid}` 单 Key 成为热点。做法与库存分段同源：**把计数拆成 N 个分片 Key，读时求和。**

```go
const counterShards = 16

// 写：随机选一个分片 INCR（写分散）
func IncrLikeCount(ctx context.Context, rdb *redis.Client, cid string) error {
    idx := rand.Intn(counterShards)
    return rdb.Incr(ctx, fmt.Sprintf("like:c:%s:%d", cid, idx)).Err()
}

// 读：把 N 个分片加起来（读聚合，可用 pipeline 并行）
func GetLikeCount(ctx context.Context, rdb *redis.Client, cid string) (int64, error) {
    pipe := rdb.Pipeline()
    cmds := make([]*redis.IntCmd, counterShards)
    for i := 0; i < counterShards; i++ {
        cmds[i] = pipe.Get(ctx, fmt.Sprintf("like:c:%s:%d", cid, i))
    }
    if _, err := pipe.Exec(ctx); err != nil && err != redis.Nil {
        return 0, err
    }
    var total int64
    for _, c := range cmds {
        if v, err := c.Int64(); err == nil {
            total += v
        }
    }
    return total, nil
}
```

- 写压力被 16 个分片摊平，单分片压力降到 1/16；
- 读时用 **Pipeline** 一次 RTT 取回 16 个值，性能可接受；
- 若要进一步优化读，可加一层"本地缓存 + 定时刷新"。

---

## 四、异步落库 + 幂等

```go
// 点赞明细表：UNIQUE(user_id, content_id) 保证同一用户同一内容只有一条
// INSERT IGNORE INTO likes(user_id, content_id, created_at) VALUES (?, ?, ?)
// 影响行数 0 == 重复，直接 ACK 掉消息（幂等）

func consumeLike(ctx context.Context, msg LikeEvent) error {
    // 1. 幂等落库
    _, err := db.ExecContext(ctx,
        `INSERT IGNORE INTO likes(user_id, content_id, created_at) VALUES (?,?,?)`,
        msg.UserID, msg.ContentID, msg.CreatedAt)
    if err != nil {
        return err // 返回错误触发 MQ 重试
    }
    // 2. 增量更新内容表的计数冗余列（可选，用于冷启动/恢复）
    _, err = db.ExecContext(ctx,
        `UPDATE contents SET like_count = like_count + 1 WHERE id = ?`, msg.ContentID)
    return err
}
```

**为什么加 `INSERT IGNORE` / 唯一索引**：MQ 是 at-least-once，消息可能重复投递，唯一索引是最可靠的幂等手段。

---

## 五、计数一致性对账

Redis 计数可能因 INCR 成功后 MQ 发送失败而虚高，或因 Redis 重启丢数据而虚低。

```go
// 每 5 分钟对账一批热门内容
func reconcile(ctx context.Context, cid string) {
    var dbCount int64
    _ = db.QueryRowContext(ctx,
        `SELECT COUNT(*) FROM likes WHERE content_id = ?`, cid).Scan(&dbCount)

    redisCount, _ := GetLikeCount(ctx, rdb, cid)
    if diff := redisCount - dbCount; diff != 0 && abs(diff) > 0 {
        // 以 DB 为准，回写 Redis（保留分片结构）
        resetCounter(ctx, cid, dbCount)
        log.Warn("like count drift", "cid", cid, "redis", redisCount, "db", dbCount)
    }
}
```

**对账策略：**
- 热内容（Top N）高频对账（5 分钟）；
- 普通内容低频对账（1 小时）或直接以 DB 为准懒加载；
- 差异超过阈值（如 1%）告警。

---

## 六、高频追问

### Q1：点赞数展示要"实时精确"吗？

不一定。绝大多数场景 Redis 计数作为展示值即可（最终一致）。**精确值以 DB 为准，只在管理后台/对账时使用**。如果业务要求"必须精确"，那就要接受更高延迟，走"Redis 计数 + 短事务"。

### Q2：取消点赞怎么处理？

对称操作：`SREM`/`SETBIT 0` 判断是否真的删掉了（Set 返回 1 / Bitmap 返回 1 才 DECR），再发"取消点赞"事件异步删 DB 记录，DB 侧 `DELETE FROM likes WHERE ...` 同样天然幂等。

### Q3：Bitmap 的 userID 很大怎么办（稀疏 ID）？

- 用 `offset = userID - minUserID` 的相对偏移压缩；
- 或者分桶：`bitmap:{cid}:{userID/1e6}`，每桶 100 万个位；
- ID 极度稀疏时退回 Set。

### Q4：Redis 挂了点赞服务怎么办？

- 降级：直接写 DB（QPS 会掉，需要限流保护）；
- 或者降级为"异步确认"：请求先入本地队列/返回"处理中"，Redis 恢复后再补；
- 绝不能因为 Redis 挂了就放开去重，会产生大量重复数据。

---

## 面试话术

**Q：设计一个点赞系统，说说你的思路。**

> 点赞的核心矛盾是**高频写 + 要去重 + 要计数**。我的方案是：写路径用 Redis 去重（用户量大用 Bitmap，稀疏用 Set），去重通过后再把"点赞事件"丢进 MQ 异步落库，同时 Redis 计数。读路径直接读 Redis 计数，毫秒返回。爆款内容的计数 Key 我会拆成 16 个分片，读的时候用 Pipeline 求和，解决热 Key。最后配一个定时对账，用 DB 的 `COUNT(*)` 校准 Redis 计数，保证最终一致。

**Q：为什么不用数据库的 `like_count = like_count + 1` 直接扛？**

> 单条热门内容的点赞写会集中在同一行上，InnoDB 行锁把并发串行化，几万 QPS 直接把连接池和磁盘 IO 打满。**热点行更新的正确解法是"把一行拆成多行/多个 Key"，用空间换并发**，这也是计数分片的本质。
