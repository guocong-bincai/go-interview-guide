# 消息推送系统设计（在线/离线通道 + 多厂商 + 去重聚合）

> 考察频率：★★★★☆  难度：★★★★☆  优先级：P1
> 关键词：长连接网关、路由表、APNs/FCM/厂商通道、消息去重、离线补推、频控降噪

## 核心问题

| 问题 | 描述 |
|------|------|
| 在线投递 | 百万长连接，如何快速定位用户在哪台网关 |
| 离线补偿 | 用户不在线，消息怎么存、上线怎么补 |
| 多通道 | 自建长连接 / APNs / FCM / 小米华为 OPPO 厂商通道如何统一调度 |
| 可靠性 | 消息不丢、不重、有序（至少不严重乱序） |
| 降噪 | 如何避免同一事件轰炸用户（去重、聚合、静默时段） |

---

## 一、整体架构

```
业务系统 ──▶ 推送服务 ──▶ 消息队列(Kafka) ──▶ 推送 Worker
                 │                              │
                 │                              ├─ 查询路由表：uid → gatewayID
                 │                              ├─ 在线 → 投递给长连接网关
                 │                              └─ 离线 → 走厂商通道 / 存离线消息
                 ▼
            ┌──────────────────────────────┐
            │ 长连接网关集群（Go + epoll）  │
            │  gateway-1 … gateway-N       │
            └──────────────────────────────┘
                              ▲
                              │ 连接建立时注册路由
                    ┌─────────┴─────────┐
                    │ 路由表 (Redis)     │  uid → gatewayID, online status, TTL
                    └───────────────────┘
```

---

## 二、长连接网关与路由表

### 1. 路由表设计

```go
// key: push:route:{uid}   value: gatewayID   带上 TTL（心跳续期）
// 网关进程启动时写自己的标识，连接建立/心跳时刷新 TTL
func RegisterRoute(ctx context.Context, rdb *redis.Client, uid, gatewayID string) error {
    return rdb.Set(ctx, "push:route:"+uid, gatewayID, 90*time.Second).Err()
}

// 心跳续期（客户端每 30s 一次心跳，TTL 90s 给 3 次容错）
func RefreshRoute(ctx context.Context, rdb *redis.Client, uid, gatewayID string) error {
    // 用 Lua 防止过期后误续：只有值还是自己才续期
    script := redis.NewScript(`
        if redis.call('GET', KEYS[1]) == ARGV[1] then
            return redis.call('EXPIRE', KEYS[1], 90)
        end
        return 0
    `)
    return script.Run(ctx, rdb, []string{"push:route:" + uid}, gatewayID).Err()
}
```

> 关键点：**TTL 续期必须校验值还是自己**，否则会写出"幽灵路由"——网关 A 已断开、网关 B 接了同一用户，A 的旧心跳把路由又续回 A。

### 2. 投递流程

```go
func Deliver(ctx context.Context, msg PushMessage) error {
    gatewayID, err := rdb.Get(ctx, "push:route:"+msg.UID).Result()
    if err == redis.Nil {
        // 离线：走离线路径
        return deliverOffline(ctx, msg)
    }
    if err != nil {
        return err
    }
    // 在线：通过内部 RPC 把消息转给对应网关（网关再写回 socket）
    return gatewayClient.Send(ctx, gatewayID, msg)
}
```

**规模扩展**：网关数量上千时，路由表用 Redis Cluster 分片（按 uid hash）；网关间通信走内部 RPC/消息总线，避免全连接。

---

## 三、离线消息与补推

| 策略 | 说明 |
|------|------|
| 离线消息表 | 写入 `offline_msg(uid, msg_id, payload, create_time)`，上线后按游标拉取 |
| 每用户队列 | Redis List / Stream 存最近 N 条（如 1000 条），超出丢弃最老 |
| 时间窗口 | 只保留 7 天离线消息，过期不补 |
| 上线拉取 | 客户端携带本地最大 `msg_id`，服务端只回增量（游标，避免重复推） |

```go
// 用 Redis Stream 做每用户离线队列（天然支持 ID 游标 + 上限裁剪）
func pushOffline(ctx context.Context, rdb *redis.Client, uid string, payload []byte) error {
    key := "offline:stream:" + uid
    _, err := rdb.XAdd(ctx, &redis.XAddArgs{
        Stream: key,
        MaxLen: 1000, // 近似裁剪，避免大 Key
        Approx: true,
        Values: map[string]interface{}{"p": string(payload)},
    }).Result()
    return err
}

// 上线后按 "上次读到的 ID" 拉增量
func pullOffline(ctx context.Context, rdb *redis.Client, uid, lastID string) ([]redis.XMessage, error) {
    return rdb.XRangeN(ctx, "offline:stream:"+uid, "("+lastID, "+", 100).Result()
}
```

**为什么用"拉"而不是"推"**：离线消息量不可控，推送端不知道用户何时上线；由客户端上线后**带游标拉取**，天然幂等、不会重复。

---

## 四、多通道统一调度（APNs / FCM / 厂商通道）

国内 Android 生态必须对接多厂商推送（小米、华为、OPPO、vivo、荣耀…），海外用 FCM，iOS 用 APNs。

