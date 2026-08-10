# Saga 分布式事务模式

> 考察频率：★★★☆☆  难度：★★★★☆
> 关键词：编排/协调、长事务、补偿事务、正向恢复

## Saga 是什么

Saga 是一种**长事务**（Long-Running Transaction）的解决方案，核心思想是：

> **将一个分布式事务拆分为多个本地事务，每个本地事务有对应的补偿操作。当某个步骤失败时，逆向执行前面所有步骤的补偿操作（C），以恢复到初始状态。**

与 TCC 的核心区别：
- **TCC** 有 Try 预留阶段，Saga **没有预留**，直接执行
- **TCC** 通过资源预留实现隔离性，Saga **不保证隔离性**（正反向执行是紧耦合的）
- **Saga** 天然适合**长流程、异步链路**（如订单 -> 支付 -> 发货 -> 物流）

---

## 两种模式：编排 vs 协调

### 编排模式（Choreography）

每个参与者自己决定下一步做什么，通过事件驱动：

```
  订单服务                库存服务              支付服务
     |                     |                    |
     |---创建订单---------->|                    |
     |                     |---扣减库存--------->|
     |                     |                    |---扣款-----+
     |                     |                    |            |
     |<-----订单失败--------|<----库存不足-------|----支付失败+
     |                     |
     |---补偿：恢复库存---->|
     |                     |
```

**特点**：去中心化，适合步骤较少的简单流程。缺点是参与者之间耦合度高，流程复杂时难以追踪。

### 协调模式（Orchestration）

引入一个**协调者（Orchestrator）** 集中控制整个流程：

```go
type OrderOrchestrator struct {
    orderSvc   *OrderService
    stockSvc   *StockService
    paymentSvc *PaymentService
}

func (o *OrderOrchestrator) CreateOrder(ctx context.Context, req *CreateOrderReq) error {
    // 1. 创建订单（状态：处理中）
    order, err := o.orderSvc.Create(ctx, req)
    if err != nil {
        return err
    }

    // 2. 扣减库存
    if err := o.stockSvc.Deduct(ctx, order.ID, req.Items); err != nil {
        // 补偿：取消订单
        o.orderSvc.Cancel(ctx, order.ID)
        return err
    }

    // 3. 扣款
    if err := o.paymentSvc.Charge(ctx, order.ID, req.Amount); err != nil {
        // 补偿：恢复库存
        o.stockSvc.Restore(ctx, order.ID)
        // 补偿：取消订单
        o.orderSvc.Cancel(ctx, order.ID)
        return err
    }

    // 4. 确认订单
    return o.orderSvc.Confirm(ctx, order.ID)
}
```

**特点**：集中管理，适合步骤多、流程复杂的场景。

---

## Go 实现：Saga 编排器

### 1. 步骤定义

```go
type SagaStep struct {
    Name         string
    Execute      func(ctx context.Context) error  // 正向操作
    Compensate   func(ctx context.Context) error  // 补偿操作
    RetryPolicy  *RetryPolicy                      // 重试策略
}

// Saga 上下文，存储每个步骤的执行结果
type SagaContext struct {
    TXID        string
    Steps       []*SagaStep
    StepResults map[string]any  // stepName -> result
    CompletedSteps []int         // 已完成的步骤索引
    mu          sync.Mutex
}
```

### 2. Saga 执行引擎

```go
type SagaExecutor struct {
    logger      *log.Logger
    sagaStore   SagaStore       // 持久化事务状态
}

func (e *SagaExecutor) Execute(ctx context.Context, steps []*SagaStep) error {
    sagaCtx := &SagaContext{
        TXID:        generateTXID(),
        Steps:       steps,
        StepResults: make(map[string]any),
    }

    // 持久化事务开始状态
    e.sagaStore.Create(sagaCtx)

    completed := 0
    for i, step := range steps {
        err := e.executeStep(ctx, sagaCtx, step)
        if err != nil {
            // 失败，逆向补偿已完成步骤
            e.compensate(ctx, sagaCtx, completed)
            return fmt.Errorf("saga step %s failed: %w", step.Name, err)
        }
        completed = i + 1
    }

    e.sagaStore.MarkCompleted(sagaCtx.TXID)
    return nil
}

func (e *SagaExecutor) executeStep(ctx context.Context, sagaCtx *SagaContext, step *SagaStep) error {
    // 支持重试
    var lastErr error
    for attempt := 0; attempt <= step.RetryPolicy.MaxRetries; attempt++ {
        err := step.Execute(ctx)
        if err == nil {
            sagaCtx.mu.Lock()
            sagaCtx.CompletedSteps = append(sagaCtx.CompletedSteps, len(sagaCtx.Steps))
            sagaCtx.mu.Unlock()
            return nil
        }

        lastErr = err
        if !step.RetryPolicy.ShouldRetry(err) {
            break
        }
        time.Sleep(step.RetryPolicy.Backoff(attempt))
    }
    return lastErr
}

// 逆向补偿（从最后一步往前回退）
func (e *SagaExecutor) compensate(ctx context.Context, sagaCtx *SagaContext, completed int) {
    // 逆序补偿已完成步骤
    for i := completed - 1; i >= 0; i-- {
        step := sagaCtx.Steps[i]
        if err := step.Compensate(ctx); err != nil {
            // 补偿失败，记录日志，定时重试（补偿失败也要保证幂等）
            e.logger.Errorf("compensate step %s failed: %v", step.Name, err)
            e.sagaStore.MarkCompensateFailed(sagaCtx.TXID, step.Name)
        }
    }
}
```

### 3. 订单-支付完整 Saga 示例

