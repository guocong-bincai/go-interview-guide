# Redis 数据类型选型：String / Hash / List / Set / ZSet 何时用哪个

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：内存优化、数据结构特征、业务场景映射、序列化开销

## 面试官考察意图

面试官通常从这条链提问：

1. "你的系统用什么存用户信息？Redis String 还是 Hash？" → 看选型能力
2. "为什么不用 String 存整个对象序列化后的值？" → 引出内存效率和灵活性
3. "排行榜怎么实现？" → ZSet 的应用
4. "怎么统计每天 UV？" → Bitmap 或 Set 的交集运算

**区分度**在于能否说清每种数据结构的底层实现、适用场景和内存代价。

---

## 核心答案（30 秒版）

| 数据结构 | 适合场景 | 不适合场景 |
|---------|---------|-----------|
| **String** | KV 缓存、计数器、分布式锁、令牌桶 | 字段需要独立操作的复合对象 |
| **Hash** | 用户画像（姓名+年龄+邮箱）、配置管理 | 需要全局集合操作（去重/求交集） |
| **List** | 消息队列、时间线 Feed、分页游标 | 需要随机访问特定元素的位置 |
| **Set** | 标签系统、好友关注列表、权限判断 | 需要排序功能 |
| **ZSet** | 排行榜、定时任务延迟队列、加权路由 | 只需要无序集合的场景 |

---

## 深度展开

### 1. String vs Hash：经典选型陷阱

```
场景：存储用户信息，包含 name, age, email, avatar_url

❌ 错误做法：一个 String 存 JSON
   SET user:10001 '{"name":"张三","age":28,"email":"zhangsan@example.com"}'
   
   问题：
   - 改了 age 要读全量 JSON → 反序列化 → 改字段 → 序列化 → 写回
   - 无法直接原子修改单个字段
   - 内存浪费（每次都要存完整结构）

✅ 正确做法：Hash 按字段拆分
   HSET user:10001 name 张三
   HSET user:10001 age 28
   HSET user:10001 email zhangsan@example.com
   
   优势：
   - 只修改需要的字段（原子 HSET）
   - HMGET 可以只取几个字段，减少网络传输
   - 支持哈希压缩编码（ziplist），小对象极省内存
```

**Hash 压缩编码机制：**

```
当满足以下两个条件时，Hash 使用 ziplist（紧凑数组）而非 quicklist：
  ① 所有 field-value 的值都 < hash-max-ziplist-value (默认 64 bytes)
  ② field-number ≤ hash-max-ziplist-entries (默认 128)

ziplist 在内存中是连续的一块区域，比 linked list 节省约 50% 内存。
超过阈值后自动切换为 quicklist（多个 ziplist 用双向链表连接）。
```

```go
// Go 中使用 Hash 的正确姿势
import "github.com/redis/go-redis/v9"

func updateUserField(rdb *redis.Client, userID int64, field string, value string) error {
    // ✅ 单字段更新：原子操作，不读不写其他字段
    return rdb.HSet(ctx, fmt.Sprintf("user:%d", userID), field, value).Err()
}

func getUserSummary(rdb *redis.Client, userID int64) (map[string]string, error) {
    // ✅ 批量取多个字段：一次 RTT，只传输需要的字段
    return rdb.HMGet(ctx, fmt.Sprintf("user:%d", userID), "name", "age", "avatar_url").Result()
}
```

### 2. List 的典型应用场景

```bash
# 场景一：消息队列（简单版）
LPUSH task_queue "process_order:order_123"     # 生产者
BRPOP task_queue 5                              # 消费者阻塞式读取

# 场景二：Feed 流（发贴 + 关注拉取）
LPUSH feed:user:10001 "{type:'post', id: 5001}"  # 发帖
# 用户 A 的关注列表
SMEMBERS follow:user:A     # → 10001, 10002, 10003
# 聚合多人的 Feed（Merge Sort 合并排序）
ZUNIONSTORE merged_feed 3 feed:user:10001 feed:user:10002 feed:user:10003
SCORE each message timestamp

# 场景三：分页游标
LRANGE timeline:start:offset:end   # LIMIT offset, count
```

**⚠️ List 的坑：** LPUSH/RPOP 是 O(1)，但 LINDEX/O(1) 只对头部尾部高效。LINSERT 和 LREM 对大 List 是 O(N)。

### 3. Set vs ZSet：社交场景对比

```
场景一：共同好友
   SINTER user:100:fans other_user:101:fans
   → 返回两人的粉丝交集（潜在好友推荐）

场景二：热搜排行
   ZINCRBY hot_search 1 "Go语言"      # 每次搜索 +1
   ZRANGEBYSCORE hot_search 1000 -1 WITHSCORES LIMIT 0 10
   → 获取 TOP 10 热词

场景三：限时秒杀库存
   # 每个 SKU 一个 ZSet
   ZADD flash_sale:sku:1001 <timestamp+60s> order_id_1
   ZADD flash_sale:sku:1001 <timestamp+60s> order_id_2
   # 到时间自动过期（Sorted Set 天然带 score = 到期时间）
```

### 4. 特殊数据结构实战

```bash
# Bitmap：每日签到/活跃用户统计
# ⚠️ 前提：用户 ID 不能太稀疏，否则内存浪费
BITFIELD user:active_bits GET unsigned 0 OVERFLOW ABT INCRBY 0 1
# → 设置/增加第 N 个用户的活跃标记

# HyperLogLog：UV 统计（百万级日活精确到 ~0.81% 误差）
PFADD daily_uv:2026-09-10 user_123 user_456 user_789
PFCOUNT daily_uv:2026-09-10
→ 返回 ~12,345（实际值可能有 ±80 的误差）

# Geo：附近商家查询
GEOADD restaurants 116.404 39.915 "店A"
GEORADIUS restaurants 116.404 39.915 5 km WITHDIST ASC LIMIT 10
→ 返回周边 5km 内最近 10 家店
```

---

## 高频追问

### Q1：Hash 的 ziplist 什么时候会扩容成 quicklist？

```
触发条件（任一满足即扩容）：
1. 新增一个 field-value，其长度 > hash-max-ziplist-value（默认 64B）
2. Hash 中的 field 数量 > hash-max-ziplist-entries（默认 128）
3. 手动强制转换：HSET user:10001 __force_expand__ 1

扩容方向是单向的（ziplist → quicklist），不会缩回去。
所以建表时要预估最大可能的 field 数量和单字段大小。
```

### Q2：Bitmap 存 1000 万用户每天签到要花多少内存？

```
计算：
- 1000 万用户 = 10^7 bits
- 1 bit per day
- 假设保留 1 年 = 365 天
- 总位数 = 10^7 × 365 = 3.65 × 10^9 bits
- 内存 = 3.65 × 10^9 / 8 / 1024 / 1024 ≈ 438 MB

结论：对于中等规模应用可行，超大型可考虑分片（按月份或区段）
```

---

## 延伸阅读

- [Redis Data Structures Deep Dive](https://redis.io/docs/latest/develop/data-types/)
- [Redis Internals: How SDS Works](https://redis.io/blog/how-redis-uses-sds-for-string-storage/)
- [Redis Memory Optimization Guide](https://redis.io/docs/latest/operate/rs/tutorials/memory_optimization/)
