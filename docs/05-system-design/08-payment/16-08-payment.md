# 支付系统设计

> 考察频率：★★★★☆  难度：★★★★☆

## 核心问题

支付系统的核心挑战：
1. **资金安全**：不能多扣、不能少扣、不能重复扣
2. **幂等性**：网络超时重试不能导致重复支付
3. **对账准确**：商户、用户、支付通道三方一致性
4. **高可用**：支付是生命线，不能垮

---

## 1. 支付流程

### 典型支付链路

```
用户下单 → 创建订单(待支付) → 唤起支付 → 第三方支付回调 → 更新订单状态
                                          ↓
                                    支付失败 → 订单超时取消
```

### Go 实现订单创建

```go
type Order struct {
    ID          string    `json:"id"`
    UserID      string    `json:"user_id"`
    Amount      int64     `json:"amount"`       // 分
    Status      string    `json:"status"`       // pending/paid/cancelled/timeout
    TradeNO     string    `json:"trade_no"`     // 第三方支付流水号
    CreatedAt   time.Time `json:"created_at"`
    PaidAt      *time.Time `json:"paid_at"`
}

// 创建订单 - 幂等key防止重复下单
func CreateOrder(ctx context.Context, req *CreateOrderReq) (*Order, error) {
    // 1. 幂等检查：用幂等key查询是否已存在
    existing, err := getOrderByIdempotentKey(ctx, req.IdempotentKey)
    if err == nil && existing != nil {
        return existing, nil  // 已存在，直接返回
    }

    // 2. 创建订单
    order := &Order{
        ID:            generateOrderID(),
        UserID:        req.UserID,
        Amount:        req.Amount,
        Status:        "pending",
        IdempotentKey: req.IdempotentKey,
        CreatedAt:     time.Now(),
    }

    // 3. 开启事务：创建订单 + 锁定库存
    err = db.Transaction(func(tx *gorm.DB) error {
        if err := tx.Create(order).Error; err != nil {
            return err
        }
        // 库存扣减（如果实物商品）
        if err := tx.Model(&Product{}).Where("id = ?", req.ProductID).
            Update("stock", gorm.Expr("stock - ?", req.Quantity)).Error; err != nil {
            return err
        }
        return nil
    })
    return order, err
}
```

### 支付回调处理

```go
// 支付回调 - 绝对幂等
func HandlePayCallback(ctx context.Context, notify *PayNotify) error {
    // 1. 签名校验（防伪造）
    if !verifySign(notify, secretKey) {
        return errors.New("invalid sign")
    }

    // 2. 幂等：更新订单状态
    order, err := getOrderByTradeNO(ctx, notify.TradeNO)
    if err == nil && order.Status == "paid" {
        return nil  // 已支付，直接返回（幂等）
    }

    // 3. 开启事务：更新状态 + 记录流水
    return db.Transaction(func(tx *gorm.DB) error {
        // 更新订单
        if err := tx.Model(&Order{}).Where("trade_no = ?", notify.TradeNO).
            Updates(map[string]interface{}{
                "status":  "paid",
                "paid_at": time.Now(),
            }).Error; err != nil {
            return err
        }
        // 记录支付流水
        return tx.Create(&PaymentLog{
            TradeNO:   notify.TradeNO,
            Amount:    notify.Amount,
            Status:    notify.Status,
            PaidAt:    time.Now(),
        }).Error
    })
}
```

---

## 2. 幂等设计

### 为什么幂等是支付的命

支付天然需要重试（网络超时、用户反复点击）。如果没幂等：
- 用户点击了 3 次 → 扣了 3 份钱 → 客诉爆炸
- 回调超时重发 → 重复加款 → 资金损失

### 幂等实现方案

**方案一：唯一索引 + INSERT ON DUPLICATE KEY**

```go
// 支付流水表用唯一索引约束
type PaymentFlow struct {
    ID            uint   `gorm:"primaryKey"`
    IdempotentKey string `gorm:"uniqueIndex"`  // 业务方传入的幂等key
    TradeNO       string `gorm:"index"`        // 第三方流水号
    Amount        int64
    Status        string
}

// 创建支付记录 - 数据库唯一约束保证幂等
func CreatePayment(ctx context.Context, flow *PaymentFlow) error {
    err := db.Create(flow).Error
    if mysql.IsDuplicate(err) {
        return nil  // 已存在，幂等返回
    }
    return err
}
```

**方案二：状态机 + 乐观锁**

```go
func UpdateOrderStatus(tx *gorm.DB, orderID string, fromStatus, toStatus string) error {
    result := tx.Model(&Order{}).
        Where("id = ? AND status = ?", orderID, fromStatus).
        Update("status", toStatus)
    if result.RowsAffected == 0 {
        return errors.New("状态已变更或订单不存在")
    }
    return nil
}

// 支付时严格按状态机流转
// pending → paying → paid
// pending → cancelled (用户取消)
// pending → timeout (超时)
```

