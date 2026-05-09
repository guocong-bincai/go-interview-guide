# Redis 常见数据类型使用场景：String / Hash / List / Set / ZSet / HyperLogLog / Bitmap / Geo

> 考察频率：★★★★☆  优先级：P0（高频基础题，逢面必问）

---

## 面试官考察意图

这道题考察候选人对 Redis 各数据类型的**实战应用能力**。
初级只知道"String 存字符串，ZSet 做排行榜"，高级要能讲清楚**每种类型背后的编码选择、业务选型、以及为什么不用其他方案**。同时，面试官也会追问"如果数据量大了怎么办"，考察候选人对 Redis 局限性和横向扩展方案的理解。

---

## 核心答案（30 秒版）

| 数据类型 | 典型场景 | 核心命令 |
|----------|----------|----------|
| **String** | 缓存、计数器、分布式锁、Session | SET/GET/INCR/SETNX |
| **Hash** | 对象存储、配置缓存、购物车 | HSET/HGET/HGETALL |
| **List** | 消息队列（轻量）、最新 N 个、栈/队列 | LPUSH/LPOP/LRANGE |
| **Set** | 标签、好友关系、去重 | SADD/SMEMBERS/SINTER |
| **ZSet** | 排行榜、延时队列、有序集合 | ZADD/ZRANGE/ZINCRBY |
| **HyperLogLog** | UV 统计（日活/周活/月活） | PFADD/PFCOUNT |
| **Bitmap** | 签到、状态位、连续活跃 | SETBIT/GETBIT/BITOP |
| **Geo** | LBS、附近的人/店 | GEOADD/GEORADIUS |

**选型原则：** 简单 String 能搞定的不用 Hash，需要排序的用 ZSet，统计类用 HyperLogLog/Bitmap。

---

## 深度展开

### 1. String：缓存 / 计数器 / 分布式锁

#### 1.1 缓存场景

```go
// Go 代码：缓存穿透防护
func getUser(conn *redis.Client, userID string) (*User, error) {
    // 1. 先查 Redis
    cached, err := conn.Get(ctx, "user:"+userID).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }
    if err != redis.Nil {
        return nil, err
    }

    // 2. 缓存不存在，查 DB
    user, err := db.QueryUser(userID)
    if err != nil {
        return nil, err
    }

    // 3. 回填缓存（防击穿：分布式锁或逻辑过期）
    data, _ := json.Marshal(user)
    conn.Set(ctx, "user:"+userID, data, 1*time.Hour)
    return user, nil
}
```

**为什么不用 Hash 存用户对象？**
- 单个字段频繁读写 → String 序列化更灵活
- 整个对象读取 → Hash 的 HGETALL 更高效（减少反序列化）

#### 1.2 计数器

```bash
# 页面 UV 计数
INCR page:view:20260109:video_123
INCR page:view:20260109:article_456

# 分布式限流
INCR rate:limit:user:12345
EXPIRE rate:limit:user:12345 1
# 超过阈值返回错误
```

#### 1.3 分布式锁（SETNX）

```go
// 正确姿势：SETNX + EXPIRE 原子化（Lua 或 SET NX EX）
func acquireLock(conn *redis.Client, lockKey string, ttl time.Duration) bool {
    // 方式 1：SET NX EX（推荐，原子操作）
    ok, err := conn.SetNX(ctx, lockKey, "locked", ttl).Result()
    if ok {
        return true
    }

    // 方式 2：Lua 保证原子性
    script := `
        if redis.call("exists", KEYS[1]) == 0 then
            redis.call("set", KEYS[1], ARGV[1], "EX", ARGV[2])
            return 1
        else
            return 0
        end
    `
    result, _ := conn.Eval(script, []string{lockKey}, "locked", int(ttl.Seconds())).Int()
    return result == 1
}
```

**坑：** 不能先 SETNX 再 EXPIRE（不是原子的，如果进程崩溃会永久锁住）。

### 2. Hash：对象 / 配置 / 购物车

#### 2.1 对象存储（替代 String 序列化）

```bash
# String 存储（需序列化/反序列化）
SET user:1001 '{"name":"张三","age":30,"city":"北京"}'

# Hash 存储（直接按字段操作）
HSET user:1001 name "张三" age 30 city "北京"
HGET user:1001 name        # → "张三"
HINCRBY user:1001 age 1    # → 31
HGETALL user:1001          # 获取所有字段
```

**Hash 适用场景：**
- 对象的多个字段需要**独立读写**
- 不需要序列化/反序列化
- 字段数量有限（< 512 个，否则退化为 hashtable）

**注意：Hash 的小数据优化**
```bash
# 字段数 < 512 且所有值 < 64 字节时，用 ziplist 编码（紧凑省内存）
# 超过阈值自动转为 hashtable（O(1) 但内存开销大）
```

#### 2.2 购物车场景

