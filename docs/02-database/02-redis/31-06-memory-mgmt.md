[🏠 首页](../../../README.md) · [🗤️ 数据库](../../README.md) · [💾 Redis](../README.md)

---

# Redis 内存管理：碎片整理 / MULTI 回滚 / PubSub vs Stream

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：内存碎片、active defrag、MULTI/EXEC、Lua原子性、PubSub不可靠、Stream可靠消息

## 面试官考察意图

Redis 作为内存数据库，**内存问题就是它最大的敌人**。
这道题的面试链条通常是：

1. "Redis 是内存数据库，内存用完了怎么办？" → 引出淘汰策略
2. "用了淘汰策略是不是就万事大吉？" → 引出内存碎片
3. "你说事务保证原子性，那能回滚吗？" → 引出 Redis 事务的局限
4. "那做消息队列用什么？PubSub 可靠吗？" → 引出 Stream

**区分度**在于能否说清 active defragmentation 的底层原理（replace + migrate）、为什么 MULTI/EXEC 只有语法错误会整体拒绝而执行错误不会回滚，以及 Stream 对比 PubSub 到底多了哪些可靠性保障。

---

## 核心答案（30 秒版）

| 问题 | 一句话回答 |
|------|-----------|
| **内存碎片是什么？** | Redis 分配了内存但没用的部分；频繁删改 Big Key 会产生大量碎片，碎片率超过 1.5 就需要处理 |
| **怎么清理？** | ① `CONFIG SET activedefrag yes` 开启主动碎片整理（推荐）② 重启释放（简单粗暴）|
| **Redis 事务能回滚吗？** | **不能回滚**。MULTI/EXEC 中语法错误会 reject 整个批次，但执行错误只跳过错误命令继续执行后续命令 |
| **PubSub 和 Stream 的区别？** | PubSub 无持久化、离线消息丢失；Stream 支持 ACK 消费确认、持久存储、消费者组（Consumer Group） |
| **Pipeline 性能提升多少？** | 减少 RTT 开销，通常 10~50 倍提升；但大批量操作可能占用主线程，注意配合 TIMEOUT 使用 |

---

## 深度展开

### 1. 内存碎片：产生原因与监控

#### 1.1 为什么会产生碎片？

```
场景：频繁更新大 Key
┌─────────────┬──────────┬──────────┐
│   操作       │  分配    │  实际使用 │
├─────────────┼──────────┼──────────┤
│ SET key big  │ 10MB     │ 10MB     │ ✅ 无碎片
│ DEL key      │ 0        │ 0        │ 💥 内存未立即归还 OS！
│ SET key big2 │ 10MB     │ 10MB     │ 💥 旧块散落各处
│ GET key      │ 10MB     │ 10MB     │ ┘
└─────────────┴──────────┴──────────┘
结果：RSS 显示用了 ~30MB，但实际数据只占了 20MB
→ 碎片率 = (RSS - 实际使用) / RSS = 33%
```

**关键机制**：C 语言 `malloc/free` 释放内存后，OS 并不立即回收页面（coalesce 开销大），导致 Redis 进程的 RSS 高于实际数据占用。

#### 1.2 如何监控？

```bash
# 方式 1：INFO memory
redis-cli INFO memory
used_memory:85940000
used_memory_rss:129800000    # 操作系统看到的内存
mem_fragmentation_ratio:1.51 # 碎片率 > 1.5 警告

# 方式 2：redis-cli --bigkeys --memory
redis-cli --memory --bigkeys

# 计算碎片率公式：
# fragmentation_ratio = used_memory_rss / used_memory
# 正常范围：1.0 ~ 1.5
# > 1.5 : 需要关注
# > 2.0 : 严重碎片，建议干预
```

```go
// Go 客户端读取碎片率的示例代码
func checkMemoryFragmentation(r *redis.Client) error {
    info, err := r.Info("memory").Result()
    if err != nil {
        return err
    }

    // 解析 mem_fragmentation_ratio
    for _, line := range strings.Split(info, "\n") {
        if strings.HasPrefix(line, "mem_fragmentation_ratio:") {
            ratio := parseFloat(strings.TrimSpace(strings.TrimPrefix(line, "mem_fragmentation_ratio:")))
            if ratio > 1.5 {
                return fmt.Errorf("high fragmentation: %.2f", ratio)
            }
        }
    }
    return nil
}
```

