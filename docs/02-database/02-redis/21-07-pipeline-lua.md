[🏠 首页](../../../README.md) · [🗺️ 数据库](../../README.md) · [🔴 Redis](../README.md)

---

# Redis Pipeline 与 Lua 脚本

> 考察频率：★★★★☆  优先级：P1

## 面试官考察意图

考察候选人对 Redis 性能优化手段的掌握深度。
初级只知道 Redis 单命令快，高级要能讲清楚 **Pipeline 的 RTT 压缩原理与原子性边界、Lua 脚本的 Redis 单线程原子性保证、以及 MULTI/EXEC 事务与 Lua 的取舍**，并能针对超卖、分布式限流等场景给出完整的 Go + Redis 实战代码。
本篇也是 05-distributed-lock.md（分布式锁）的进阶补充，锁的续期逻辑就依赖 Lua 脚本。

---

## 核心答案（30 秒版）

| 手段 | 核心原理 | 原子性 | 适用场景 |
|------|---------|--------|---------|
| **Pipeline** | 批量 N 条命令，1 次 RTT 往返 | ❌ 无（命令逐条执行） | 批量读取、无关联写 |
| **Lua 脚本** | Redis 单线程执行脚本期间不处理其他命令 | ✅ 完全原子 | 先读后写、条件判断 |
| **MULTI/EXEC** | 队列缓存命令，一次 RTT 提交 | ⚠️ 部分原子（错误不回滚） | 简单批量 |

生产中最常用的组合：**Pipeline 批量获取 + Lua 脚本原子操作**。

---

## 深度展开

### 1. Pipeline 原理

#### RTT 是 Redis 性能瓶颈的根源

Redis 是单线程处理命令，QPS 能轻松达到 10w+。但每条命令的**网络往返时间（RTT）**是瓶颈：

```go
// 逐条执行：N 次 RTT
rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
for _, key := range keys {
    rdb.Get(ctx, key)    // 每次都是一次网络往返（假设 RTT=1ms）
}
// 1000 个 key = 1 秒（纯网络等待，Redis CPU 空闲）

// Pipeline：1 次 RTT
pipe := rdb.Pipeline()
for _, key := range keys {
    pipe.Get(key)
}
results, err := pipe.Exec(ctx)  // 一次性发请求，一次性收响应
// 1000 个 key ≈ 10ms（接近 Redis 本身的处理速度）
```

benchmark 参考（localhost RTT≈0.1ms，跨机房 RTT≈5ms）：

| 方式 | 1000 次 GET（本地） | 1000 次 GET（跨机房） |
|------|------------------|---------------------|
| 逐条执行 | ~100ms | ~5000ms |
| Pipeline | ~10ms | ~15ms |

#### Pipeline 的实现机制

```
客户端                           Redis 服务端
  │                                  │
  │  GET k1                          │
  │  GET k2                          │
  │  GET k3                          │
  │  ...                             │
  │  MGET k1 k2 k3 ...  ← 一次性发   │
  │─────────────────────>            │
  │                                  │  处理所有命令
  │  [resp1, resp2, resp3...]  ← 一次性收
  │<──────────────────────           │
  │  解析 responses 并一一对应
```

Pipeline 本身不改变 Redis 处理命令的方式——它只是**客户端侧的批处理优化**。Redis 收到一批命令后仍然逐条执行、逐条响应，只是响应被合并在一次网络传输中。

#### Go 中使用 go-redis Pipeline

```go
import "github.com/redis/go-redis/v9"

func pipelineExample(ctx context.Context, rdb *redis.Client) error {
    // 批量获取用户信息
    pipe := rdb.Pipeline()
    
    cmds := make([]*redis.StringCmd, len(userIDs))
    for i, uid := range userIDs {
        cmds[i] = pipe.Get(ctx, fmt.Sprintf("user:%s", uid))
    }
    
    // 执行（一次性发请求）
    _, err := pipe.Exec(ctx)
    if err != nil && err != redis.Nil {
        return err
    }
    
    // 解析结果
    for _, cmd := range cmds {
        val, err := cmd.Result()
        if err == redis.Nil {
            // 用户不存在
            continue
        }
        // 处理 val...
    }
    return nil
}
```

#### Pipeline 的关键局限

```go
// ❌ Pipeline 不能保证原子性（中间命令失败不回滚）
pipe := rdb.Pipeline()
pipe.Incr(ctx, "counter")
pipe.Expire(ctx, "counter", time.Hour)
pipe.Exec(ctx)
// 如果 Incr 成功了但 Expire 因网络问题失败
// counter 已经 +1，但没设置过期时间
// 数据一致性问题

// ❌ Pipeline 不适合强事务依赖
// 命令 A 的结果决定命令 B 的执行内容
// Pipeline 中 A 和 B 同时发送，无法根据 A 的结果决定 B 的参数
```

---

### 2. Lua 脚本原子性

#### 为什么 Lua 是原子的

Redis 在执行 Lua 脚本时**单线程独占**：

```
时间轴：
[命令1] [命令2] [Lua脚本执行中...] [命令3] [命令4]
                        ↑
              单线程处理，不会有其他命令插入
```

