# 库存系统设计（防超卖 + 热点分段 + 库存回补）

> 考察频率：★★★★★  难度：★★★★☆  优先级：P0
> 关键词：超卖、Redis Lua、分段库存、热点 Key、乐观锁、库存回补、对账

## 核心问题

| 问题 | 描述 |
|------|------|
| 超卖 | 并发扣减下如何保证 `stock >= 0`，绝不卖超 |
| 热点 | 单车爆款商品库存 Key 被打爆，单分片 Redis 扛不住 |
| 一致性 | Redis 扣减成功但订单失败，库存怎么补回 |
| 回补 | 取消/超时/退款后库存如何准确还回去，且不重复还 |
| 对账 | Redis 与 DB 库存长期不一致如何发现和修复 |

---

## 一、从"数据库扣减"说起

```sql
-- 最朴素的写法（有超卖风险，因为「查」和「减」不是原子的）
SELECT stock FROM product WHERE id = 1;   -- 查到 1
-- 另一个请求也查到 1
UPDATE product SET stock = 1 - 1 WHERE id = 1; -- 两个请求都执行，卖出 2 件
```

### 正确写法一：条件更新（CAS）

```sql
UPDATE product
SET stock = stock - 1
WHERE id = 1 AND stock >= 1;   -- 影响行数为 0 即库存不足
```

> 单行更新由于 InnoDB 行锁是原子的，**只要带上 `AND stock >= 1`，就不会超卖**。这是最可靠、最推荐的兜底。

### 正确写法二：乐观锁版本号

```sql
UPDATE product
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5 AND stock >= 1;
```

并发冲突时影响行数为 0，业务层重试。缺点：冲突高时重试成本大。

### 正确写法三：悲观锁（`SELECT ... FOR UPDATE`）

```sql
BEGIN;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;  -- 持有行锁
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

> 面试注意：**悲观锁会把热点行串行化**，大促时几百 QPS 排队就足以把连接池打满，一般不作为主方案。

---

## 二、Redis 预扣减 + Lua 原子脚本（大促主方案）

思路：把库存预热到 Redis，用 Lua 脚本完成"校验 + 扣减 + 记录用户"的原子操作，DB 只做异步落库与兜底。

```lua
-- KEYS[1] = stock:{skuID}       库存
-- KEYS[2] = bought:{skuID}      已购用户集合（一人一单）
-- ARGV[1] = userID
-- ARGV[2] = quantity
local stock = redis.call('GET', KEYS[1])
if not stock then return -1 end              -- 库存未预热
if tonumber(stock) < tonumber(ARGV[2]) then
    return 0                                  -- 库存不足
end
if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then
    return 2                                  -- 重复购买
end
redis.call('DECRBY', KEYS[1], ARGV[2])
redis.call('SADD', KEYS[2], ARGV[1])
return 1                                      -- 成功
```

```go
var preDecrScript = redis.NewScript(`
local stock = redis.call('GET', KEYS[1])
if not stock then return -1 end
if tonumber(stock) < tonumber(ARGV[1]) then return 0 end
if redis.call('SISMEMBER', KEYS[2], ARGV[2]) == 1 then return 2 end
redis.call('DECRBY', KEYS[1], ARGV[1])
redis.call('SADD', KEYS[2], ARGV[2])
return 1
`)

type DeductResult int

const (
    DeductOK        DeductResult = 1
    DeductSoldOut   DeductResult = 0
    DeductNotWarmup DeductResult = -1
    DeductDuplicate DeductResult = 2
)

func PreDeduct(ctx context.Context, rdb *redis.Client, skuID, userID string, qty int) (DeductResult, error) {
    keys := []string{"stock:" + skuID, "bought:" + skuID}
    r, err := preDecrScript.Run(ctx, rdb, keys, qty, userID).Int()
    return DeductResult(r), err
}
```

> Lua 脚本在 Redis 单线程内**整体执行不被打断**，所以"判断用户 + 判断库存 + 扣减 + 记录"四步天然原子。

---

## 三、热点库存 Key 分段（核心加分项）

单车爆款库存如 10 万件，全打在 `stock:sku_1001` 一个分片上，单分片 QPS 上限（约 10 万）会被直接打满。

**方案：把库存拆成 N 个分片 Key。**

```
1000 件库存 → 拆成 10 个 Key：stock:sku_1001_0 ... stock:sku_1001_9，每个 100 件

用户请求 → hash(userID) % 10 → 只打某一个分片
```

```go
const shardNum = 10

func PreDeductSharded(ctx context.Context, rdb *redis.Client, skuID, userID string, qty int) (DeductResult, error) {
    for i := 0; i < shardNum; i++ {
        // 以 userID 为起点轮询，保证同一用户优先固定分片（也保证不超卖）
        idx := (hash(userID) + i) % shardNum
        key := fmt.Sprintf("stock:%s_%d", skuID, idx)
        r, err := preDecrScript.Run(ctx, rdb,
            []string{key, "bought:" + skuID}, qty, userID).Int()
        if err != nil {
            return 0, err
        }
        if DeductResult(r) == DeductOK {
            return DeductOK, nil
        }
        if DeductResult(r) == DeductDuplicate {
            return DeductDuplicate, nil
        }
    }
    return DeductSoldOut, nil
}