---

## 3. 资金安全

### 防重复扣款

```go
// 支付时扣款 - 乐观锁 + 余额校验
func DeductBalance(tx *gorm.DB, userID string, amount int64) error {
    // 乐观锁：更新行数=1才成功
    result := tx.Exec(`
        UPDATE users 
        SET balance = balance - ?, version = version + 1 
        WHERE user_id = ? AND balance >= ? AND version = ?`,
        amount, userID, amount, currentVersion,
    )
    if result.RowsAffected == 0 {
        return errors.New("余额不足或并发更新")
    }
    return nil
}
```

### 防止负余额

```sql
-- 建表时加 CHECK 约束
ALTER TABLE users ADD CONSTRAINT chk_balance CHECK (balance >= 0);
```

---

## 4. 对账系统

### 日对账流程

每天凌晨跑对账任务：
1. **下载对账单**：从微信/支付宝下载前一天的交易流水
2. **本地交易查询**：查本地支付流水表
3. **双向比对**：
   - 本地有、第三方无 → 掉单
   - 第三方有、本地无 → 补单
   - 金额不一致 → 差异流水

```go
// 对账任务
func Reconcile(ctx context.Context, date string) (*ReconcileResult, error) {
    // 1. 下载微信对账单
    wxRecords, err := downloadWxBill(date)
    if err != nil {
        return nil, err
    }

    // 2. 查询本地记录
    localRecords, err := getLocalPayments(date)
    if err != nil {
        return nil, err
    }

    // 3. 构键值 map
    wxMap := make(map[string]*WxRecord)
    for _, r := range wxRecords {
        wxMap[r.TransactionID] = r
    }

    // 4. 双向比对
    result := &ReconcileResult{}
    for _, local := range localRecords {
        wx, ok := wxMap[local.TradeNO]
        if !ok {
            result.MissingInWx = append(result.MissingInWx, local)  // 本地有，微信无
            continue
        }
        if wx.Amount != local.Amount {
            result.AmountMismatch = append(result.AmountMismatch, local)  // 金额不一致
        }
        delete(wxMap, local.TradeNO)
    }
    // wxMap 剩余的就是微信有、本地无
    result.MissingInLocal = maps.Values(wxMap)

    return result, nil
}
```

### 差异处理

| 差异类型 | 处理方式 |
|---------|---------|
| 本地有、第三方无 | 补单（本地落库，发通知） |
| 第三方有、本地无 | 查证后补单或退款 |
| 金额不一致 | 进入人工处理流程 |
| 重复扣款 | 原路退回 |

---

## 5. 超时与关单

### 订单超时自动关单

```go
// 启动定时关单任务
func startOrderTimeoutChecker() {
    ticker := time.NewTicker(30 * time.Second)
    for range ticker.C {
        // 查找超过15分钟未支付的订单
        orders, _ := getTimedOutOrders(15 * time.Minute)
        for _, order := range orders {
            cancelOrder(order)  // 释放库存
        }
    }
}

func cancelOrder(order *Order) error {
    return db.Transaction(func(tx *gorm.DB) error {
        // 1. 更新订单状态
        if err := tx.Model(order).Where("status = ?", "pending").
            Update("status", "cancelled").Error; err != nil {
            return err
        }
        // 2. 释放库存
        if err := tx.Model(&Product{}).Where("id = ?", order.ProductID).
            Update("stock", gorm.Expr("stock + ?", order.Quantity)).Error; err != nil {
            return err
        }
        return nil
    })
}
```

---

## 6. 面试话术

**Q：支付系统如何保证不丢消息？**

> 回调消息处理要"at least once"：收到回调后先持久化（落库），再返回成功。配合定时任务对账兜底。另外主动查询（轮询）作为回调的补偿机制。

**Q：如何防止用户重复点击？**

> 三层防护：1）前端按钮点击后置灰；2）后端用幂等 key 挡重试；3）支付渠道侧用同一个 out_trade_no 覆盖（微信支付支持）。

**Q：如何做幂等？**

> 核心是用业务幂等 key（如 order_id + 支付渠道）做唯一索引，数据库层保证只插入一次。配合状态机，任何状态变更操作都要校验当前状态，防止跨状态覆盖。

**Q：支付系统怎么设计高可用？**

> 1）多通道：微信挂了切支付宝；2）降级：非核心功能（分享券）在高峰期降级；3）熔断：第三方支付超时自动熔断，防止拖垮整个系统；4）对账作为最终一致性兜底。

---

## 总结

| 核心问题 | 解决方案 |
|---------|---------|
| 重复支付 | 幂等key + 唯一索引 |
| 余额超扣 | 乐观锁 + CHECK约束 |
| 回调丢失 | 持久化 + 定时对账 |
| 对账差异 | 双向比对 + 人工处理 |
| 超时关单 | 定时任务 + 状态机 |
