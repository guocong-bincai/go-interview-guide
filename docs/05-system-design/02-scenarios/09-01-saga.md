# Saga 模式在业务中的落地

> 考察频率：★★★☆☆  难度：★★★★☆
> 重点：能区分编排型/协调型 Saga，知道补偿事务的设计要点

## Saga 是什么

Saga 是一种**最终一致**的分布式事务模式，将长事务拆分为多个**本地事务 + 补偿操作**。

### 对比 2PC/TCC

| 维度 | 2PC | TCC | Saga |
|------|-----|-----|------|
| 隔离性 | 强 | 强 | 最终一致 |
| 侵入性 | 中 | 高 | 低 |
| 性能 | 低 | 中 | 高 |
| 回滚 | 自动 | 自动（Try/Cancel） | 手动补偿 |
| 适用场景 | 短事务 | 跨服务强一致 | 长链路最终一致 |

---

## 编排型 vs 协调型 Saga

### 编排型（Choreography）

各服务通过事件自协调，无中心编排器。

```
OrderService:
    on order.create → emit order.created
        ↓
InventoryService:
    on order.created → reserve → emit inventory.reserved OR inventory.failed
        ↓
PaymentService:
    on inventory.reserved → charge → emit payment.completed OR payment.failed
        ↓
ShippingService:
    on payment.completed → ship → emit shipping.delivered
```

**缺点：** 事件多时难以追踪；缺乏统一视图；循环依赖风险。

### 协调型（Orchestration）

单一 Saga Orchestrator 统一控制全局流程。

```go
type OrderSagaOrchestrator struct {
    orderSvc     *OrderService
    inventorySvc *InventoryService
    paymentSvc   *PaymentService
    shippingSvc  *ShippingService
}

func (s *OrderSagaOrchestrator) Execute(ctx context.Context, order *Order) error {
    // Step 1: 创建订单
    if err := s.orderSvc.Create(ctx, order); err != nil {
        return err
    }

    // Step 2: 预留库存
    if err := s.inventorySvc.Reserve(ctx, order.ID, order.Items); err != nil {
        // 补偿 Step 1
        s.orderSvc.Cancel(ctx, order.ID)
        return err
    }

    // Step 3: 扣款
    if err := s.paymentSvc.Charge(ctx, order.UserID, order.Amount); err != nil {
        // 补偿 Step 2
        s.inventorySvc.Release(ctx, order.ID, order.Items)
        // 补偿 Step 1
        s.orderSvc.Cancel(ctx, order.ID)
        return err
    }

    // Step 4: 发货
    if err := s.shippingSvc.Ship(ctx, order.ID); err != nil {
        // 补偿 Step 3
        s.paymentSvc.Refund(ctx, order.UserID, order.Amount)
        // 补偿 Step 2
        s.inventorySvc.Release(ctx, order.ID, order.Items)
        // 补偿 Step 1
        s.orderSvc.Cancel(ctx, order.ID)
        return err
    }

    return nil
}
```

---

## Saga 的补偿设计

### 补偿 ≠ 回滚

**回滚：** 撤销操作，恢复原状（如数据库事务回滚）  
**补偿：** 执行反向操作，修正状态（如退款、取消订单）

### 补偿事务设计原则

```go
// ✅ 好的补偿：幂等 + 查状态
func (s *InventoryService) ReleaseCompensate(ctx context.Context, orderID string) error {
    // 1. 检查是否已释放（幂等）
    released, _ := s.checkAlreadyReleased(ctx, orderID)
    if released {
        return nil
    }
    
    // 2. 执行释放
    _, err := s.db.ExecContext(ctx, 
        "UPDATE inventory SET reserved = reserved - ? WHERE order_id = ?", qty, orderID)
    return err
}

// ❌ 差的补偿：直接操作，忽略当前状态
func BadRelease(ctx context.Context, orderID string) error {
    // 假设之前扣了10件，直接加回10件
    // 问题：如果之前已经释放过，就会多加
    db.Exec("UPDATE inventory SET reserved = reserved + 10 ...")
}
```

### 补偿的幂等设计

```go
type CompensationRecord struct {
    SagaID     string
    StepID     int
    Status     string  // pending / completed / failed
    Attempt    int
    CreatedAt  time.Time
    UpdatedAt  time.Time
}

func (svc *InventoryService) Release(ctx context.Context, orderID string) error {
    // 检查是否已经补偿过
    record, _ := svc.compensationRepo.Find(ctx, orderID, "release_inventory")
    if record != nil && record.Status == "completed" {
        return nil  // 幂等，已完成
    }
    
    // 执行释放
    _, err := svc.db.ExecContext(ctx, "UPDATE inventory SET reserved = reserved - ? ...", ...)
    
    // 记录补偿状态
    svc.compensationRepo.Upsert(ctx, &CompensationRecord{
        SagaID:  getSagaID(ctx),
        StepID:  2,
        Status:  "completed",
    })
    return err
}
```

