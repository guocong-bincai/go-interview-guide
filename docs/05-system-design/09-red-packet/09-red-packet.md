# 抢红包系统设计

> 考察频率：★★★★★  优先级：P0  字节/腾讯/美团必考
> 关键词：二倍均值法、Redis原子操作、扣减库存、并发控制

## 1. 核心挑战

```
微信红包春节峰值：~480 亿次请求/天
= 约 55 万 QPS（集中在除夕夜 23:59 前后几分钟）

核心要求：
1. 高并发：海量用户同时抢红包
2. 原子性：总金额不能超发（总发出金额 = 总领取金额）
3. 唯一性：每人只能抢一次
4. 公平性：随机分配，均值接近平均值
5. 低延迟：P99 < 200ms
```

---

## 2. 拆分算法：二倍均值法（核心）

### 为什么用二倍均值

普通随机：`random(0, 总金额/人数)` → 可能有人拿太多，有人拿太少

**二倍均值法保证：每人拿到的金额数学期望 = 总金额/人数（公平）**

### 算法公式

```
设：剩余金额 = M，剩余人数 = N

第 i 个人领取金额 = random(0, M/N × 2)

数学证明：
E(第i人) = ∫₀^{2M/N} x · (1/(2M/N)) dx
         = (1/(2M/N)) · [x²/2]₀^{2M/N}
         = (M/N²) · (2M/N)
         = M/N

结论：每人领取金额的期望值 = 平均值（公平）
```

### Go 实现

```go
import (
    "math/rand"
    "sync"
)

// SplitRedPacket 拆分红包金额（二倍均值算法）
// amount 单位：分（整数，避免浮点精度问题），n 个红包
func SplitRedPacket(amount int64, n int) []int64 {
    if n <= 0 || amount < n {
        return nil
    }

    amounts := make([]int64, 0, n)
    var (
        remainingAmount = amount
        remainingPeople = int64(n)
    )

    for i := 0; i < n-1; i++ {
        // 二倍均值：[0, remainingAmount/remainingPeople*2]
        max := remainingAmount/remainingPeople*2 - 1
        if max < 1 {
            max = 1
        }
        x := rand.Int63n(max) + 1 // 确保至少 1 分

        amounts = append(amounts, x)
        remainingAmount -= x
        remainingPeople--
    }

    // 最后一个人拿剩余全部
    amounts = append(amounts, remainingAmount)
    return amounts
}

// 并发安全版本
type RedPacketSplitter struct {
    mu sync.Mutex
}

func (s *RedPacketSplitter) Split(amount int64, n int) []int64 {
    s.mu.Lock()
    defer s.mu.Unlock()
    return SplitRedPacket(amount, n)
}
```

---

## 3. 方案一：实时计算（适合小并发）

### 预先生成金额，放入 Redis List

```
Redis 数据结构：
key: red_packet:{packet_id}:amounts
type: LIST（LPUSH 金额，RPOP 依次领取）
TTL: 24h

key: red_packet:{packet_id}:taken:{user_id}
value: "1"（已领取标记）
TTL: 24h
```

```go
func initRedPacket(redis *redis.Client, packetID string, amounts []int64) error {
    pipe := redis.Pipeline()
    key := "red_packet:" + packetID + ":amounts"
    for _, amt := range amounts {
        pipe.LPush(key, amt)
    }
    pipe.Expire(key, 24*time.Hour)
    _, err := pipe.Exec()
    return err
}

func grabRedPacket(redis *redis.Client, packetID, userID string) (int64, error) {
    takenKey := "red_packet:" + packetID + ":taken:" + userID

    // 1. 检查是否已领取（SETNX 原子）
    taken, err := redis.SetNX(context.Background(), takenKey, "1", 24*time.Hour).Result()
    if err != nil {
        return 0, err
    }
    if !taken {
        return 0, errors.New("already grabbed")
    }

    // 2. 原子扣减（RPOP 取出一个金额）
    amount, err := redis.RPop(context.Background(), "red_packet:"+packetID+":amounts").Int64()
    if err == redis.Nil {
        // 已被抢完，退款（删除领取标记）
        redis.Del(takenKey)
        return 0, errors.New("no more")
    }
    if err != nil {
        redis.Del(takenKey)
        return 0, err
    }

    return amount, nil
}
```

