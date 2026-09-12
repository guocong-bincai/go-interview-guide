# 弹幕系统设计（实时性 + 分房间 + 内存缓冲落库）

> 考察频率：★★★★☆  难度：★★★★☆  优先级：P1
> 关键词：实时推送、分房间分片、内存环形缓冲、批量落库、长连接、降级策略

## 核心问题

| 问题 | 描述 |
|------|------|
| 实时性 | 弹幕要在 100ms 内出现在同房间所有观众屏幕上 |
| 高并发 | 热点直播间峰值几十万条/秒写入 |
| 可丢弃 | 弹幕允许少量丢失（不是钱），但绝不能被延迟拖垮主链路 |
| 历史回放 | 用户拖进度条要能看到历史弹幕 |
| 内容安全 | 涉黄涉政、刷屏广告要实时拦截 |

---

## 一、核心设计取舍：弹幕不是"消息"，是"流媒体"

面试第一句就要说清楚**业务特性决定了架构**：

- **允许丢弃**：丢几条弹幕用户无感，但不能因为背压把直播间卡死 → 可以溢出丢弃；
- **弱一致**：不同观众看到的弹幕顺序允许有细微差异 → 不需要全局强一致；
- **时效性强**：弹幕只在"当前时刻"有意义，10 秒前的弹幕推过来就过期了 → 可设置**过期时间**；
- **分房间天然隔离**：每个直播间的弹幕互不相关 → **按 roomID 分片是最自然的水平扩展方式**。

---

## 二、整体架构

```
观众A  观众B  观众C
  │      │      │
  └──────┼──────┘  WebSocket / 长连接（弹幕上行）
         ▼
  ┌──────────────────────────┐
  │   弹幕接入层（网关集群）    │  roomID → 固定网关（一致性哈希）
  └────────────┬─────────────┘
               │ 1. 内容安全过滤（本地词表 + 异步人工审核）
               │ 2. 写本地内存环形缓冲（不落盘，不等 DB）
               ▼
  ┌──────────────────────────┐
  │   弹幕广播（Redis Pub/Sub 或 MQ）  │  按 roomID 频道
  └────────────┬─────────────┘
               │ 3. 广播给同房间的所有网关
               ▼
  ┌──────────────────────────┐
  │   内存环形缓冲（每秒批量 flush）  │
  └────────────┬─────────────┘
               ▼
        Kafka → 存储（Redis / HBase / ClickHouse）
```

**核心优化：写路径全程内存 + 批量落库，绝不让 DB 在弹幕主链路上。**

---

## 三、上行：接收与过滤

```go
// 单房间内存环形缓冲：固定容量，满了覆盖最老的（天然限流，不会 OOM）
type RingBuffer struct {
    mu   sync.Mutex
    buf  []Danmaku
    size int
    pos  int
}

func NewRingBuffer(n int) *RingBuffer {
    return &RingBuffer{buf: make([]Danmaku, n), size: n}
}

func (r *RingBuffer) Push(d Danmaku) {
    r.mu.Lock()
    r.buf[r.pos] = d
    r.pos = (r.pos + 1) % r.size
    r.mu.Unlock()
}

// 批量取出并清空（配合每秒 flush）
func (r *RingBuffer) Drain() []Danmaku {
    r.mu.Lock()
    defer r.mu.Unlock()
    out := make([]Danmaku, 0, r.size)
    for i := 0; i < r.size; i++ {
        if r.buf[i].Content != "" {
            out = append(out, r.buf[i])
            r.buf[i] = Danmaku{}
        }
    }
    return out
}
```

```go
// 接收一条弹幕：过滤 → 广播 → 入缓冲
func (s *Service) OnDanmaku(ctx context.Context, roomID string, d Danmaku) error {
    // 1. 基础校验（长度、频率）
    if len([]rune(d.Content)) > 50 {
        return ErrTooLong
    }
    // 2. 本地词表快速过滤（毫秒级，不看 DB）
    if s.filter.Contains(d.Content) {
        return s.audit.ReportAsync(d) // 异步送人工审核
    }
    // 3. 按 roomID 广播到同房间所有网关
    if err := s.broadcast.Publish(ctx, "danmaku:"+roomID, d); err != nil {
        return err
    }
    // 4. 落入内存缓冲，等待批量落库
    s.buffer(roomID).Push(d)
    return nil
}
```

**关键设计点：**
- **不等待落库**：写 DB 成功才返回会让延迟从 10ms 涨到 50ms+，且 DB 抖动直接拖垮直播；
- **环形缓冲天然限流**：容量写满就覆盖最老的数据，宁可丢弃也不堆积；
- **过滤前置**：内容安全必须在上行同步做（本地词表 + 前缀树/Trie），不能等落库后异步审核。

---

## 四、广播：同房间内实时分发

| 方案 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| 网关内存直接广播 | 零网络开销 | 跨网关房间做不到 | 单网关房间 |
| **Redis Pub/Sub** | 实现简单、多网关 | 消息不持久化、Redis 抖动丢消息 | 中小规模 |
| **MQ（Kafka/RocketMQ）** | 削峰、可回放 | 延迟略高（毫秒级） | 大流量、需要回放 |
| 自建 gossip/网关直连 | 延迟最低 | 实现复杂 | 超大规模 |

```go
// 按 roomID 做一致性哈希，让同一房间尽量落在同一网关（减少跨网关广播）
func gatewayForRoom(roomID string) string {
    h := crc32.ChecksumIEEE([]byte(roomID))
    return gateways[h%uint32(len(gateways))]
}
```

