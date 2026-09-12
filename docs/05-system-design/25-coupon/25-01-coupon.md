# 优惠券/营销系统设计（并发领取 + 防重 + 超时回滚）

> 考察频率：★★★★☆  难度：★★★★☆  优先级：P1
> 关键词：券模板与券实例、并发领券、Redis Lua、唯一索引、防刷、超时回滚、核销幂等

## 核心问题

| 问题 | 描述 |
|------|------|
| 超发 | 限量 1 万张券，发放 1.2 万张就是资损 |
| 重复领 | 同一用户反复点，领到多张 |
| 防刷 | 机器批量注册小号薅券 |
| 回滚 | 领了不用（未使用过期）库存怎么处理 |
| 核销 | 下单核销，如何保证一张券只用一次、且能回滚 |

---

## 一、数据模型：模板 vs 实例

```
券模板 coupon_template            用户券实例 user_coupon
┌──────────────────────┐         ┌──────────────────────────┐
│ id                   │ 1    N  │ id                       │
│ name                 │────────▶│ template_id              │
│ total_stock  总库存   │         │ user_id                  │
│ remaining_stock 剩余  │         │ status: UNUSED/USED/EXPIRED│
│ per_user_limit 限领   │         │ order_id   核销订单号      │
│ start/end_time       │         │ receive_time / use_time   │
└──────────────────────┘         └──────────────────────────┘
                                  UNIQUE(user_id, template_id, seq)
```

**关键：模板表管"库存"，实例表管"归属"。** 领券 = 模板库存 -1 + 插入一条实例。

---

## 二、并发领券：三重防线

### 第一道：Redis 预扣（扛流量）

```lua
-- KEYS[1] = coupon:stock:{templateID}     剩余库存
-- KEYS[2] = coupon:user:{templateID}      已领用户集合
-- ARGV[1] = userID
-- ARGV[2] = perUserLimit
local stock = redis.call('GET', KEYS[1])
if not stock then return -1 end                 -- 未预热
if tonumber(stock) <= 0 then return 0 end       -- 已领完
local cnt = redis.call('SCARD', KEYS[2])        -- 简化：集合去重
if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then
    return 2                                     -- 已领取
end
redis.call('DECR', KEYS[1])
redis.call('SADD', KEYS[2], ARGV[1])
return 1
```

```go
var receiveScript = redis.NewScript(`
local stock = redis.call('GET', KEYS[1])
if not stock then return -1 end
if tonumber(stock) <= 0 then return 0 end
if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then return 2 end
redis.call('DECR', KEYS[1])
redis.call('SADD', KEYS[2], ARGV[1])
return 1
`)

func Receive(ctx context.Context, rdb *redis.Client, templateID, uid string) (int, error) {
    return receiveScript.Run(ctx, rdb,
        []string{"coupon:stock:" + templateID, "coupon:user:" + templateID},
        uid).Int()
}
```

> 若 `per_user_limit > 1`，不能用 Set，改用 `HINCRBY coupon:user:{tid} {uid} 1` 计数并判断上限。

### 第二道：DB 兜底（保不超发）

```sql
-- 扣减库存：条件更新保证不超发
UPDATE coupon_template
SET remaining_stock = remaining_stock - 1
WHERE id = ? AND remaining_stock > 0;
-- 影响行数 = 0 → 库存已空，回滚 Redis

-- 插入实例：唯一索引保证不重复领
INSERT INTO user_coupon(user_id, template_id, seq, status)
VALUES (?, ?, ?, 'UNUSED');
-- 唯一键冲突 → 已领过，回滚 Redis
```

```go
func ReceiveWithDB(ctx context.Context, tx *sql.Tx, templateID string, uid int64) error {
    res, err := tx.ExecContext(ctx,
        `UPDATE coupon_template SET remaining_stock = remaining_stock - 1
         WHERE id = ? AND remaining_stock > 0`, templateID)
    if err != nil {
        return err
    }
    if n, _ := res.RowsAffected(); n == 0 {
        return ErrSoldOut
    }
    _, err = tx.ExecContext(ctx,
        `INSERT INTO user_coupon(user_id, template_id, seq, status) VALUES (?,?,0,'UNUSED')`,
        uid, templateID)
    if err != nil {
        return ErrAlreadyReceived // 唯一索引冲突
    }
    return nil
}
```

### 第三道：防刷（保成本）

```go
// 1. 设备指纹 + IP 限流：同一设备/IP 每分钟最多 N 次领券
// 2. 新用户风控：注册 < 24h 的账号领大额券需人工/风控审核
// 3. 行为特征：同一批 IP 段、同一机型集中领券 → 拉黑
func riskCheck(ctx context.Context, rdb *redis.Client, uid, deviceID, ip string) bool {
    ok1, _ := rdb.SetNX(ctx, "coupon:dev:"+deviceID, "1", time.Minute).Result()
    ok2, _ := rdb.SetNX(ctx, "coupon:ip:"+ip, "1", time.Minute).Result()
    return ok1 && ok2
}
```

---

## 三、超时回滚：领了不用怎么办

| 场景 | 处理 |
|------|------|
| 券有过期时间 | 到期状态置 `EXPIRED`，**不回滚模板库存**（券已发出，是营销成本，已计入预算）|
| 限量抢券但用户取消 | 若业务允许"取消后释放名额"，走回滚（较少见）|
| 库存预售/预约 | 预约占用名额，超时未支付的释放 |

```go
// 定时/延迟消息：券到期自动置为过期
func expireCoupons(ctx context.Context) {
    _, _ = db.ExecContext(ctx,
        `UPDATE user_coupon SET status='EXPIRED'
         WHERE status='UNUSED' AND expire_time < NOW()`)
}
```