---

## 4. 方案二：数据库 + Redis 原子扣减（适合大并发）

### 数据模型

```sql
-- 红包活动表
CREATE TABLE red_packet_activity (
    id         BIGINT PRIMARY KEY,
    amount     BIGINT NOT NULL COMMENT '总金额（分）',
    left_amount BIGINT NOT NULL COMMENT '剩余金额',
    count      INT NOT NULL COMMENT '红包个数',
    left_count INT NOT NULL COMMENT '剩余个数',
    created_at DATETIME,
    expire_at  DATETIME,
    INDEX (expire_at)
);

-- 抢红包记录表
CREATE TABLE red_packet_record (
    id          BIGINT PRIMARY KEY,
    activity_id BIGINT NOT NULL,
    user_id     BIGINT NOT NULL,
    amount      BIGINT NOT NULL COMMENT '领取金额（分）',
    grabbed_at  DATETIME,
    UNIQUE KEY uk_activity_user (activity_id, user_id),  -- 保证每人只能抢一次
    INDEX (activity_id)
);
```

### Redis 原子扣减（Lua 脚本）

```lua
-- grab_red_packet.lua：Redis 原子操作，防止超发
-- KEYS[1] = activity:{id}:left_amount
-- KEYS[2] = activity:{id}:left_count
-- KEYS[3] = activity:{id}:taken:{user_id}
-- ARGV[1] = user_id
-- ARGV[2] = expire_seconds

-- 1. 检查是否已领取
if redis.call('EXISTS', KEYS[3]) == 1 then
    return -1  -- 已领取
end

-- 2. 检查剩余数量
local left_count = tonumber(redis.call('GET', KEYS[2]))
if left_count <= 0 then
    return -2  -- 已抢完
end

-- 3. 计算本次可领取金额（剩余金额/剩余人数*2）
local left_amount = tonumber(redis.call('GET', KEYS[1]))
local amount
if left_count == 1 then
    amount = left_amount  -- 最后一个人拿全部
else
    local max = left_amount * 2 / left_count
    amount = math.random(1, max)
end

-- 4. 原子扣减（Lua 脚本整体原子执行）
redis.call('DECRBY', KEYS[1], amount)
redis.call('DECR', KEYS[2])
redis.call('SETEX', KEYS[3], ARGV[2], '1')

return amount
```

```go
func grabRedPacketLua(redis *redis.Client, activityID, userID int64) (int64, error) {
    keyAmount := fmt.Sprintf("activity:%d:left_amount", activityID)
    keyCount  := fmt.Sprintf("activity:%d:left_count", activityID)
    keyTaken  := fmt.Sprintf("activity:%d:taken:%d", activityID, userID)

    result, err := redis.Eval(context.Background(), `
        if redis.call('EXISTS', KEYS[3]) == 1 then
            return -1
        end
        local left_count = tonumber(redis.call('GET', KEYS[2]))
        if left_count <= 0 then
            return -2
        end
        local left_amount = tonumber(redis.call('GET', KEYS[1]))
        local amount
        if left_count == 1 then
            amount = left_amount
        else
            local max = left_amount * 2 / left_count
            amount = math.random(1, max)
        end
        redis.call('DECRBY', KEYS[1], amount)
        redis.call('DECR', KEYS[2])
        redis.call('SETEX', KEYS[3], 86400, '1')
        return amount
    `, []string{keyAmount, keyCount, keyTaken}, userID, 86400).Int64()

    if err != nil {
        return 0, err
    }
    return result, nil
}
```