#### 1.3 碎片率异常高的排查思路

```
碎片率高（>1.5）
    ↓
是大 Key 频繁读写导致？
    ├─ YES → 拆分子 key（Hash 分桶）、改用 UNLINK 删除
    └─ NO → 检查是否有大量短命 key（TTL 极短）
            ├─ YES → 这些 key 被快速写入+过期+删除，
            │          造成碎片累积 → 考虑调小 maxmemory 
            │          或增加 AOF 重写频率
            └─ 无法定位 → 重启 Redis 治标不治本
```

---

### 2. Active Defragmentation：主动碎片整理

Redis 4.0+ 引入了 **active defragmentation**——在业务不阻塞的前提下，后台逐步将散落的内存块合并。

```ini
# redis.conf 配置
activedefrag yes                          # 是否启用
active-defrag-ignore-bytes 100mb          # 大于此值的对象不参与整理
active-defrag-threshold-lower 10          # 碎片率低于 10% 时停止整理
active-defrag-threshold-upper 100         # 碎片率超过 100% 时全力整理
active-defrag-cycle-min 5                 # 每次循环最少百分比
active-defrag-cycle-max 25                # 每次循环最多百分比
```

**工作原理：**

```
Active Defrag 后台线程循环：

1. 扫描所有数据库
2. 找到内存碎片率过高的页
3. 对于页中的每个对象：
   a. 分配一块更大的连续内存
   b. 将数据复制到新内存（memmove）
   c. 替换原指针指向新内存
   d. 将旧内存标记为空闲
4. 重复直到碎片率降至阈值以下
```

```
Before:                    After:
┌─────┬─────┬─────┬─────┐   ┌───────────┬───────┬───────┐
│obj1 │FREE │obj2 │FREE │→  │obj1 obj2 obj3│ FREE │ FREE │
└─────┴─────┴─────┴─────┘   └───────────┴───────┴───────┘
  碎片                            无碎片
```

**⚠️ 注意事项：**

| 场景 | 影响 |
|------|------|
| CPU 密集型实例 | Active Defrag 会额外消耗 CPU，注意压测 |
| 超大对象 | `active-defrag-ignore-bytes` 设小一点，避免搬运大对象拖慢服务 |
| 内存极度紧张 | 先增大 `maxmemory` 再开 defrag，否则搬运时需要双份内存 |

---

### 3. Redis 事务的局限性：为什么没有 ROLLBACK？

```
MULTI → 入队阶段
EXEC  → 执行阶段

如果入队时有语法错误（如 WRONGTYPE）：EXEC 返回 ERR，整个批次的命令都不会执行。
如果执行时有运行时错误（如对一个 String 类型调用 LPOP）：仅该条命令失败，其他命令照常执行，不会回滚！
```

```bash
# 场景：SET 一个已存在的 Hash 类型 key
> SET mykey "string_value"
OK
> MULTI
OK
> HGETALL mykey    # 语法正确，入队成功
QUEUED
> LPUSH mykey item # 语法正确，入队成功
QUEUED
> EXEC
ERR Operation against a key holding the wrong kind of value
# ⚠️ 整个 EXEC 被 reject，两条命令都没执行
```

```bash
# 场景二：执行时的运行错误
> SET count 5
OK
> MULTI
OK
> INCR count           # ✅ 正确，count 变成 6
QUEUED
> LPUSH queue 1        # ❌ 运行时错误（count 是 String，不是 List）
QUEUED
> DECR count           # ✅ 正确，count 变成 5
QUEUED
> EXEC
1) (integer) 6         # ⚠️ 第一条成功了！
2) (error) WRONGTYPE ... # ⚠️ 第二条报错了！
3) (integer) 5          # ⚠️ 第三条也成功了！
# 没有回滚，失败的命令之后的命令照样执行了
```

**本质原因**：Redis 是多路复用单线程模型，不支持复杂的事务恢复机制。
相比关系型数据库的事务：

