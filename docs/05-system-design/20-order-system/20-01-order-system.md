# 订单系统设计（状态机 + 超时关单 + 分库分表）

> 考察频率：★★★★★  难度：★★★★☆  优先级：P0
> 关键词：订单状态机、超时关单、延迟队列、时间轮、RocketMQ 延时消息、分库分表、幂等

## 核心问题

| 问题 | 描述 |
|------|------|
| 状态流转 | 订单状态多、流转复杂，如何防止非法流转（已取消又变已支付）？ |
| 超时关单 | 15 分钟未支付要自动取消并回滚库存，怎么做到准点？ |
| 幂等 | 用户重复点击、MQ 重复投递，如何保证不重复下单？ |
| 扩展性 | 单表上亿订单，如何分库分表且不影响按用户查询？ |
| 一致性 | 订单、库存、支付三方状态如何最终一致？ |

---

## 一、整体架构

```
              ┌────────────┐
 用户下单  →  │  订单服务   │  ← 幂等校验（requestId / 唯一索引）
              └─────┬──────┘
                    │ 1. 本地事务写订单（待支付）
                    ▼
              ┌────────────┐
              │  消息队列   │  ← 事务消息 / Outbox，保证「订单落库」与「事件发出」原子
              └─────┬──────┘
        ┌───────────┼────────────┐
        ▼           ▼            ▼
   ┌────────┐  ┌────────┐   ┌──────────┐
   │库存服务 │  │支付服务 │   │ 超时关单  │
   └────────┘  └────────┘   └──────────┘
```

设计原则：**订单表只存状态，状态变更全部走状态机；重活（库存、通知、风控）异步化。**

---

## 二、订单状态机

### 1. 状态定义与合法流转

```
待支付(PENDING) ──支付成功──▶ 已支付(PAID) ──发货──▶ 已发货(SHIPPED) ──签收──▶ 已完成(COMPLETED)
      │                                                              │
      ├──超时/取消──▶ 已取消(CANCELED)                                └──退款──▶ 已退款(REFUNDED)
      └──支付失败──▶ 支付失败(PAY_FAILED)
```

**核心规则：状态只能沿有向图单向流转，任何"跳变"都必须拒绝。**

```go
package order

import "errors"

type Status int

const (
    StatusPending Status = iota // 待支付
    StatusPaid                  // 已支付
    StatusShipped               // 已发货
    StatusCompleted             // 已完成
    StatusCanceled              // 已取消
    StatusRefunded              // 已退款
)

// allowedTransitions 定义状态机的合法边
var allowedTransitions = map[Status]map[Status]bool{
    StatusPending: {StatusPaid: true, StatusCanceled: true},
    StatusPaid:    {StatusShipped: true, StatusRefunded: true},
    StatusShipped: {StatusCompleted: true, StatusRefunded: true},
    StatusCompleted: {},
    StatusCanceled:  {},
    StatusRefunded:  {},
}

func (from Status) CanTransferTo(to Status) bool {
    return allowedTransitions[from][to]
}

// Transfer 带版本号的 CAS 状态流转，天然幂等 + 防并发覆盖
func Transfer(orderID string, from, to Status) error {
    if !from.CanTransferTo(to) {
        return errors.New("illegal status transition")
    }
    // UPDATE orders SET status=?, version=version+1
    // WHERE id=? AND status=? AND version=?
    // 影响行数为 0 == 状态已被别人改过，直接返回，不重试
    return nil
}
```

### 2. 为什么用"状态 + 版本号 CAS"而不是 `if` 判断

- 支付回调、超时关单、用户取消可能**同时到达**；
- 用 `WHERE status = 期望值` 做 CAS，只有一个能成功，其余影响行数为 0，天然互斥；
- 配合 `version` 列可防止"读-改-写"之间的覆盖。

---

## 三、超时关单：四种方案对比

| 方案 | 准点性 | 实现成本 | 海量订单 | 生产推荐度 |
|------|--------|----------|----------|-----------|
| 定时扫表（每分钟 `WHERE status=PENDING AND create_time<?`）| 差（最多延迟 1min，且越积越多）| 低 | 差（全表扫，索引成本高）| ⭐⭐ |
| **延迟队列 / MQ 延时消息** | 好 | 中 | 好 | ⭐⭐⭐⭐⭐ |
| **时间轮（Timing Wheel）** | 很好 | 高 | 很好 | ⭐⭐⭐⭐ |
| Redis ZSet + 定时轮询 | 好 | 低 | 一般 | ⭐⭐⭐ |

### 方案 A：RocketMQ 延时消息（最常用）

```go
// 下单成功后发送 15 分钟延时消息
func sendDelayClose(orderID string) error {
    msg := &rocketmq.Message{
        Topic: "order_timeout",
        Body:  []byte(orderID),
        DelayTimeLevel: 9, // RocketMQ 4.x 固定档位，15min 对应档位需换算
    }
    _, err := producer.SendSync(context.Background(), msg)
    return err
}

// 消费者：到点后尝试关单
func consumeTimeout(ctx context.Context, orderID string) error {
    // CAS：只有仍处于 PENDING 才取消，已支付则直接忽略
    return Transfer(orderID, StatusPending, StatusCanceled)
}
```

> 注意：RocketMQ 4.x 延时消息是**固定 18 个档位**（1s/5s/10s…2h），不支持任意时间；RocketMQ 5.x 支持任意时刻。若用 Kafka，原生无延时消息，需自建时间轮或落库轮询。

### 方案 B：时间轮（Netty `HashedWheelTimer` 思想）