---

## 5. 数据库最终一致性（异步落库）

### 异步落库：Redis → MySQL

```
抢红包流程：
1. 用户请求 → Redis Lua 原子扣减 + 返回金额 ✅（快）
2. 记录领取 → 异步写入 MQ
3. MQ Consumer → 落库 MySQL
4. 定时任务 → 对账（Redis vs MySQL）
```

```go
// 异步落库（防止数据库成为瓶颈）
func asyncSaveRecord(activityID, userID, amount int64) {
    msg, _ := json.Marshal(GrabEvent{activityID, userID, amount})
    mq.Send("red_packet_grab", msg)
}

// MQ Consumer
func consumeGrabEvent(ctx context.Context, msg *sarama.ConsumerMessage) {
    var event GrabEvent
    json.Unmarshal(msg.Value, &event)

    // 落库（带唯一约束，自动去重）
    _, err := db.ExecContext(ctx, `
        INSERT IGNORE INTO red_packet_record (activity_id, user_id, amount, grabbed_at)
        VALUES (?, ?, ?, NOW())
    `, event.ActivityID, event.UserID, event.Amount)
    // INSERT IGNORE：已存在则忽略（防止重复落库）
}
```

---

## 6. 完整流程图

```
用户发起抢红包请求
         ↓
    反作弊检查（频率限制、账号异常）
         ↓
    Redis SETNX 检查是否已领取
         ↓（未领取）
    Lua 脚本原子扣减 Redis 余额
         ↓（成功）
    返回领取金额给用户
         ↓（同时异步）
    发 MQ 消息 → Consumer 落库 MySQL
         ↓
    定时对账：Redis left_amount vs MySQL SUM(amount)
```

---

## 7. 超发问题解决

**超发原因（不解决）：**
```
T1: 用户A查询 left_amount = 10，left_count = 2
T2: 用户B查询 left_amount = 10，left_count = 2
T3: 用户A扣减 left_amount = 5（成功，left_amount = 5）
T4: 用户B扣减 left_amount = 5（成功，left_amount = 0）
问题：总发出 10 分，但只剩 2 份，超发了！
```

**解决：Lua 脚本整体原子执行**
```lua
-- Lua 脚本中 GET 和 DECRBY 是一个原子操作
-- 不会有并发问题，因为 Redis 是单线程执行 Lua 脚本
local left_amount = tonumber(redis.call('GET', KEYS[1]))
local amount = math.random(1, left_amount * 2 / left_count)
redis.call('DECRBY', KEYS[1], amount)  -- 这里不会被打断
```

---

## 高频追问

### Q1：Redis 挂了怎么办？

**降级方案：**
1. **读取 DB**：从 MySQL 读取 `SUM(领取金额)` 实时计算剩余
   - 问题：DB 查询 QPS 不够
2. **限流 + 队列**：请求入 MQ，Redis 恢复后逐步处理
3. **预热到多个 Redis**：Redis Cluster，主备切换

### Q2：如何保证每人只能抢一次？

**三层防护：**
1. **Redis SETNX**：第一次请求时设置标记（最快）
2. **MySQL 唯一索引**：`UNIQUE KEY uk_activity_user (activity_id, user_id)`（兜底）
3. **业务检查**：落库时 INSERT IGNORE（最终保证）

### Q3：金额精度问题（浮点数）

**必须用整数（分），不能用 float：**
```go
// ❌ 错误：浮点精度问题
float64 amount = 0.01  // 0.0099999999

// ✅ 正确：用整数（分）
int64 amount = 1  // 1 分 = 0.01 元
```

### Q4：红包过期未领取怎么处理？

**超时退余额：**
```go
// 定时任务：扫描过期红包，退还余额
func refundExpiredPackets() {
    db.Exec(`UPDATE red_packet_activity
        SET left_amount = amount, left_count = count
        WHERE expire_at < NOW() AND left_count > 0`)
}
```