Lua 脚本执行期间，Redis 不会处理其他任何客户端的命令，直到脚本执行完毕。这是 Redis 设计上的选择：用 Lua 脚本替代复杂的多命令事务。

#### EVAL vs EVALSHA

```bash
# EVAL：每次发送脚本源码（网络开销大）
EVAL "return redis.call('GET', KEYS[1])" 1 "mykey"

# EVALSHA：发送脚本 SHA1（先 EVALSHA 后 Redis 缓存脚本）
# 首次：EVAL 将脚本缓存在 Redis
EVAL "return redis.call('GET', KEYS[1])" 1 "mykey"
# -> (integer) 3

# 后续：EVALSHA 直接用缓存的 SHA
EVALSHA "a5b9c3d7..." 1 "mykey"
# -> (integer) 3

# Redis 重启后脚本缓存丢失，EVALSHA 会报错 NOSCRIPT
# 解决：SCRIPT LOAD + EVALSHA 组合
```

#### 超卖问题的 Lua 实现

这是 Redis 超卖面试题的标准解法（先查后改的竞态问题）：

```go
// Go 端：使用 Lua 脚本保证原子性
const decrementStockLua = `
local stock = redis.call('GET', KEYS[1])
if stock == false then
    return -1  -- 商品不存在
end
stock = tonumber(stock)
if stock <= 0 then
    return 0  -- 库存不足
end
redis.call('DECR', KEYS[1])
return stock - 1
`

func DecrementStock(ctx context.Context, rdb *redis.Client, goodsID string) (int64, error) {
    result, err := rdb.Eval(ctx, decrementStockLua, []string{"stock:" + goodsID}).Int64()
    return result, err
}

// Lua 脚本执行过程：
// 1. GET stock:goods_001  -> "10"
// 2. 判断 stock > 0        -> true
// 3. DECR stock:goods_001 -> 9
// 整个过程原子，中间不会被其他客户端插入
```

#### 分布式限流的 Lua 实现（令牌桶）

```go
const rateLimitLua = `
local key = KEYS[1]
local now = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local rate = tonumber(ARGV[3])  -- 每秒补充令牌数

local last = tonumber(redis.call('HGET', key, 'last') or now)
local tokens = tonumber(redis.call('HGET', key, 'tokens') or capacity)

-- 令牌补充（时间差 * rate）
local elapsed = now - last
local补充 = math.floor(elapsed * rate)
tokens = math.min(capacity, tokens + 补充)

-- 尝试获取令牌
if tokens >= 1 then
    redis.call('HSET', key, 'tokens', tokens - 1, 'last', now)
    return 1  -- 允许
else
    redis.call('HSET', key, 'tokens', tokens, 'last', now)
    return 0  -- 拒绝
end
`

func Allow(ctx context.Context, rdb *redis.Client, key string, capacity int, rate float64) (bool, error) {
    nowMs := time.Now().UnixMilli()
    result, err := rdb.Eval(ctx, rateLimitLua, []string{key},
        nowMs, capacity, rate).Int64()
    return result == 1, err
}
```

---

### 3. MULTI/EXEC 事务 vs Lua 对比

| 维度 | MULTI/EXEC | Lua 脚本 |
|------|-----------|---------|
| **原子性** | 部分原子（命令错误不回滚） | 完全原子（脚本中止即回滚） |
| **错误处理** | 单条命令失败后续继续 | 脚本出错直接停止 |
| **网络开销** | N+1 次 RTT（EXEC 本身一次） | N+1 次 RTT（EVAL 发送源码） |
| **性能** | 较差（命令逐条传输） | 较好（脚本一次性传输） |
| **灵活性** | 低（只能按顺序执行） | 高（支持条件分支、循环） |
| **适用场景** | 简单批量操作 | 复杂条件判断、读后写 |
| **回滚能力** | ❌ 不支持（Redis 设计选择） | ❌ 不支持 |

#### MULTI/EXEC「不支持回滚」的设计原因

Redis 作者 Antirez（Salvatore Sanfilippo）的解释：

> Redis 是追求**简单性和高性能**的 KV 数据库，不是关系型数据库。
> 事务回滚需要维护 undo log，这会：
> 1. 增加复杂度（Redis 单线程，复杂度是噩梦）
> 2. 影响性能（每次写操作都要记 undo）
> 3. 与 Redis 的简单设计哲学冲突

所以 Redis 选择让 MULTI/EXEC 的错误**直接报错不回滚**，如果需要回滚就用 Lua 脚本在执行前做判断。

#### WATCH 乐观锁

```go
// WATCH 实现 CAS：监控 key 变化，变化则事务取消
func incrementWithWatch(ctx context.Context, rdb *redis.Client, key string) error {
    const maxRetries = 3
    for i := 0; i < maxRetries; i++ {
        // WATCH 开启监控
        err := rdb.Watch(ctx, func(tx *redis.Tx) error {
            n, err := tx.Get(ctx, key).Int()
            if err != nil && err != redis.Nil {
                return err
            }
            // 在事务内（Redis 会保证执行期间 key 未变化）
            _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
                pipe.Set(ctx, key, n+1, 0)
                return nil
            })
            return err
        }, key)
        
        if err == nil {
            return nil  // 成功
        }
        if err == redis.TxFailedErr {
            continue   // 监控的 key 被其他客户端修改，重试
        }
        return err
    }
    return errors.New("max retries exceeded")
}
```