```go
// 简易时间轮：每个槽位是一个双向链表，指针每秒推进一格
// 15 分钟 = 900 秒，槽位数取 2 的幂（如 1024），时间轮转一圈 1024s
// 任务落在 (currentSlot + 900) % 1024 槽位，指针扫到即触发
type TimingWheel struct {
    slots  []*list.List
    tick   time.Duration // 每格时长
    cur    int
    mu     sync.Mutex
}
```

时间轮优点：**内存级、O(1) 插入**，适合单机内大量定时任务；缺点：需要自己做持久化（进程重启会丢任务），生产上通常"时间轮 + DB 兜底扫表"双保险。

### 方案 C：Redis ZSet 轮询

```go
// score = 关单时间戳(ms)，member = orderID
rdb.ZAdd(ctx, "order:timeout", redis.Z{
    Score:  float64(time.Now().Add(15 * time.Minute).UnixMilli()),
    Member: orderID,
})

// 定时任务每 5s 拉一批到期的
func pollTimeout(ctx context.Context) {
    now := time.Now().UnixMilli()
    ids, _ := rdb.ZRangeByScore(ctx, "order:timeout", &redis.ZRangeBy{
        Min: "0", Max: strconv.FormatInt(now, 10), Offset: 0, Count: 200,
    }).Result()
    for _, id := range ids {
        _ = Transfer(id, StatusPending, StatusCanceled)
        rdb.ZRem(ctx, "order:timeout", id)
    }
}
```

### 兜底：为什么一定要"定时扫表"作为最后一道防线

MQ 丢消息、时间轮重启丢任务、Redis 主从切换丢数据 —— 任何一种都可能漏单。**每天/每小时跑一次对账扫描**（`WHERE status=PENDING AND create_time < NOW()-INTERVAL 1 HOUR`），把漏网订单补齐并取消，是生产环境的最后一道保险。

---

## 四、订单号生成

| 方案 | 示例 | 优点 | 缺点 |
|------|------|------|------|
| 数据库自增 | 1000001 | 简单、趋势递增 | 分库分表后不唯一、暴露量级 |
| UUID | 550e8400… | 全局唯一 | 无序、索引写入差、可读性差 |
| **雪花算法 Snowflake** | 1946xxxx… | 趋势递增、无中心依赖 | 依赖时钟，需回拨处理 |
| **日期 + 分片 + 自增序列** | 2026091310000123 | 可读、含分片信息 | 需自己实现序列 |

```go
// 订单号 = 日期(8) + 业务码(2) + 分片号(4) + 自增序列(6)
// 便于从订单号直接定位分片表，避免全库扫描
func GenOrderNo(shardID int, seq uint64) string {
    return fmt.Sprintf("%s%02d%04d%06d",
        time.Now().Format("20060102"),
        10, // 业务码：普通订单
        shardID,
        seq%1000000,
    )
}
```

> **基因法**：把 `user_id` 的低位（如 `user_id % 1024`）嵌入订单号的某几位，这样"按 user_id 查订单"和"按订单号路由分片"用的是同一套分片键，避免跨 1024 张表扫描。

---

## 五、分库分表

- **分片键选择**：B 端交易查支付单/订单号，C 端用户查自己的订单 —— 选 `user_id` 分片 + 订单号基因，两者都能命中单分片；
- **路由表**：`user_id % 1024` → 库 `% 16`，表 `% 64`；
- **历史归档**：完成超过 1 年的订单迁到冷库/ClickHouse，主库只留热数据；
- **注意**：跨分片的聚合查询（如"平台昨日 GMV"）不要走订单主库，走 **Binlog → 数仓/OLAP**。

---

## 六、高频追问

### Q1：用户重复提交下单怎么办？

三层防护：
1. **前端**：按钮置灰 + 提交后阻塞；
2. **服务端幂等**：用客户端生成的 `requestId` 做 Redis `SETNX`（或 `INSERT ... ON DUPLICATE KEY`）去重；
3. **DB 兜底**：`UNIQUE(user_id, request_id)` 唯一索引，重复插入直接报错。

```go
// 幂等：requestId 5 分钟内只能成功一次
ok, err := rdb.SetNX(ctx, "order:req:"+requestID, "1", 5*time.Minute).Result()
if !ok {
    return ErrDuplicateSubmit
}
```

### Q2：支付回调重复到达会重复加库存吗？

不会。支付回调先做**幂等判断**（`pay_trade_no` 唯一索引 + 订单状态 CAS），只有 `PENDING → PAID` 成功的那一次才发"支付成功事件"，库存/积分等下游消费事件时再做一次幂等（消费记录表 + 唯一索引）。

### Q3：如何防止"取消订单"和"支付成功"竞态？

两者都对订单行做 `WHERE status=PENDING` 的 CAS 更新，只有一个能改成功。支付侧失败则原路退款，取消侧失败则忽略。**永远相信数据库的条件更新，而不是先查后改。**

---

## 面试话术

**Q：设计一个订单系统，你会怎么讲？**

> 我会分四块讲：第一，订单的核心是**状态机**，我会用状态枚举 + 版本号 CAS 保证状态只能合法流转、且天然幂等；第二，**超时关单**我用 MQ 延时消息（或时间轮）做主方案，再加一个定时扫表做兜底，防止消息丢失；第三，**幂等**靠 requestId 去重 + 数据库唯一索引双保险；第四，数据量大了按 `user_id` 分库分表，订单号里带分片基因，保证按用户和按订单号都能单分片命中。

**Q：为什么不用定时任务扫表关单就够了？**

> 单表小的时候够用，但订单量上千万后，每分钟全表扫 `status=PENDING` 的索引区间会越来越慢，而且关闭精度差。MQ 延时消息可以做到准点触发、压力均匀，扫表我只会留作兜底对账。
