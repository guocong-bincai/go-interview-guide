# CQRS 模式

> 考察频率：★★★☆☆  难度：★★★★☆
> 重点：理解读写分离思想、Event Sourcing 结合场景

## 什么是 CQRS

CQRS（Command Query Responsibility Segregation）将**命令（写）**和**查询（读）**分离，使用不同的模型处理读写操作。

### 传统架构 vs CQRS

**传统架构：**
```
同一个数据模型 → 读写一体
Domain Model ← CRUD 操作都走这里
```
**问题：** 读操作需要复杂 JOIN，写优化影响读性能；读写混合导致模型复杂。

**CQRS 架构：**
```
Command（写）              Query（读）
   ↓                          ↓
Command Handler           Read Model/View
   ↓                          ↓
Command DB              Read DB（专用读副本）
（写优化）               （读优化：反范式化、索引）
```

---

## CQRS 核心实现

### 简单 CQRS：读写分离

```go
// ========== Command Side（写）==========
type OrderCommandHandler struct {
    db *sql.DB
}

func (h *OrderCommandHandler) CreateOrder(cmd *CreateOrderCommand) error {
    // 写优化：规范化存储
    tx, _ := h.db.Begin()
    _, err := tx.Exec(
        "INSERT INTO orders (id, user_id, status, created_at) VALUES (?, ?, ?, ?)",
        cmd.OrderID, cmd.UserID, "pending", time.Now())
    for _, item := range cmd.Items {
        _, err = tx.Exec(
            "INSERT INTO order_items (order_id, product_id, qty) VALUES (?, ?, ?)",
            cmd.OrderID, item.ProductID, item.Qty)
    }
    tx.Commit()
    return err
}

// ========== Query Side（读）==========
type OrderQueryHandler struct {
    readDB *sql.DB  // 读副本，专用于复杂查询
}

type OrderView struct {
    OrderID      string
    UserName     string
    ProductNames []string
    TotalAmount  float64
    Status       string
}

func (h *OrderQueryHandler) GetOrderDetail(orderID string) (*OrderView, error) {
    // 读优化：JOIN 查询，提前反范式化
    row := h.readDB.QueryRow(`
        SELECT o.id, u.name, o.status, 
               GROUP_CONCAT(p.name), SUM(p.price * oi.qty)
        FROM orders o
        JOIN users u ON o.user_id = u.id
        JOIN order_items oi ON o.id = oi.order_id
        JOIN products p ON oi.product_id = p.id
        WHERE o.id = ?
        GROUP BY o.id`, orderID)
    
    var view OrderView
    row.Scan(&view.OrderID, &view.UserName, &view.Status, 
             &view.ProductNames, &view.TotalAmount)
    return &view, nil
}
```

---

## 事件驱动的 CQRS

### 通过消息同步读写数据

```
Command DB（写）  →  Domain Event  →  Event Handler  →  Read DB（读）
     ↓                                        ↓
   订单创建                              更新 Read Model
   order.created                        写入 OrderDetailView
```

```go
// Event 定义
type DomainEvent interface {
    EventType() string
    OccurredAt() time.Time
}

type OrderCreated struct {
    OrderID   string
    UserID    string
    Items     []OrderItem
    TotalAmt  float64
}

func (e *OrderCreated) EventType() string { return "order.created" }

// Command Handler 发布事件
func (h *OrderCommandHandler) CreateOrder(cmd *CreateOrderCommand) error {
    order := NewOrder(cmd)
    
    // 保存写模型
    h.db.Exec("INSERT INTO orders ...", order)
    
    // 发布事件（通过 Message Broker）
    h.eventBus.Publish(&OrderCreated{
        OrderID:  order.ID,
        UserID:   order.UserID,
        Items:    order.Items,
        TotalAmt: order.TotalAmount,
    })
    return nil
}

// Event Handler 更新读模型
func (h *OrderEventHandler) OnOrderCreated(e *OrderCreated) error {
    // 写入 Read DB（反范式化视图）
    return h.readDB.Exec(`
        INSERT INTO order_detail_view 
        (order_id, user_name, product_names, total_amount, status)
        VALUES (?, ?, ?, ?, ?)`,
        e.OrderID, e.UserID, e.ProductNames(), e.TotalAmt, "pending")
}
```

---

## Event Sourcing（事件溯源）

### 核心思想

不存储最终状态，存储**所有历史事件**，通过重放事件重建状态。

```go
// 传统方式
type Account struct {
    Balance float64  // 当前余额
}
// 问题：丢失了所有历史操作

// Event Sourcing
type Account struct {
    events []AccountEvent  // 存储所有事件
}

type AccountEvent struct {
    Type      string
    Amount    float64
    Balance   float64  // 快照
    Timestamp time.Time
}

// 通过重放事件重建状态
func (a *Account) Rebuild() float64 {
    balance := 0.0
    for _, e := range a.events {
        switch e.Type {
        case "deposit":
            balance += e.Amount
        case "withdraw":
            balance -= e.Amount
        }
    }
    return balance
}
```

### Event Sourcing + CQRS 完整架构

```
Command Side:
Client → Command → Command Handler → Event Store → Event Bus
                                                    ↓
Query Side:                                          ↓
Read Model ← Event Handler ← Event Bus ←────────────┘

查询时：
- 直接从 Read Model 读取（快）
- 可选择重放事件重建任意时间点状态（审计）
```

---

## CQRS 的挑战与应对

| 挑战 | 解决方案 |
|------|---------|
| **数据一致性延迟** | 读写延迟可能导致脏读；接受最终一致 |
| **事件 schema 演进** | 使用版本号，Upcaster 模式 |
| **事件爆炸** | 定期 Snapshot，快照重建状态 |
| **查询复杂度增加** | Read Model 按查询模式设计 |

---

## 适用场景

**CQRS 适合：**
- 读多写少，或写多读少（明显不对称）
- 需要独立扩展读写
- 复杂查询（报表、分析）
- Event Sourcing 场景

**CQRS 不适合：**
- 简单 CRUD 系统
- 强一致性要求极高的场景
- 团队不熟悉事件驱动

---

## 常见面试问题

### Q：CQRS 和读写分离的区别？

> 传统读写分离是**同一模型的不同副本**，本质还是"拉取数据"。CQRS 是**不同模型**，读模型是专门为查询设计的视图。CQRS 可以做到 Event Sourcing，保留完整历史。

### Q：如何保证 CQRS 的最终一致性？

> 通过 Event Bus 异步同步数据；消费端幂等处理；Event Handler 失败时重试；读端接受一定延迟（通常 < 1s）。

### Q：Event Sourcing 的优缺点？

> **优点：** 完整审计日志、支持时间点回溯、易于调试、事件驱动天然集成。**缺点：** Event Schema 变更复杂、快照策略复杂、查询历史聚合麻烦。