---

## Saga 状态机

### Saga 执行状态

```
START → order_created → inventory_reserved → payment_charged → completed
                ↓                ↓                    ↓
            cancelled         cancelled           refunded
                ↓                ↓                    ↓
         compensation_1   compensation_2      compensation_3
```

### Go 状态机实现

```go
type SagaState int
const (
    StateStart SagaState = iota
    StateOrderCreated
    StateInventoryReserved
    StatePaymentCharged
    StateCompleted
    StateCancelled
)

type SagaTransition struct {
    From  SagaState
    To    SagaState
    Event string
    Do    func(ctx context.Context) error
    Comp  func(ctx context.Context) error  // 补偿方法
}

func (s *OrderSaga) process(event string) error {
    for _, t := range s.transitions {
        if t.From == s.currentState && t.Event == event {
            if err := t.Do(s.ctx); err != nil {
                return s.compensate()  // 失败则补偿
            }
            s.currentState = t.To
            return nil
        }
    }
    return fmt.Errorf("invalid transition: %s from %s", event, s.currentState)
}
```

---

## Saga 的失败处理

### 补偿重试策略

```go
type RetryPolicy struct {
    MaxAttempts int
    BaseDelay   time.Duration
    MaxDelay    time.Duration
}

func (p *RetryPolicy) NextDelay(attempt int) time.Duration {
    delay := p.BaseDelay * time.Duration(math.Pow(2, float64(attempt)))
    if delay > p.MaxDelay {
        delay = p.MaxDelay
    }
    return delay
}

// 补偿失败时的处理
func (s *Saga) compensate() error {
    for i := len(s.completedSteps) - 1; i >= 0; i-- {
        step := s.completedSteps[i]
        
        for attempt := 0; attempt < s.retryPolicy.MaxAttempts; attempt++ {
            if err := step.Comp(s.ctx); err == nil {
                break  // 成功，继续补偿下一步
            }
            time.Sleep(s.retryPolicy.NextDelay(attempt))  // 重试等待
        }
        
        // 多次重试仍失败
        if attempt == s.retryPolicy.MaxAttempts {
            // 人工介入告警
            s.alertManager.Notify(&Alert{
                Type:       "saga_compensation_failed",
                SagaID:     s.id,
                FailedStep: step.Name,
            })
            return err
        }
    }
    return nil
}
```

---

## Saga 与事件溯源结合

```go
// Saga 的每一步都记录为事件
type SagaEvent struct {
    SagaID    string
    Step      string
    Type      string  // "step_executed" | "compensated"
    Payload   string
    Timestamp time.Time
}

// 存储完整执行历史
func (s *Saga) Record(event *SagaEvent) error {
    return s.eventStore.Save(s.sagaID, event)
}

// 任意时刻可重放/审计
func (s *Saga) Replay(sagaID string) (*SagaState, error) {
    events, _ := s.eventStore.GetAll(sagaID)
    
    state := &SagaState{}
    for _, e := range events {
        state.Apply(e)  // 重放事件
    }
    return state, nil
}
```

---

## 常见问题

### Q：Saga 和 TCC 的核心区别？

> TCC 是"资源预留"模式（Try 冻结，Confirm 扣款，Cancel 解冻），**资源层面强隔离**；Saga 没有预留，直接正向执行，失败后手动补偿，**最终一致但隔离性弱**。简单说：TCC = 预留 + 确认；Saga = 执行 + 补偿。

### Q：Saga 适合什么场景？

> **长链路**（订单→支付→物流→履约）；**外部系统**（不能改对方数据库的 TCC）；**最终一致即可**（不强求实时一致）；**并发低**（TCC 性能更好，Saga 适合低并发长事务）。

### Q：Saga 补偿失败了怎么办？

> 1. **重试**：指数退避重试（一般 3-5 次）；2. **告警**：多次失败触发人工介入；3. **记录**：将失败记录到"待补偿表"，定时任务补偿；4. **最大努力通知**：重试若干次后放弃，发送告警。

### Q：如何保证 Saga 的幂等性？

> 每步操作记录 SagaID + StepID；补偿前先查状态（是否已完成）；用唯一键约束；数据库层加幂等字段（status=completed）。