func hash(s string) int {
    h := fnv.New32a()
    _, _ = h.Write([]byte(s))
    return int(h.Sum32())
}
```

| 维度 | 单 Key | 分段 Key |
|------|--------|----------|
| 单分片 QPS | 全部压力 | 均摊到 N 个 |
| 库存准确性 | 简单 | 各段独立扣减，各段不超卖即整体不超卖 |
| 复杂度 | 低 | 需要处理"某段扣完"的降级 |

> 分段还有一个好处：**每段库存只需要 DB 里 `stock` 字段的一部分**，DB 侧可以用 `UPDATE ... WHERE stock >= 1` 保证不超卖，形成双重保险。

---

## 四、库存回补（取消 / 超时 / 退款）

**原则：回补必须幂等，绝不能重复加库存。**

```go
func RestoreStock(ctx context.Context, rdb *redis.Client, skuID, orderID string, qty int) error {
    // 用「回补流水表」的唯一索引保证同一个 order 只回补一次
    // DB: INSERT INTO stock_restore_log(order_id) VALUES (?)
    //     ON DUPLICATE KEY UPDATE id=id  → 影响行数 0 表示已回补
    exist, err := rdb.SetNX(ctx, "restore:"+orderID, "1", 24*time.Hour).Result()
    if err != nil {
        return err
    }
    if !exist {
        return nil // 已回补过
    }
    // 分片回补：与扣减用同一分片函数即可（也可直接 INCR 主 Key）
    shardKey := fmt.Sprintf("stock:%s_%d", skuID, hash(orderID)%shardNum)
    return rdb.IncrBy(ctx, shardKey, int64(qty)).Err()
}
```

### 什么时候回补

| 场景 | 回补时机 | 注意 |
|------|----------|------|
| 用户主动取消 | 状态机 `PENDING → CANCELED` 成功后 | 已支付订单取消要走退款，不是简单回补 |
| 超时未支付 | 延迟消息触发关单成功后 | 必须 CAS 成功才回补 |
| 支付失败 | 支付回调失败后 | 同超时 |
| 退款成功 | 退款流程回调成功后 | 需要区分"未发货退款"是否回转库存 |

---

## 五、一致性对账（生产必做）

Redis 库存与 DB 库存一定会有漂移，靠"事后对账"兜底：

```
1. 每次扣减都写一条「库存流水」（order_id, sku_id, delta）到 MQ
2. 定时任务（如每 5 分钟）：
   - 从流水聚合出 sku 的「已扣减总量」
   - 与 DB 中 actual_sold 比对
   - 若 |Redis 剩余 - (DB 总库存 - 已扣减)| > 阈值 → 告警 + 以 DB 为准回写 Redis
3. 业务侧监控：Redis 剩余库存 = 0 但仍有大量请求 → 触发限流/降级
```

**面试要点：库存这类"钱"相关的数据，永远要有一个"以 DB 为准"的最终裁决权，Redis 只是加速器。**

---

## 六、高频追问

### Q1：Redis 挂了怎么办？会不会超卖？

- Redis 集群主从 + 哨兵，主挂自动切；
- 真的全挂 → **降级到直接打 DB**（`UPDATE ... WHERE stock >= 1`），DB 的条件更新天然不超卖，只是吞吐下降；
- 绝对不能"Redis 挂了就直接放行"，那必然超卖。

### Q2：一人一单怎么保证？

- Lua 里 `SISMEMBER bought:sku`（如上面的脚本）；
- 或用 `SETNX locked:{sku}:{user}`；
- DB 侧 `UNIQUE(user_id, sku_id)` 唯一索引兜底。

### Q3：库存预热到 Redis，活动开始前超卖/少卖怎么处理？

- 预热时以 DB 为准写 Redis，并核对总量；
- 活动开始后**Redis 是权威**，DB 异步落库；
- 若 Redis 与 DB 不一致，活动结束后以对账结果修正 DB（"少卖"通常比"超卖"可接受，宁可少卖不可超卖）。

### Q4：为什么要分段而不是只提高 Redis 性能？

> Redis 单线程处理命令，单个 Key 的操作无法并行；分段本质上是**把一个热点 Key 变成 N 个普通 Key，用并发换吞吐**。这是解决 Redis 热 Key 的通用手段（点赞、计数、排行榜同源）。

---

## 面试话术

**Q：如何防止超卖？**

> 我会分两层讲。**可靠性的底座是数据库**：`UPDATE product SET stock = stock-1 WHERE id=? AND stock>=1`，靠 InnoDB 行锁和条件更新保证绝不超卖。**性能层用 Redis 预扣减**：Lua 脚本一次性完成库存校验、用户去重和扣减，把绝大部分流量挡在 DB 之前。爆款商品的库存 Key 我会拆成 10 个分片，用 hash 分流，避免单个 Redis Key 成为热点。

**Q：Redis 扣了但订单创建失败，库存怎么办？**

> 靠**回补 + 幂等**。订单失败会发一条回补消息，回补前用 `restore:{orderID}` 做 SETNX，或者用回补流水表的唯一索引，保证同一笔订单只回补一次。同时我还会有定时对账任务，用库存流水聚合值去核对 Redis 和 DB，发现漂移就以 DB 为准修正 Redis。
