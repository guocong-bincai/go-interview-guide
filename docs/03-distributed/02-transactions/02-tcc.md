# TCC 分布式事务模式

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：Try/Confirm/Cancel、幂等设计、空回滚、悬挂问题

## TCC 是什么

TCC（Try-Confirm-Cancel）是分布式事务的一种补偿式解决方案，**将整个事务拆分為三阶段**：

| 阶段 | 作用 | 特点 |
|------|------|------|
| **Try** | 预留资源，锁定检查 | 所有参与者完成预留，类似于乐观锁 |
| **Confirm** | 确认执行，使用预留资源 | Try 成功后自动执行，资源真正被占用 |
| **Cancel** | 取消释放，释放预留资源 | Try 失败或超时后触发，回滚到初始状态 |

与 2PC 对比：TCC 是业务层面的协议，由业务代码控制 commit/rollback，因此可以自己决定资源粒度和补偿逻辑。

---

## 核心流程图

```
                        TCC 事务流程
                        ============

   应用层
     |
     v
 +--------+
 |  Try   |  ← 预留资源阶段，所有参与者尝试预留
 +----+---+
      |
      v
 +--------+
 | ALL OK?|----No----→ +--------+
 |        |             | Cancel |  ← 有参与者失败，全部取消
 +--------+             +--------+
   | Yes
   v
 +---------+
 | Confirm |  ← 确认执行，预留资源正式转为实际占用
 +---------+
```

---

## Go 实现框架

### 1. TCC 接口定义

```go
// TccAction 是单个 TCC 参与者的接口
type TccAction interface {
    Try(ctx context.Context, res *TccResource) (bool, error)   // 预留资源
    Confirm(ctx context.Context, res *TccResource) error       // 确认执行
    Cancel(ctx context context.Context, res *TccResource) error // 取消释放
}

// TccResource 是事务资源上下文，在三阶段间传递
type TccResource struct {
    TxID        string                 // 全局事务 ID
    BranchID    string                 // 分支事务 ID
    ActionName  string                 // 操作名称
    TryData     map[string]interface{} // Try 阶段产生的预留数据
}
```

### 2. TCC 服务（协调者）

```go
type TccTransactionManager struct {
    participants map[string]TccAction
    txStore      TccStore             // 持久化事务状态（Redis/DB）
}

func (m *TccTransactionManager) Execute(ctx context.Context, actions map[string]TccAction, business func() error) error {
    txID := generateTXID()

    // 1. 调用所有参与者的 Try
    prepared := make([]string, 0)
    for name, action := range actions {
        res := &TccResource{
            TxID:       txID,
            BranchID:   generateBranchID(),
            ActionName: name,
        }
        ok, err := action.Try(ctx, res)
        if err != nil || !ok {
            // Try 失败，回滚已成功的 Try
            m.cancel(prepared)
            return err
        }
        prepared = append(prepared, name)
        // 将 res.TryData 持久化，用于 Confirm/Cancel
        m.txStore.Save(txID, name, res)
    }

    // 2. Try 全部成功执行业务
    if err := business(); err != nil {
        // 业务失败，调用 Cancel
        m.cancel(prepared)
        return err
    }

    // 3. 调用所有参与者的 Confirm
    return m.confirmAll(prepared)
}

func (m *TccTransactionManager) confirmAll(prepared []string) error {
    for _, name := range prepared {
        res, _ := m.txStore.Load(txID, name)
        if err := m.participants[name].Confirm(ctx, res); err != nil {
            // Confirm 失败需要重试，直到成功（幂等保证）
            m.retryConfirm(name, res)
        }
    }
    return nil
}

func (m *TccTransactionManager) cancel(prepared []string) {
    for _, name := range prepared {
        res, _ := m.txStore.Load(txID, name)
        m.participants[name].Cancel(ctx, res)
    }
}
```

---

## 实际业务场景：账户转账

### 账户表设计

```go
// account 数据库表结构
type Account struct {
    ID        int64
    Balance   decimal.Decimal // 当前余额
    FrozenAmt decimal.Decimal // 冻结金额（预留未确认）
    Version   int64           // 乐观锁版本
}
```

### Try 阶段：预留资金

```go
func (a *AccountAction) Try(ctx context.Context, res *TccResource) (bool, error) {
    req := ctx.Value("deduct_req").(*DeductRequest)

    // 冻结金额 = 实际扣款金额（也可以放大预留，如预留 2 倍）
    frozenAmt := req.Amount

    rows, err := db.ExecContext(ctx, `
        UPDATE account
        SET frozen_amt = frozen_amt + ?
        WHERE id = ? AND balance - frozen_amt >= ?
    `, frozenAmt, req.AccountID, frozenAmt)

    if err != nil {
        return false, err
    }

    affected, _ := rows.RowsAffected()
    if affected == 0 {
        // 余额不足，无法预留
        return false, nil
    }

    // 记录预留数据（Confirm/Cancel 时用）
    res.TryData = map[string]interface{}{
        "account_id": req.AccountID,
        "frozen_amt": frozenAmt,
    }
    return true, nil
}
```

### Confirm 阶段：真正扣款

```go
func (a *AccountAction) Confirm(ctx context.Context, res *TccResource) error {
    accountID := res.TryData["account_id"].(int64)
    frozenAmt := res.TryData["frozen_amt"].(decimal.Decimal)

    // 扣减余额，同时释放冻结金额（一并完成）
    _, err := db.ExecContext(ctx, `
        UPDATE account
        SET balance = balance - ?,
            frozen_amt = frozen_amt - ?,
            version = version + 1
        WHERE id = ? AND version = ?
    `, frozenAmt, frozenAmt, accountID, res.TryData["version"])

    return err
}
```