```go
// 购物车：user:cart:{userID} → Hash
// field = 商品 ID，value = 数量
HSET cart:1001 product:123 2
HSET cart:1001 product:456 1
HINCRBY cart:1001 product:123 1  # 加数量
HGET cart:1001 product:123       # 查看数量
HDEL cart:1001 product:123        # 删除商品
HLEN cart:1001                    # 商品种类数
```

**对比 String（JSON）：**
- Hash：可以单独增减商品数量
- String（JSON）：需要完整反序列化、修改、序列化

### 3. List：消息队列 / 最新 N 个 / 栈

#### 3.1 轻量级消息队列（不推荐生产级）

```go
// 生产者
conn.LPush(ctx, "queue:orders", orderJSON)

// 消费者
result, err := conn.BRPop(ctx, 5*time.Second, "queue:orders").Result()
if err == nil {
    processOrder(result)
}
```

**为什么说不推荐生产级？**
- 没有消息确认机制（消费后不删除，可重复消费）
- 不支持多消费者组
- 无优先级队列

**生产级用 Stream（Redis 5.0+）：**
```bash
XADD mystream * field value
XREAD STREAMS mystream 0
XGROUP CREATE mystream mygroup 0
XREADGROUP GROUP mygroup consumer1 STREAMS mystream ">"
```

#### 3.2 最新 N 个内容

```go
// 微博 Timeline：最新 100 条
conn.LPush(ctx, "timeline:user:1001", postJSON)
conn.LTrim(ctx, "timeline:user:1001", 0, 99)  // 只保留最新 100 条
timeline := conn.LRange(ctx, "timeline:user:1001", 0, 99)
```

#### 3.3 栈（FILO）与队列（FIFO）

```bash
# 栈：LPUSH + LPOP（LIFO）
LPUSH stack a b c
LPOP stack  # → c

# 队列：LPUSH + RPOP（FIFO）
LPUSH queue a b c
RPOP queue  # → a

# 双端队列：LPUSH + RPOP 或 RPUSH + LPOP
```

### 4. Set：标签 / 好友关系 / 去重

#### 4.1 标签系统

```go
// 给文章打标签
SADD article:123:tags "Go" "Redis" "微服务"
SMEMBERS article:123:tags        # 获取所有标签
SISMEMBER article:123:tags "Go"  # 检查是否包含某标签

// 用户画像：喜欢"游戏"标签的用户
SINTER user:tags:gaming user:tags:20260109  # 交集：今天活跃的游戏用户
```

#### 4.2 好友关系

```go
// 关注/粉丝
SADD user:1001:following 1002 1003 1004
SADD user:1002:followers 1001

// 共同好友
SINTER user:1001:following user:1003:following  # → 共同关注

// 推荐可能认识
SUNIONSTORE temp user:1001:following user:1002:following
SDIFF temp user:1001:following  # 去除已关注的
```

#### 4.3 去重

```go
// 抽奖：每个用户只抽一次
isNew, _ := conn.SAdd(ctx, "lottery:20260109:participants", userID).Result()
if isNew == 0 {
    return errors.New("已参与")
}
```

### 5. ZSet：排行榜 / 延时队列 / 有序集合

#### 5.1 排行榜

```go
// 游戏排行榜：实时更新 + 查询 TopN
ZADD leaderboard:game:20260109 100 "player_A"
ZADD leaderboard:game:20260109 95 "player_B"
ZADD leaderboard:game:20260109 120 "player_C"

// 查询 Top 10
top10, _ := conn.ZRevRangeWithScores(ctx, "leaderboard:game:20260109", 0, 9).Result()

// 查询某玩家排名（O(log n)）
rank, _ := conn.ZRevRank(ctx, "leaderboard:game:20260109", "player_A").Result()

// 加分（活动奖励）
ZINCRBY leaderboard:game:20260109 50 "player_A"
```

**为什么不直接用 Set 存储？**
- Set 无序，无法排名
- 需要 sorted set 维护有序性

**生产问题：跨时段排行榜（如日榜、周榜、月榜）**

```go
// 日榜聚合到周榜：周日 00:00 执行
ZUNIONSTORE leaderboard:weekly:2026-W02 \
    leaderboard:daily:2026-W02-mon \
    leaderboard:daily:2026-W02-tue \
    leaderboard:daily:2026-W02-wed \
    leaderboard:daily:2026-W02-thu \
    leaderboard:daily:2026-W02-fri \
    leaderboard:daily:2026-W02-sat \
    leaderboard:daily:2026-W02-sun
```

#### 5.2 延时队列

```go
// 订单超时取消：ZSet + ZRANGEBYLEX + ZREM
// score = 过期时间戳
func addDelayTask(conn *redis.Client, taskID string, delay time.Duration) {
    expireAt := time.Now().Add(delay).Unix()
    conn.ZAdd(ctx, "delay:queue", &redis.Z{
        Score:  float64(expireAt),
        Member: taskID,
    })
}

// 消费者：扫描到期任务
func consumeDelayQueue(conn *redis.Client) {
    for {
        // 获取所有已到期的任务
        now := time.Now().Unix()
        tasks, _ := conn.ZRangeByScore(ctx, "delay:queue", "0", fmt.Sprintf("%d", now), redis.ZRangeByScore{Limit: 100}).Result()
        
        for _, task := range tasks {
            // 执行任务
            processTask(task)
            // 从队列移除
            conn.ZRem(ctx, "delay:queue", task)
        }
        time.Sleep(1 * time.Second)
    }
}
```

