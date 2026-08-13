# 接口幂等性设计

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：接口幂等、唯一请求ID、去重表、唯一约束、Token 机制、状态机、乐观锁、重复提交

---

## 面试官考察意图

"用户连点两次下单，怎么保证只扣一次钱？"——这是微服务/支付场景的**必考题**。面试官想考察：

1. 是否理解幂等的本质（**同一个请求执行多次 = 执行一次**）
2. 是否掌握至少 2~3 种落地手段（唯一约束、去重表、Token、状态机）
3. 是否能说清幂等和并发控制的区别与配合
4. 是否有真实场景经验（支付、下单、MQ 消费）

这道题是区分"会写接口"和"会设计系统"的分水岭。

---

## 核心答案（30 秒版）

**幂等 = 一次和多次请求的结果一致，且不产生副作用。**

四大落地手段：

| 手段 | 原理 | 适用场景 |
|------|------|---------|
| **数据库唯一约束** | 唯一键冲突 → 第二次直接失败/返回已存在 | 订单号、支付流水号 |
| **去重表** | 请求进来先查/插去重记录（唯一ID） | 通用接口幂等 |
| **Token 机制** | 前端先取 token，提交时带 token 并消费 | 表单重复提交 |
| **状态机校验** | 状态流转只允许单向，重复请求状态不匹配则拒绝 | 订单状态变更 |

**核心口诀：要么天然幂等（查询/删除），要么用唯一标识 + 存储去重（写入类操作），要么用状态机限制流转（状态变更类操作）。**

---

## 深度展开

### 1. 先分类：哪些操作天然幂等，哪些需要设计？

| 操作 | 天然幂等？ | 说明 |
|------|-----------|------|
| GET 查询 | ✅ | 多次查询结果一致 |
| DELETE | ✅（通常） | 删除不存在的数据也是成功 |
| PUT（全量更新） | ✅ | 同值多次更新结果一致 |
| POST 创建 | ❌ | 多次提交创建多条记录 |
| 扣款/扣库存 | ❌ | 多次执行重复扣减 |
| 状态流转（待支付→已支付） | ❌ | 需状态机约束 |

### 2. 方案一：数据库唯一约束（最可靠，推荐首选）

利用数据库**唯一索引**天然去重，业务上最省心：

```sql
-- 订单表：订单号唯一
CREATE TABLE `order` (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_no VARCHAR(64) NOT NULL COMMENT '业务订单号（唯一）',
  user_id BIGINT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  status TINYINT NOT NULL DEFAULT 0,
  UNIQUE KEY uk_order_no (order_no)
) ENGINE=InnoDB;
```

```go
// 创建订单：靠唯一索引兜底，重复提交直接报 DuplicateKey
func (s *Service) CreateOrder(ctx context.Context, req CreateOrderReq) (*Order, error) {
    order := &Order{
        OrderNo: req.OrderNo, // 业务方生成（或服务端生成）
        UserID:  req.UserID,
        Amount:  req.Amount,
    }
    if err := s.db.Create(order).Error; err != nil {
        // 唯一键冲突 → 说明订单已存在 → 返回已存在的订单（幂等成功）
        if isDuplicateKey(err) {
            var exist Order
            s.db.Where("order_no = ?", req.OrderNo).First(&exist)
            return &exist, nil
        }
        return nil, err
    }
    return order, nil
}
```

**要点**：
- 唯一键 = 业务幂等键（订单号、支付流水号、消息ID）
- 冲突时**返回已存在的数据**而不是报错（对客户端是幂等成功）
- 注意唯一键冲突的异常捕获（MySQL 1062 / `Duplicate entry`）

### 3. 方案二：去重表（通用幂等中间层）

不依赖业务表结构，用独立的去重表记录请求：

```sql
CREATE TABLE idempotent_record (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  biz_type VARCHAR(32) NOT NULL COMMENT '业务类型',
  req_id   VARCHAR(64) NOT NULL COMMENT '唯一请求ID',
  resp     JSON COMMENT '首次处理的响应结果（缓存）',
  created_at DATETIME NOT NULL,
  UNIQUE KEY uk_biz_req (biz_type, req_id)
) ENGINE=InnoDB;
```

```go
// 通用幂等模板：先占位，后处理
func Idempotent(ctx context.Context, bizType, reqID string, fn func() (any, error)) (any, error) {
    // 1. 先插入占位记录（唯一约束保证并发下只有一个成功）
    rec := &IdempotentRecord{BizType: bizType, ReqID: reqID}
    if err := db.Create(rec).Error; err != nil {
        // 2. 已存在 → 查历史结果直接返回（幂等命中）
        if isDuplicateKey(err) {
            var old IdempotentRecord
            db.Where("biz_type = ? AND req_id = ?", bizType, reqID).First(&old)
            if old.Resp != nil {
                return old.Resp, nil
            }
            return nil, ErrProcessing // 还在处理中
        }
        return nil, err
    }
    // 3. 首次处理，结果回填
    resp, err := fn()
    db.Model(rec).Update("resp", resp)
    return resp, err
}
```

**去重表注意点**：
- 占位记录要设**过期时间**（如 24h 清理），避免无限膨胀
- 处理中（Processing）与完成（Done）要区分，处理中返回"请稍后重试"
- 去重表与业务表**同一数据库**（同事务）才可靠，跨库有分布式问题

### 4. 方案三：Token 机制（防表单重复提交）

前端提交前先向后端申请 token，提交时携带并**一次性消费**：