| 特性 | MySQL InnoDB | Redis |
|------|-------------|-------|
| 原子性 | ✅ 完整 ACID | ✅ 仅保证 EXEC 期间不被中断 |
| 回滚 | ✅ 通过 Undo Log | ❌ **不支持 ROLLBACK** |
| 隔离级别 | RC / RR | **无隔离级别**（单线程，天然串行）|
| WATCH | N/A | ✅ 乐观锁（WATCH + EXEC）|

**面试话术**：
> "Redis 不做 ROLLBACK 是有意为之的设计选择。因为 Redis 是单线程模型，加 undo log 和恢复机制会增加复杂度且收益有限。如果真的需要严格的事务语义，应该用 Lua 脚本保证原子性，或者在应用层实现补偿逻辑。"

---

### 4. PUB/SUB vs STREAM：可靠性差异

这是生产选型中最常遇到的抉择。

#### 4.1 Pub/Sub：广播式消息，无持久化

```bash
# 发布者
> PUBLISH channel "hello world"
(integer) 1    # 只有一个订阅者收到了

# 订阅者 A（在线时）
> SUBSCRIBE channel
1) "subscribe"
2) "channel"
3) (integer) 1
1) "message"
2) "channel"
3) "hello world"    # ✅ 收到消息

# 订阅者 B（离线后再上线）
# （期间发布了一条新消息）
> SUBSCRIBE channel
1) "subscribe"
2) "channel"
3) (integer) 1
# ❌ 离线期间发布的消息完全丢失！
```

**Pub/Sub 致命弱点**：
1. **无持久化**：消息只存在于内存，发布即焚
2. **无 ACK**：无法知道订阅者是否收到
3. **无堆积**：订阅者掉线，消息立刻丢弃
4. **全集群广播**（Cluster 模式下）：消息复制给所有节点

#### 4.2 Stream：可靠的消息队列实现

```bash
# 添加消息
> XADD mystream * user alice action login
-> 1673456789012-0

# 消费者组消费（带 ACK）
> XGROUP CREATE mystream group1 $          # $ = 从最新消息开始
> XREADGROUP GROUP group1 consumer1 COUNT 1 BLOCK 0 STREAMS mystream >
# 返回消息后，需要手动 ACK
> XACK mystream group1 1673456789012-0
(integer) 1    # 消费确认成功

# 查看待处理消息（pending entries list）
> XPENDING mystream group1
```

**Stream vs Pub/Sub 对比表**：

| 特性 | Pub/Sub | Stream |
|------|---------|--------|
| 消息持久化 | ❌ 内存中瞬时存在 | ✅ 可持久化存储到磁盘 |
| 消费确认(ACK) | ❌ 无 | ✅ XACK |
| 断线重连后收到历史消息 | ❌ 丢失 | ✅ PEL（Pending Entries List）|
| 消费者组 | ❌ | ✅ 多个消费者分担负载 |
| 消息积压能力 | ❌ 无 | ✅ 有序追加，可回溯读取 |
| 适用场景 | 实时通知、日志广播 | 订单处理、事件溯源、任务队列 |

```go
// Go 中使用 Stream 实现可靠消息队列（示意）
import "github.com/redis/go-redis/v9"

func consumeMessage(ctx context.Context, rdb *redis.Client) {
    msgs, err := rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
        Group:    "order-group",
        Consumer: "worker-1",
        Streams:  []string{"orders-stream", ">"},
        Count:    10,
        Block:    5 * time.Second,
    }).Result()
    
    if err != nil || len(msgs) == 0 {
        return  // timeout 或错误
    }
    
    for _, m := range msgs[0].Messages {
        processOrder(m)
        // 处理成功后必须 ACK
        rdb.XAck(ctx, "orders-stream", "order-group", m.ID)
    }
}
```

---

### 5. Pipeline 性能优化：何时用、怎么用

#### 5.1 Pipeline 的本质

Pipeline 就是**批量打包命令，一次网络往返完成多个请求**，避免逐条命令的 RTT 开销。