> **面试要点**：绝大多数平台的优惠券**过期不退回总库存**，因为券代表营销预算已消耗；是否回滚要由业务定义，面试时主动说明这个取舍，比盲目"回滚"更显经验。

---

## 四、核销：幂等 + 回滚

```go
// 核销：状态机 CAS，只有 UNUSED 能变 USED
func UseCoupon(ctx context.Context, tx *sql.Tx, couponID int64, orderID string) error {
    res, err := tx.ExecContext(ctx,
        `UPDATE user_coupon SET status='USED', order_id=?, use_time=NOW()
         WHERE id=? AND status='UNUSED'`, orderID, couponID)
    if err != nil {
        return err
    }
    if n, _ := res.RowsAffected(); n == 0 {
        return ErrCouponUsedOrNotExist // 已使用或不存在
    }
    // 记录核销流水（幂等审计）
    _, err = tx.ExecContext(ctx,
        `INSERT INTO coupon_use_log(coupon_id, order_id, use_time) VALUES (?,?,NOW())`,
        couponID, orderID)
    return err
}

// 订单取消 → 券退回（幂等）
func RefundCoupon(ctx context.Context, tx *sql.Tx, couponID int64, orderID string) error {
    res, err := tx.ExecContext(ctx,
        `UPDATE user_coupon SET status='UNUSED', order_id=NULL, use_time=NULL
         WHERE id=? AND order_id=? AND status='USED'`, couponID, orderID)
    if err != nil {
        return err
    }
    if n, _ := res.RowsAffected(); n == 0 {
        return nil // 已退回或不适用，幂等忽略
    }
    return nil
}
```

**核心：所有状态变更都用"带条件的 UPDATE + 判断影响行数"，天然并发安全且幂等。**

---

## 五、热点券（爆款大额券）优化

和大促库存同源，爆款券的 `coupon:stock:{id}` 也会成为热 Key：

- **分段库存**：把 1 万张券拆成 10 个分片 Key，各 1000 张，用户 hash 分流；
- **本地缓存**：券模板详情（面额、门槛、有效期）本地缓存 + 定时刷新，读路径不打 Redis；
- **闸门限流**：活动前用"秒杀式"限流（预约码/排队页），把瞬时流量削平；
- **队列化**：领券请求先入 MQ，异步发放（牺牲实时性换稳定性）。

---

## 六、一致性对账

```sql
-- 对账 SQL：Redis 剩余库存 vs DB 剩余库存
-- 差异超过阈值 → 以 DB 为准回写 Redis
SELECT t.id, t.remaining_stock,
       (t.total_stock - (SELECT COUNT(*) FROM user_coupon u WHERE u.template_id = t.id)) AS calc_stock
FROM coupon_template t
WHERE t.remaining_stock <> (
    t.total_stock - (SELECT COUNT(*) FROM user_coupon u WHERE u.template_id = t.id)
);
```

> 兜底思路始终一致：**Redis 是加速器，DB 是真相源。** 发现不一致就以 DB 为准修正 Redis。

---

## 七、高频追问

### Q1：Redis 扣减成功但 DB 插入失败，怎么办？

两个选择：
1. **回滚 Redis**：`INCR` 加回库存 + `SREM` 移除用户，返回失败给前端（用户体验差但准确）；
2. **异步补偿**：把失败请求写入重试队列，由消费者重试落库，成功后券以"待发放"状态最终到达（体验好）。

生产上常见做法是：**同步返回失败 + 写入补偿队列**，补偿在别人释放库存或扩容后重试。

### Q2：同一个用户并发点 10 次领券？

- Redis `SISMEMBER/HINCRBY` 在 Lua 原子脚本里判断，只有一次成功；
- DB 唯一索引 `UNIQUE(user_id, template_id, seq)` 兜底；
- 前端按钮置灰只是体验优化，**服务端幂等才是根本**。

### Q3：券的库存扣减为什么要用 Redis + DB 双写？

- Redis 扛并发（所有请求先在 Redis 过滤，绝大部分请求失败即返回）；
- DB 保正确（条件更新 + 唯一索引，即使 Redis 出问题也不超发）；
- 两者是"性能层 + 正确性层"的关系，不是二选一。

### Q4：怎么设计券的过期提醒和自动核销？

- 券到期前 24h 发推送（复用推送系统，P2 优先级）；
- 定时任务把过期券置 `EXPIRED`（分片扫描，避免大事务）；
- 核心指标：领取率、使用率、过期率、ROI（营销成本 / 带动 GMV）。

---

## 面试话术

**Q：设计一个优惠券系统。**

> 我会分三层讲。**数据模型**上，券模板管库存、用户券实例管归属，两者一对多。**并发领取**用三重防线：Redis Lua 脚本原子完成"库存校验 + 用户去重 + 扣减"，把绝大多数流量挡在 DB 之前；DB 用 `UPDATE ... WHERE remaining_stock > 0` 条件更新保证不超发，用 `UNIQUE(user_id, template_id)` 保证不重领；最外层再加设备/IP 限流和风控防薅羊毛。**状态流转**上，核销和退回都用带条件的 UPDATE + 判断影响行数，天然幂等。最后配一个对账任务，以 DB 为准校准 Redis 库存。

**Q：爆款券的库存 Key 被打爆怎么办？**

> 和秒杀库存同源，核心是**把热点 Key 拆开**：把总库存按 hash 分成 N 个分片 Key，用户请求分流到不同分片，写压力均摊；券的模板信息用本地缓存，读路径不打 Redis；再配合预约码/排队页做闸门限流，把瞬时流量削平。