#### 5.3 有序集合的其他用法

```go
// 时间轴排序：按时间戳作为 score
ZADD user:feed:1001 1709123456 "post:123"
ZADD user:feed:1001 1709123678 "post:456"
ZRANGEBYSCORE user:feed:1001 1709123000 1709124000  // 时间范围内

// 权重排序：多维度综合排序
ZADD products:score 85.5 "product:A"  // 85.5 = 销量分*0.6 + 好评分*0.4
```

### 6. HyperLogLog：UV 统计

```go
// 日活统计：PFADD 不重复计数
conn.PFAdd(ctx, "uv:daily:20260109", userID1, userID2, userID3)
count, _ := conn.PFCount(ctx, "uv:daily:20260109").Result()
// 误差率约 0.81%，内存极低（每个 key 仅 12KB）
```

**为什么用 HyperLogLog 而不是 Set？**
- Set 存每个用户 ID： millions 用户 → 数 GB 内存
- HyperLogLog：固定 12KB，误差 < 1%
- 适合大流量场景（日活、UV、月活）

**注意：PFADD 返回 1 表示新增加入，0 表示已存在**

### 7. Bitmap：签到 / 状态位 / 连续活跃

```go
// 用户 2026 年 1 月签到记录
// key = "sign:2026-01:user:12345"，offset = 日期（1~31）
conn.SetBit(ctx, "sign:2026-01:user:12345", 8, 1)  // 1 月 9 日签到

// 查询某天是否签到
bit, _ := conn.GetBit(ctx, "sign:2026-01:user:12345", 8).Result()

// 统计本月签到天数
BITCOUNT sign:2026-01:user:12345

// 连续签到天数（BitOp + 滑动窗口）
BITOP AND temp sign:2026-01:user:12345 sign:2026-01:user:12345:prev
```

**适用场景：**
- 用户签到（1 bit/天，1 年仅 365 bits ≈ 46 bytes）
- 每日活跃状态（1 bit/用户，千万用户 ≈ 1.25MB）
- 连续活跃天数计算

**不适合场景：**
- 需要存储实际数据的（如用户资料）
- 数据量极小（用 String 更简单）

### 8. Geo：LBS 附近的人 / 店

```go
// 添加门店位置
conn.GeoAdd(ctx, "stores:beijing", &redis.GeoLocation{
    Name:      "朝阳店",
    Longitude: 116.3974,
    Latitude:  39.9093,
})

// 查询附近 5km 的门店
nearby, _ := conn.GeoRadius(ctx, "stores:beijing", 116.3974, 39.9093, &redis.GeoRadiusQuery{
    Radius:    5, // km
    Unit:      "km",
    WithCoord: true,
    WithDist:  true,
    Sort:      "ASC", // 按距离排序
}).Result()

for _, loc := range nearby {
    fmt.Printf("店名: %s, 距离: %.2fkm, 坐标: (%.4f, %.4f)\n",
        loc.Name, loc.Dist, loc.Longitude, loc.Latitude)
}
```

**Geo 底层实现：**
- 使用 GeoHash 将经纬度编码为 52 位整数
- 存储在 Sorted Set 中，score = 编码值
- `GEORADIUS` 实际是 `ZRANGEBYSCORE` + 距离计算

**限制：**
- 不支持多维度（只有经纬度）
- 跨城市搜索性能差（需要先分区）

---

## 高频追问

### Q1：String > 512MB 怎么办？

Redis String 最大 512MB。如果需要存大文件：
- **对象存储**：存 OSS/S3，Redis 只存 URL
- **分片存储**：将大文件切分为 10MB 每块，append only

### Q2：ZSet 和 List 的区别？

| 维度 | ZSet | List |
|------|------|------|
| 排序方式 | 按 score（权重）排序 | 按插入顺序 |
| 查排名 | O(log n) | O(n) |
| 查范围 | O(log n + m) | O(l + n) |
| 重复元素 | 不允许（score 相同则覆盖） | 允许重复 |
| 典型场景 | 排行榜、权重排序 | 时间轴、队列 |

### Q3：Redis 数据过期后会被立即删除吗？

不会。惰性删除（访问时检查）+ 定期删除（后台扫描）配合。如果过期 key 大量存在且不再访问，会占用内存直到触发 maxmemory 淘汰。

### Q4：HyperLogLog 可以合并统计吗？

可以。`PFMERGE destkey sourcekey1 sourcekey2`，适合多平台 UV 汇总（如 Web + App + 小程序）。

---

## 延伸阅读

- [Redis Data Types - redis.io](https://redis.io/docs/data-types/)
- [Redis Streams - redis.io](https://redis.io/docs/data-types/streams/)
- 《Redis 设计与实现》—— 黄健宏，数据结构章节