```
不使用 Pipeline：              使用 Pipeline：
┌────┐                        ┌────┐
│Client│                      │Client│
└──┬──┘                        └──┬──┘
   ├──→ SET key1 val1 ←RTT1      ├──→ MSET key1 val1 key2 val2 ←1 RTT
   ├──→ SET key2 val2 ←RTT2      └──→ 获取全部结果
   ├──→ SET key3 val3 ←RTT3
   └──→ SET key4 val4 ←RTT4
   
耗时: 4 × RTT                     耗时: 1 × RTT
```

#### 5.2 Pipeline 实测效果

```go
// 假设 RTT = 1ms，发送 10000 条 SET 命令
// 不使用 Pipeline: ~10000ms
// 使用 Pipeline:    ~2-5ms
// 提速约 2000-5000x
```

#### 5.3 Pipeline 的最佳实践

```go
// ✅ 好的做法：分批控制单次大小
const batchSize = 2000
for i := 0; i < len(keys); i += batchSize {
    end := min(i+batchSize, len(keys))
    pipe := rdb.Pipeline()
    for _, key := range keys[i:end] {
        pipe.Get(ctx, key)
    }
    results, _ := pipe.Exec(ctx)
    // 处理结果
}

// ❌ 不好的做法：单个 pipeline 塞几万条
// 会导致单个 request buffer 过大，长时间占用主线程

// ⚠️ Pipeline 中命令的执行错误不会被包装成整体 error
// 需要在 Exec 返回后逐个检查结果
results, err := pipe.Exec(ctx)
if err != nil {
    // err 可能是 redis.Nil（key 不存在）或真正的网络错误
}
for i, result := range results {
    if e := result.Err(); e != nil {
        log.Printf("command %d failed: %v", i, e)
    }
}
```

**Pipeline vs Lua 脚本对比**：

| 维度 | Pipeline | Lua 脚本 |
|------|----------|---------|
| 原子性 | ❌ 非原子，中间可能被插入其他命令 | ✅ 整个脚本原子执行 |
| 性能 | ⚡ 极高（批量网络往返） | 🟡 中等（需等待执行完毕）|
| 条件逻辑 | ❌ 不支持 | ✅ 支持 if/while 等 |
| 适用场景 | 批量读/写，无依赖 | 需要原子性的复合操作（库存扣减）|
| 主线程阻塞 | 低（网络包大但处理快） | 高（大 key 操作可能阻塞主线程）|

**面试结论**：
> Pipeline 用于**无依赖的批量操作**，最大化吞吐；Lua 用于**需要原子性的复合操作**，保证逻辑一致性。两者不是互斥的，可以结合使用。

---

## 高频追问

### Q1：Redis 什么时候该用 UNLINK 而不是 DEL？

```bash
# DEL：同步删除，Big Key 时会阻塞主线程
> DEL huge_key
# 可能阻塞 100ms+

# UNLINK：异步删除，非阻塞
> UNLINK huge_key
# 立即返回，后台异步释放
```

**规则**：Key 值超过 10KB 或集合元素超过 5000，一律用 `UNLINK`。

### Q2：Redis Cluster 模式下 Pipeline 还能用吗？

**可以**，但有约束：
- 所有 key 必须在同一个 slot（否则会报错或拆成多个请求）
- 可以用 `KEYS` 模式（如 `user:*`）来确保同一 slot
- 跨 slot 的 pipeline 需要客户端自动路由（go-redis 会自动拆分）

### Q3：如何优雅地关闭 Redis active defrag？

```bash
# 临时关闭（热切换，不影响业务）
CONFIG SET activedefrag no

# 永久生效（写入配置）
CONFIG SET activedefrag no
SAVE
```

生产中建议在维护窗口关闭 defrag 后执行 BGREWRITEAOF，然后观察碎片率是否恢复正常。

---

## 延伸阅读

- [Redis Memory Management Docs](https://redis.io/docs/latest/operate/rs/tutorials/memory_optimization/)
- [Redis Active Defragmentation](https://redis.io/docs/latest/operate/oss_and_stack/management/memory-tuning/#active-defragmentation)
- [Redis Transactions](https://redis.io/docs/latest/develop/interact/transactions/)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)
- [Redis Pipelining](https://redis.io/docs/latest/develop/use/pipelining/)
- [Redis Lua Scripting](https://redis.io/docs/latest/develop/interact/programmatic-interaction/lua-scripting/)