```go
func NewOrderPaymentSaga() []*SagaStep {
    return []*SagaStep{
        {
            Name: "create_order",
            Execute: func(ctx context.Context) error {
                orderID := ctx.Value("order_id").(int64)
                return orderService.Create(ctx, orderID)
            },
            Compensate: func(ctx context.Context) error {
                orderID := ctx.Value("order_id").(int64)
                return orderService.Cancel(ctx, orderID) // 补偿：取消订单
            },
        },
        {
            Name: "deduct_inventory",
            Execute: func(ctx context.Context) error {
                return stockService.Deduct(ctx, ctx.Value("items").([]Item))
            },
            Compensate: func(ctx context.Context) error {
                return stockService.Restore(ctx, ctx.Value("items").([]Item)) // 补偿：恢复库存
            },
        },
        {
            Name: "charge_payment",
            Execute: func(ctx context.Context) error {
                return paymentService.Charge(ctx, ctx.Value("amount").(float64))
            },
            Compensate: func(ctx context.Context) error {
                return paymentService.Refund(ctx) // 补偿：退款
            },
        },
    }
}
```

---

## Saga 的核心问题与解决方案

### 1. 隔离性问题（无预留机制）

多个 Saga 并发执行时，可能出现：
- **脏读**：事务 A 的第 2 步扣款成功，事务 B 的第 1 步读取到"已扣款但未确认"的库存
- **乐观锁冲突**：并发补偿时，对同一行数据操作导致更新失败

**解决方案**：
- 使用**语义锁**（在业务字段加锁版本号）
- Saga 事务模式下，业务层自己负责隔离（例如：支付阶段加"订单处理中"状态锁）
- 串联事务间设置合理的**事务超时**

```go
// 在库存表加锁版本号实现乐观锁
type Inventory struct {
    Stock   int   `json:"stock"`
    LockVer int64 `json:"lock_ver"` // 业务锁版本
}

// 扣库存时加锁（SELECT FOR UPDATE 业务锁）
func DeductStock(orderID int64) error {
    tx, _ := db.Begin()
    // 业务锁：只允许同一订单处理中的事务操作
    _, err := tx.Exec(`
        UPDATE inventory
        SET stock = stock - ?, lock_ver = lock_ver + 1
        WHERE sku_id = ? AND lock_ver = ?
    `, qty, skuID, expectedVer)
    if err != nil {
        tx.Rollback()
        return err
    }
    return tx.Commit()
}
```

### 2. 补偿失败问题

补偿操作也可能失败，导致事务无法完全回滚。

**解决方案**：
- 补偿操作本身需要幂等
- 引入**定时任务**扫描未完成补偿的 Saga，执行重试
- 关键业务可配合人工审核

```go
// 定时扫描未完成 Saga，执行补偿
func (e *SagaExecutor) StartCompensationWorker(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Minute)
    for {
        select {
        case <-ticker.C:
            failed := e.sagaStore.FindCompensating()
            for _, saga := range failed {
                for _, step := range saga.UncompletedSteps {
                    if err := step.Compensate(ctx); err != nil {
                        e.logger.Warnf("retry compensate %s: %v", step.Name, err)
                    }
                }
            }
        case <-ctx.Done():
            return
        }
    }
}
```

### 3. 状态机管理

Saga 的每个步骤都应有明确的状态机：

```
订单状态流转（ Saga 与业务状态结合）：
CREATED → INVENTORY_DEDUCTED → PAYMENT_PROCESSING → PAID → SHIPPED → COMPLETED
                |                      |
                v                      v
          CANCELLED              PAYMENT_FAILED →库存恢复→ CANCELLED
```

---

## Saga vs TCC 对比

| 维度 | Saga | TCC |
|------|------|-----|
| 阶段 | 2 阶段（执行 + 补偿） | 3 阶段（预留 + 确认 + 取消） |
| 资源预留 | 无预留，直接执行 | Try 预留资源 |
| 隔离性 | 弱（无全局锁） | 较强（通过资源预留实现） |
| 性能 | 高（无全局锁） | 中（预留资源有一定开销） |
| 实现复杂度 | 低（只需要补偿逻辑） | 高（需要三阶段实现） |
| 适用场景 | 长流程、异步链路 | 需要强隔离的场景 |
| 补偿粒度 | 粗（整个本地事务） | 细（单个资源操作） |

---

## 框架选型

| 框架 | 特点 |
|------|------|
| **Seata Saga** | 阿里开源，支持状态机编排，可视化流程设计 |
| **Eventuate Tram** | 基于事件的 Saga 框架，支持 Choreography |
| **Conductor** | Netflix 开源，支持复杂的 DAG 流程编排 |

---

## 面试高频追问

**Q1: Saga 为什么没有隔离性？怎么解决？**
→ 因为 Saga 没有 Try 预留阶段，直接执行，所有步骤都是"正向+补偿"紧耦合的。解决方式：业务层加语义锁（如订单状态）；Saga 之间错开执行窗口；配合定时对账。

**Q2: Saga 的补偿失败率很高吗？**
→ 补偿失败通常是网络抖动或数据库暂时不可用。采用幂等补偿 + 定时重试 + 告警机制，实际生产中补偿成功率很高。关键是对补偿失败做监控告警。

**Q3: Saga 和 TCC 怎么选型？**
→ TCC 适合资源需要预留的场景（库存锁定、账户冻结），隔离性强。Saga 适合长链路、异步流程（订单 -> 物流 -> 签收），性能更好。

**Q4: Saga 能做到强一致吗？**
→ Saga 是最终一致性模型。正向执行完就认为成功，补偿是对失败的兜底。如果要求强一致，应选择 2PC/TCC。