### Cancel 阶段：释放冻结

```go
func (a *AccountAction) Cancel(ctx context.Context, res *TccResource) error {
    accountID := res.TryData["account_id"].(int64)
    frozenAmt := res.TryData["frozen_amt"].(decimal.Decimal)

    // 释放冻结金额（回到余额）
    _, err := db.ExecContext(ctx, `
        UPDATE account
        SET frozen_amt = frozen_amt - ?
        WHERE id = ?
    `, frozenAmt, accountID)

    return err
}
```

---

## TCC 四大难题

### 1. 幂等设计

Confirm/Cancel 可能重复执行，必须幂等。

```go
// 使用分支事务ID（BranchID）做幂等控制
func (a *AccountAction) Confirm(ctx context.Context, res *TccResource) error {
    // 先查询是否已处理
    var count int
    db.QueryRowContext(ctx,
        "SELECT COUNT(*) FROM tcc_log WHERE branch_id = ? AND status = 'CONFIRMED'",
        res.BranchID,
    ).Scan(&count)
    if count > 0 {
        return nil // 幂等，已处理
    }

    // 执行确认逻辑...

    // 记录日志
    db.ExecContext(ctx,
        "INSERT INTO tcc_log (branch_id, status) VALUES (?, 'CONFIRMED')",
        res.BranchID,
    )
    return nil
}
```

### 2. 空回滚

Try 未执行，Cancel 就被调用（如网络问题导致 Try 超时）。

```go
func (a *AccountAction) Cancel(ctx context.Context, res *TccResource) error {
    // 查询 Try 是否从未执行过
    var count int
    db.QueryRowContext(ctx,
        "SELECT COUNT(*) FROM tcc_log WHERE branch_id = ? AND status = 'TRIED'",
        res.BranchID,
    ).Scan(&count)
    if count == 0 {
        // 空回滚：Try 没执行过，记录日志但不做实际回滚
        log.Warnf("空回滚: branchID=%s", res.BranchID)
        return nil
    }
    // 正常回滚...
}
```

### 3. 悬挂问题

Cancel 比 Try 先执行（网络乱序导致）。

```go
func (a *AccountAction) Try(ctx context.Context, res *TccResource) (bool, error) {
    // 检查是否已有 Cancel 记录
    var cancelStatus string
    err := db.QueryRowContext(ctx,
        "SELECT status FROM tcc_log WHERE branch_id = ? AND status = 'CANCELLED'",
        res.BranchID,
    ).Scan(&cancelStatus)

    if err == nil {
        // 已有 Cancel，Try 不再执行（悬挂）
        log.Warnf("悬挂: branchID=%s, 跳过 Try", res.BranchID)
        return false, nil
    }

    // 正常 Try 逻辑...
    return true, nil
}
```

### 4. 事务状态持久化

TCC 事务状态需要可靠持久化，否则协调者重启后无法恢复。

```go
// 事务状态表
type TccTransaction struct {
    TxID        string `json:"tx_id"`
    Status      string `json:"status"` // TRYING/CONFIRMING/CANCELLING
    CreatedAt   int64  `json:"created_at"`
    UpdatedAt   int64  `json:"updated_at"`
    TryTimeout  int    `json:"try_timeout"`  // 秒
}

// 定期扫描超时未 Confirm/Cancel 的事务，重试
func (m *TccTransactionManager) retryTimeoutTransactions() {
    timeoutTxIDs := m.txStore.FindTimeout(tryTimeout)
    for _, txID := range timeoutTxIDs {
        tx, _ := m.txStore.Load(txID)
        if tx.Status == "TRYING" {
            // Try 超时，触发 Cancel
            m.cancel(tx.BranchIDs)
        } else if tx.Status == "CONFIRMING" {
            // Confirm 超时，重试 Confirm
            m.confirmAll(tx.BranchIDs)
        }
    }
}
```

---

## 与 2PC 对比

| 维度 | 2PC | TCC |
|------|-----|-----|
| 协议层面 | 数据库层面的协议 | 业务层面的协议 |
| 侵入性 | 低（数据库自动处理） | 高（业务代码实现 Try/Confirm/Cancel） |
| 性能 | 较差（全局锁） | 好（预留+本地锁） |
| 灵活性 | 低 | 高（自定义资源粒度） |
| 适用场景 | 强一致、强关联的系统 | 灵活的业务场景、跨服务调用 |
| 数据可靠性 | 依赖协调者是否正确 | 依赖业务补偿逻辑正确性 |

---

## 框架选型：Seata AT/TCC

| 框架 | 模式 | 特点 |
|------|------|------|
| **Seata AT** | 自动补偿 | 无侵入，使用undo_log回滚 |
| **Seata TCC** | 手动补偿 | 高性能，需业务实现三阶段 |
| **Hmily** | TCC | 轻量，专门做 TCC |
| **ByteTCC** | TCC | 基于 Spring，支持嵌套事务 |

---

## 面试高频追问

**Q1: Try 阶段失败了怎么办？**
→ 协调者向所有已成功 Try 的参与者发送 Cancel，触发空回滚或正常回滚。

**Q2: Confirm/Cancel 失败了怎么办？**
→ 需要定时任务重试（依赖幂等性保证），或者人工介入。Seata 框架会自动重试直到成功。

**Q3: Try 预留的资源和 Confirm 实际使用的资源如何保持一致？**
→ 建议 Try 预留的量和 Confirm 使用的量完全一致，避免不一致。对于金融场景，可以预留稍多一些作为风控。

**Q4: TCC 的性能瓶颈在哪里？**
→ 多次网络开销（Try + Confirm/Cancel），建议 Try 阶段做轻量检查，实际资源操作在 Confirm 中完成。