```go
type Channel interface {
    Name() string
    Push(ctx context.Context, deviceToken string, msg PushMessage) error
    Supports(msg PushMessage) bool // 该通道是否支持此消息类型
}

// 调度顺序：自建长连接 > 厂商通道 > 短信（降级）
func (s *PushService) Dispatch(ctx context.Context, u User, msg PushMessage) error {
    // 1. 在线优先走自建长连接（最实时、零成本）
    if u.Online {
        if err := s.gateway.Send(ctx, u.GatewayID, msg); err == nil {
            return nil
        }
        // 长连接失败 → 继续尝试厂商通道
    }
    // 2. 离线/失败：按设备平台选厂商通道
    ch := s.pickChannel(u)
    if err := ch.Push(ctx, u.DeviceToken, msg); err == nil {
        return nil
    }
    // 3. 重要消息降级到短信
    if msg.Priority == PriorityHigh {
        return s.sms.Send(ctx, u.Phone, msg.Title)
    }
    return ErrAllChannelsFailed
}
```

### 通道选择的三个维度

| 维度 | 说明 |
|------|------|
| 到达率 | 自建长连接 ~95%（App 存活时），厂商通道 ~70-90%，短信 ~99% |
| 成本 | 长连接几乎为 0，厂商通道按量计费但便宜，短信 0.03-0.05 元/条 |
| 实时性 | 长连接秒级，厂商通道受厂商调度影响可能延迟数分钟 |

---

## 五、降噪：去重 / 聚合 / 频控 / 静默时段

这是**消息推送最容易被面试官追问的加分点**：推得多不如推得准。

```go
// 1. 去重：同一业务事件 ID 只推一次
func dedup(ctx context.Context, uid, eventID string) (bool, error) {
    return rdb.SetNX(ctx, "push:dedup:"+uid+":"+eventID, "1", 10*time.Minute).Result()
}

// 2. 聚合：同一用户 5 分钟内的同类消息合并成一条（如"3 人赞了你"）
func aggregate(ctx context.Context, uid, category string) {
    // 用 Redis Hash 计数，定时/达阈值时合并发送
    rdb.HIncrBy(ctx, "push:agg:"+uid, category, 1)
}

// 3. 频控：每用户每天最多 N 条营销推送
func rateLimit(ctx context.Context, uid string, limit int64) (bool, error) {
    key := "push:quota:" + uid + ":" + time.Now().Format("20060102")
    n, err := rdb.Incr(ctx, key).Result()
    if err != nil {
        return false, err
    }
    if n == 1 {
        rdb.Expire(ctx, key, 24*time.Hour)
    }
    return n <= limit, nil
}

// 4. 静默时段：22:00 - 08:00 不推非紧急消息，延迟到次日早晨
func inQuietHours(t time.Time) bool {
    h := t.Hour()
    return h >= 22 || h < 8
}
```

> **优先级分级**：`P0`（支付/安全，立即推、可穿透静默）、`P1`（IM 消息，实时推）、`P2`（社交互动，可聚合）、`P3`（营销，严格频控）。分级调度是推送系统设计题的高分点。

---

## 六、可靠性与顺序

| 目标 | 手段 |
|------|------|
| 不丢 | 业务侧先落库（Outbox）→ 发 MQ → 消费投递；投递成功才 ACK；失败重试（指数退避） |
| 不重 | 消息携带 `msgID`，客户端按 `msgID` 去重；Redis SETNX 短期去重 |
| 有序 | 同一会话的消息按 `seq` 排序；投递端用 MQ 分区键 = `uid`，保证同一用户消息单分区有序 |
| 可观测 | 投递成功率、P99 延迟、各通道到达率、失败原因分布，全部上报 |

```go
// MQ 分区键用 uid，保证同一用户的消息进入同一分区 → 单消费者顺序处理
producer.Send(ctx, &kafka.Message{
    Topic: "push",
    Key:   []byte(uid),
    Value: payload,
})
```

---

## 七、高频追问

### Q1：百万长连接怎么扛？

- Go + epoll（netpoll），单机可支撑 10 万+ 连接；
- 连接分布到多台网关，用一致性哈希/路由表寻址；
- 关键资源：文件描述符（`ulimit -n`）、内核 `somaxconn`、内存（每连接缓冲区）；
- 心跳保活（30s）+ 空闲连接回收。

### Q2：网关掉线，用户路由怎么清理？

路由依赖 TTL（90s）自动过期，不必显式删除。网关重启后连接重新注册。**TTL 是分布式系统中"活性检测"最省心的方案**，避免依赖显式注销。

### Q3：如何保证消息不重复推给用户？

- 离线消息用**游标拉取**（客户端记录已读 offset）；
- 在线消息带 `msgID`，客户端本地去重；
- 服务端短期 `SETNX` 去重（防止重试导致重复）。

### Q4：营销推送和 IM 消息要隔离吗？

必须。**IM/通知/营销是三条独立通道+独立配额**，避免营销把 IM 的配额和通道打满。物理上可用不同 Topic、不同 Worker 池做隔离（舱壁模式）。

---

## 面试话术

**Q：设计一个推送系统。**

> 我会分四层：**接入层**是长连接网关集群，连接建立时把 `uid → gatewayID` 写进 Redis 路由表，用 TTL + 心跳续期（续期要校验值是自己，防止幽灵路由）；**调度层**收到消息先查路由，在线直接投递，离线走离线队列或厂商通道；**通道层**统一抽象 APNs/FCM/厂商通道/短信，按到达率和成本分级调度；**治理层**做去重、聚合、频控、静默时段和优先级分级。可靠性上，消息先落 Outbox 再进 MQ，分区键用 uid 保证同一用户消息有序，客户端用 msgID 去重。

**Q：离线消息为什么会重复推送？怎么解决？**

> 因为 MQ 是 at-least-once，重试和"上线拉取"之间会有重叠窗口。解决方式是**游标语义**：离线消息存 Redis Stream/DB，客户端上线时带上次读到的最大 ID，服务端只返回增量；同时客户端按 msgID 本地去重。两者结合就能做到"至少一次投递 + 客户端幂等"。