```go
// 1. 申请 token：存入 Redis，5 分钟有效
func (s *Service) GetToken(ctx context.Context, uid int64) (string, error) {
    token := uuid.NewString()
    key := fmt.Sprintf("idem:token:%s", token)
    if err := s.redis.Set(ctx, key, uid, 5*time.Minute).Err(); err != nil {
        return "", err
    }
    return token, nil
}

// 2. 提交时消费 token：Redis Lua 保证原子（存在才删除）
const consumeTokenScript = `
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end
return 0
`

func (s *Service) SubmitOrder(ctx context.Context, req SubmitReq) error {
    ok, err := s.redis.Eval(ctx, consumeTokenScript,
        []string{fmt.Sprintf("idem:token:%s", req.Token)}, req.UserID).Int()
    if err != nil {
        return err
    }
    if ok == 0 {
        return ErrDuplicateSubmit // token 不存在/已消费 → 重复提交
    }
    // ... 正常下单逻辑
    return nil
}
```

**适用场景**：表单提交、按钮双击、抢购下单。Redis 单点故障时可用 DB 唯一表替代。

### 5. 方案四：状态机校验（状态流转幂等）

状态变更类接口（支付回调、订单取消、退款审核）用**状态机**约束流转：

```go
// 订单状态机：只允许单向流转
var orderStateMachine = map[OrderStatus][]OrderStatus{
    StatusCreated:  {StatusPaid, StatusCancelled},        // 已创建 → 已支付/已取消
    StatusPaid:     {StatusShipped, StatusRefunding},     // 已支付 → 已发货/退款中
    StatusShipped:  {StatusCompleted, StatusRefunding},   // 已发货 → 已完成/退款中
    StatusRefunding: {StatusRefunded},                    // 退款中 → 已退款
}

// 支付回调：只有"已创建"才允许变"已支付"
func (s *Service) HandlePayCallback(ctx context.Context, orderNo string) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        var order Order
        if err := tx.Clauses(clause.Locking{Strength: "UPDATE"}). // 行锁防并发
            Where("order_no = ?", orderNo).First(&order).Error; err != nil {
            return err
        }
        // 状态机校验：重复回调时状态已是 PAID，不在 CREATED 的允许列表里 → 直接拒绝
        if !canTransition(order.Status, StatusPaid) {
            log.Warn("duplicate callback ignored", "order", orderNo, "status", order.Status)
            return nil // 幂等：已处理过，返回成功不报错
        }
        return tx.Model(&order).Update("status", StatusPaid).Error
    })
}
```

**要点**：状态流转图要**单向无环**；重复请求命中"非法流转"时**返回成功但不重复执行**（对支付渠道回调尤其重要——重试是常态）。

### 6. 幂等 + 并发控制：两者如何配合？

| 手段 | 解决什么问题 | 举例 |
|------|-------------|------|
| 幂等 | **重复请求**（网络重试、用户双击、MQ 重投） | 同一订单号提交两次 |
| 并发控制 | **同时请求**（两个请求同时到达，都通过校验） | 两个并发扣款都读到余额够 |

唯一约束/去重表**天然同时解决两者**（DB 层面串行化）；状态机方案需要**行锁/乐观锁**配合：

```go
// 乐观锁：version 字段防并发覆盖
UPDATE order SET amount = ?, version = version + 1
WHERE order_no = ? AND version = ?
// 影响行数 = 0 → 版本冲突，重试或失败
```

---

## 高频追问

### Q1：幂等键（唯一ID）怎么生成？
> 用**业务自然键**（订单号、流水号）最可靠；没有自然键时服务端生成 UUID/雪花 ID，**由客户端传入**（保证网络重试时同一个 ID）——注意：不能让服务端在收到请求后才生成，否则重试会生成新 ID 导致无法去重。

### Q2：MQ 消费幂等和接口幂等有什么区别？
> 本质相同（重复执行无副作用），但 MQ 场景的幂等键通常是**消息 ID（msgID）**或业务主键，且天然有"至少一次投递"语义（重复投递是常态），所以**所有消费者都必须做幂等**，不能指望生产者不重投。详见分布式模块 MQ 篇。

### Q3：幂等和"返回已存在的数据"为什么算成功而不是报错？
> 对调用方来说"这次请求的效果已经达成"，返回已存在数据（或成功）让重试自然收敛；报错会导致上层无限重试或用户困惑。**幂等设计的目标就是让重试无害。**

### Q4：去重表记录什么时候清理？
> 按业务保留期（支付类建议永久/长期，普通接口 24h~7 天）。清理策略：定时任务按 created_at 分批删除，避免大事务。

### Q5：Redis 做幂等（SETNX）和 DB 唯一约束哪个好？
> Redis：性能好，但**有丢失风险**（宕机丢数据、主从切换丢 key），适合可容忍偶发重复的场景（如防表单双击）；DB 唯一约束：可靠、事务一致，适合资金类强一致场景（支付、扣款）。**资金相关首选 DB 方案。**

---

## 延伸阅读

- 本模块：[服务治理：超时、重试、负载均衡](01-rpc/05-03-service-governance.md)（重试机制依赖幂等兜底）
- 分布式模块：[消息队列可靠性（消费幂等）](../../03-distributed/03-mq/19-02-kafka-reliability.md)
- 分布式模块：[本地消息表与最终一致性](../../03-distributed/02-transactions/18-04-msg-eventual.md)
- 系统设计：[支付系统设计](../../05-system-design/README.md)