---

### 4. 生产注意事项

#### Lua 脚本执行时间过长会阻塞 Redis

```bash
# lua-time-limit 默认 5000ms（5秒）
# 脚本执行超过此时间，Redis 返回错误
(error) ERR script is taking too long

# 生产建议：
# 1. 脚本尽量简短（< 100 行）
# 2. 避免循环中调用 Redis（用 pipeline 代替）
# 3. 监控脚本慢查询：redis-cli SLOWLOG GET 10
```

#### Cluster 模式下 Lua 脚本的限制

```go
// ❌ Cluster 模式下，所有 key 必须在同一 slot
// 否则报错：CROSSSLOT Keys in request don't hash to the same slot
result, err := rdb.Eval(ctx, luaScript, []string{"key1", "key2"}, args)

// ✅ 解决：用 {tag} 强制 key 路由到同一 slot
result, err := rdb.Eval(ctx, luaScript,
    []string{"user:{123}:profile", "user:{123}:setting"}, args)
// user:123:profile 和 user:123:setting 都路由到 #{123} slot

// ✅ 或者用 Hash 代替多个 key
redis.call('HMGET', KEYS[1], 'field1', 'field2')
// HMGET 一个 hash 的多个 field，一定在同一 slot
```

#### 脚本调试

```bash
# 本地调试模式（类似 GDB）
redis-cli --ldb-mode yes --eval "script" key1 key2, arg1 arg2

# 调试命令
redis-cli > DEBUG SEGFAULT  # 模拟崩溃
redis-cli > SCRIPT DEBUG YES  # 开启脚本调试

# 脚本存活检测
redis-cli > SCRIPT EXISTS $(echo "return 1" | redis-cli -x SCRIPT LOAD STDIN)
```

#### Pipeline + Lua 的正确组合方式

```go
// Pipeline 不能和 Lua 直接组合（Pipeline 内的 Lua 脚本不原子）
// 正确做法：Lua 内自己实现 Pipeline 逻辑（Redis 脚本内多条命令天然批量化）

// 如果需要：
// 1. 先批量 GET（Pipeline）
// 2. Lua 脚本做计算
// 3. 批量 SET（Pipeline）
// 那就分成两个独立阶段，不是一件事务

// Lua 脚本内部天然是 Pipeline（多条 redis.call 一次性发给 Redis）
const complexLua = `
local data = redis.call('MGET', unpack(KEYS))  -- 批量获取
-- 业务计算...
redis.call('MSET', unpack(result))            -- 批量写入
return affected
`
```

---

## 高频追问

**Q1: Redis 事务与数据库事务的本质区别？**

数据库事务（ACID）通过 Redo Log / Undo Log 保证原子性和持久性，支持回滚。Redis MULTI/EXEC 只是把命令批量提交，不保证回滚，持久性依赖 RDB/AOF。两者根本区别在于：数据库事务是"要么全做、要么不做"，Redis 事务是"要么全收到、要么全不收"（但执行过程中出错不管）。

**Q2: 为什么 Redis Cluster 下 Lua 脚本受限？**

Cluster 将数据按 slot 分布在多个节点。Lua 脚本执行多条命令时，如果涉及的 key 不在同一 slot，Redis 无法确定该把脚本发到哪个节点执行（因为不同 key 在不同节点）。解决方法是 `{tag}` 强制 key 路由到同一 slot，或者用 Hash 结构代替多个 key。

**Q3: Pipeline 和 Lua 能组合用吗？**

Pipeline 和 Lua 是互补关系，不是包含关系：Pipeline 是**客户端批处理**（减少 RTT），Lua 是**服务端原子执行**（保证多命令原子性）。如果你需要批量读取然后计算再批量写入，应该在 Lua 脚本里用 `MGET`/`MSET` 命令（天然就是服务端批处理），而不是先 Pipeline 再 Lua。

**Q4: Lua 脚本超时后是什么状态？**

Redis 对超时 Lua 脚本的处理是**直接杀死脚本进程**（通过 `SIGTERM`）。这意味着超时脚本的所有效果都会被回滚（因为脚本根本没执行完），不会留下半写入的数据。这是 Redis 保证了脚本的"原子性边界"——要么完全执行完，要么完全不执行。

---

## 延伸阅读

- [Redis 官方文档：Transactions & Lua](https://redis.io/docs/interact/transactions/)
- [Redis Lua 脚本指南](https://redis.io/docs/interact/programmability/eval-intro/)
- [go-redis Pipeline 文档](https://redis.uptrace.dev/guide/go-pipeline.html)
- [Redis WATCH 与乐观锁](https://redis.io/docs/data-types/transactions/)
- [Antirez: Redis事务不支持回滚的设计决策](http://antirez.com/news/122)
- [Redis Cluster Lua 脚本限制](https://redis.io/docs/data-types/transactions/#lua-scripts)