**为什么按 roomID 分片而不是按 userID**：弹幕的"扇出"是以房间为单位的。用 roomID 分片保证**同房间消息单点处理**，避免全局广播风暴。

---

## 五、下行：推给观众 + 过期丢弃

```go
// 观众端订阅房间频道，服务端只推「未过期」的弹幕
type Danmaku struct {
    RoomID  string `json:"room_id"`
    UID     string `json:"uid"`
    Content string `json:"content"`
    Offset  int64  `json:"offset"`  // 相对视频起点的毫秒偏移（用于同步回放）
    SendAt  int64  `json:"send_at"` // 服务端时间戳
}

// 转发时判断是否过期：超过 3 秒的弹幕不再实时推
func forward(d Danmaku) bool {
    return time.Now().UnixMilli()-d.SendAt <= 3000
}
```

> **"过期丢弃"是弹幕设计的关键差异化点**：普通 IM 消息必须送达，弹幕只要"当下"—— 这让你在面试中可以明确说"我们不做可靠投递，换来极低延迟和成本"。

---

## 六、历史弹幕与回放

- 上行时让客户端携带 **视频时间偏移 `offset`**（相对视频开始播放的毫秒数），而不是只存时间戳；
- 存储按 `(roomID/videoID, offset)` 建索引，拖动进度条时按 offset 区间拉取；
- 冷数据归档到 ClickHouse / HBase（按天分区），热数据（最近 1 小时）放 Redis ZSet：`ZADD danmaku:{vid} offset content`。

```go
// 拉取某视频 [start, end] 区间弹幕（ZRANGEBYSCORE）
func GetDanmakuByRange(ctx context.Context, rdb *redis.Client, vid string, start, end int64) ([]string, error) {
    return rdb.ZRangeByScore(ctx, "danmaku:"+vid, &redis.ZRangeBy{
        Min: strconv.FormatInt(start, 10),
        Max: strconv.FormatInt(end, 10),
    }).Result()
}
```

---

## 七、降级策略（面试加分）

| 压力等级 | 降级动作 |
|----------|----------|
| 正常 | 全量广播 |
| 高负载 | 限制单用户发送频率（如 1 条/秒），只广播"高等级用户"弹幕 |
| 极端 | 关闭弹幕实时推送，仅保留发送（用户本地可见），或直接降级为"弹幕池"延迟播放 |
| 恢复 | 逐步恢复频率限制，观察 P99 再全量放开 |

```go
// 单用户频率限制：1 秒最多 1 条
func allowSend(ctx context.Context, rdb *redis.Client, uid string) bool {
    ok, _ := rdb.SetNX(ctx, "danmaku:rate:"+uid, "1", time.Second).Result()
    return ok
}
```

**降级的原则：保核心链路（视频播放不卡），舍非核心（弹幕可丢可延迟）。**

---

## 八、高频追问

### Q1：弹幕和 IM 消息设计有什么本质区别？

| 维度 | IM | 弹幕 |
|------|-----|------|
| 投递语义 | 至少一次、必达 | 尽力而为、允许丢 |
| 顺序 | 会话内严格有序 | 近似有序，按 offset 排 |
| 落库 | 同步/强可靠 | 异步批量，可丢 |
| 过期 | 永久有效 | 3 秒后不再实时推 |
| 扇出 | 点对点/小群 | 大房间广播 |

### Q2：热点房间（几十万人在线）怎么办？

- 广播用 MQ 削峰 + 网关内部再做"分级采样"（只推一部分弹幕）；
- 房间分片：超热房间拆成"分片房间"，观众按 uid 分到 shard；
- 观众端做本地渲染限流（每帧最多渲染 N 条）。

### Q3：如何保证不同用户的弹幕顺序一致？

不强求全局一致。以 **offset（视频时间偏移）** 为准渲染，同一 offset 内按服务端接收时间排。这样即使不同用户收到的顺序略有差异，画面上也在同一时间点出现，观感一致。

### Q4：批量落库会不会丢数据？

进程崩溃时内存缓冲里未 flush 的弹幕会丢（如最近 1 秒）。这对弹幕业务是可接受的取舍。若要求更可靠，可以改成"每 100ms flush"或"WAL 本地预写日志 + 崩溃恢复"，但会牺牲吞吐。

---

## 面试话术

**Q：设计一个弹幕系统。**

> 关键先讲**业务取舍**：弹幕允许丢弃、弱一致、强时效，所以我不会用 IM 那种可靠投递的方案。架构上，接入层按 `roomID` 做一致性哈希把同房间流量收敛到固定网关；上行先做本地词表过滤，然后一条路走 Redis Pub/Sub（或 MQ）广播给同房间所有网关，另一条路写**内存环形缓冲**，每秒批量 flush 到 Kafka 落库，主链路完全不碰 DB。下行只推 3 秒内未过期的弹幕，超时直接丢。历史弹幕用 `(视频, offset)` 做索引，拖进度条时按 offset 区间拉取。最后准备一套分级降级策略，压力大时先限频、再采样、最后关闭实时推送，保视频播放。

**Q：为什么用内存环形缓冲，不用消息队列直接写？**

> 弹幕峰值可能几十万条/秒，如果每条都同步走 MQ/DB，连接和 IO 会成为瓶颈；而且弹幕允许丢弃，没必要为每条消息付出持久化成本。环形缓冲是**有界队列**——满了覆盖最老的，既天然限流又防止内存溢出，配合批量落库把随机写变成顺序写，吞吐能提升一个数量级。
