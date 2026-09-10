# CQRS 与 Event Sourcing：区别、选择与 Go 实战

> 考察频率：★★★★☆  优先级：P1（架构设计类必考）
> 关键词：CQRS、Event Sourcing、读写分离、快照优化、投影模型、何时使用

---

## 面试官考察意图

这道题考察候选人对 **高级系统设计模式** 的深度理解。
初级知道"CQRS 就是读写分离"，"Event Sourcing 就是用事件代替表记录"——然后就说不下去了。高级要能讲清楚 **CQRS 和 Event Sourcing 的关系（它们是正交的概念）、各自的适用场景、为什么它们经常被一起使用、如何用 Go 优雅地实现它们、以及最重要的——什么时候不应该用它们**。这是面试中判断候选者是否达到 Senior/Staff 级别的关键题目。

---

## 核心答案（30秒版）

**CQRS 和 Event Sourcing 是正交的两种设计模式，可以独立或组合使用。**

- **CQRS = Command Query Responsibility Segregation**：把写操作（Command）和读操作（Query）分开建模。写用独立的 Write Model，读用独立的 Read Model。目的是让读写各自优化——比如写用关系型数据库保证一致性，读用搜索引擎/DenoSQL 追求极致查询性能。
- **Event Sourcing = 不存状态，只存事件**：不保存实体的当前状态（如 balance=100），而是保存导致状态变化的全部事件序列（如 deposit(50) → withdraw(20) → deposit(70)）。当前状态 = 重放所有事件的计算结果。

**关系**：Event Sourcing 天然需要 CQRS，因为事件存储的是命令的日志，不方便直接查询。而 CQRS 不一定需要 Event Sourcing（你可以简单地把 read model 放在 Elasticsearch 里）。

**核心取舍**：复杂度 vs 能力。引入了 Event Sourcing 后，你获得了完整的审计追踪、任意时刻的状态回放能力、天然的分布式事件流支持——但代价是系统复杂度翻倍。如果业务不需要这些能力，直接用 CRUD 就好。

---

## 深度展开

### 1. CQRS 详解

#### 1.1 传统 CRUD 的问题

```
传统单体应用：

┌───────────────┐
│    Database   │ ← 既是写的目标也是读的来源
│  (同一个 Table)│
└───▲──────┴────┘
    │          │
写 ▼          ▲ 读
 ┌─────┐    ┌─────┐
 │Cmd  │    │Query│
 └─────┘    └─────┘

问题：
1. 读多写少时，复杂的 JOIN 查询拖慢写入
2. 读写走同一个模型，无法分别优化
3. 数据量大时，索引更新影响写入性能
```

#### 1.2 CQRS 的拆分

```
CQRS 架构：

              ┌──────────────────┐
              │   Commands       │
              │  (Write Side)    │
              │                  │
write ──────▶ │ Aggregate Root   │──▶ Events ──▶ Event Store
              └──────────────────┘
                      │
            ┌─────────▼─────────┐
            │  Event Handler    │
            │  (Projection)     │──▶ Read Model DB
            └─────────┬─────────┘
                      │
              ┌───────▼────────┐
              │   Queries      │
              │  (Read Side)   │
read ◀─────── │  (Elastic /   │
              │   DenoSQL)     │
              └────────────────┘

关键特性：
1. Write Side: 强一致性 + 事务保证 + 聚合内并发控制
2. Read Side: 最终一致性 + 高性能查询 + 可独立扩展
3. 两边通过事件同步，异步但可靠
```

#### 1.3 Go 中的 CQRS 实现框架

Go 生态中有几个成熟的 CQRS 库：

| 库 | 特点 | 适用场景 |
|-----|------|---------|
| [go-cqrs](https://github.com/thediveim/go-cqrs) | 轻量，Mediator 模式 | 小型项目快速上手 |
| [eventhus](https://github.com/mishudark/eventhus) | 泛型友好，简洁 API | Go 1.18+ 项目 |
| [Watermill](https://github.com/ThreeDotsLabs/watermill) | 基于消息队列的 CQRS | 需要跨进程事件处理 |
| [axon-go](https://github.com/AxonFramework/axon-go) | Axon 框架的 Go 移植 | 大型 Enterprise 项目 |

**最小可用 CQRS 示例（纯手写）**：

```go
package cqrs

import (
    "context"
    "errors"
    "time"
)

// ========== Command side ==========

type Command interface{}

type CreateOrderCommand struct {
    UserID     string
    ItemIDs    []string
    TotalPrice float64
}

type UpdateOrderStatusCommand struct {
    OrderID string
    Status  string
}

type CommandHandler func(ctx context.Context, cmd Command) error

func (s *Service) HandleCommand(ctx context.Context, cmd Command) error {
    switch c := cmd.(type) {
    case CreateOrderCommand:
        return s.handleCreateOrder(ctx, c)
    case UpdateOrderStatusCommand:
        return s.handleUpdateStatus(ctx, c)
    default:
        return fmt.Errorf("unknown command: %T", cmd)
    }
}

// ========== Query side ==========

type Query interface{}

type GetOrderQuery struct {
    OrderID string
}

type OrderDTO struct {
    ID        string    `json:"id"`
    UserID    string    `json:"user_id"`
    Status    string    `json:"status"`
    Items     []ItemDTO `json:"items"`
    Total     float64   `json:"total"`
    CreatedAt time.Time `json:"created_at"`
}

type QueryHandler func(ctx context.Context, query Query) (interface{}, error)

func (s *Service) HandleQuery(ctx context.Context, query Query) (interface{}, error) {
    switch q := query.(type) {
    case GetOrderQuery:
        return s.handleGetOrder(ctx, q)
    default:
        return nil, fmt.Errorf("unknown query: %T", query)
    }
}

func (s *Service) handleGetOrder(ctx context.Context, q GetOrderQuery) (*OrderDTO, error) {
    // 从 Read Model 查（可能不是最新的，但最终一致）
    var dto OrderDTO
    err := s.readDB.QueryRowContext(ctx,
        "SELECT id, user_id, status, total, created_at FROM orders WHERE id = ?", q.OrderID,
    ).Scan(&dto.ID, &dto.UserID, &dto.Status, &dto.Total, &dto.CreatedAt)
    
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrOrderNotFound
    }
    if err != nil {
        return nil, fmt.Errorf("query failed: %w", err)
    }
    
    // 查 items
    rows, _ := s.readDB.QueryContext(ctx, "SELECT item_id, name, price FROM order_items WHERE order_id = ?", q.OrderID)
    for rows.Next() {
        var item ItemDTO
        rows.Scan(&item.ID, &item.Name, &item.Price)
        dto.Items = append(dto.Items, item)
    }
    
    return &dto, nil
}
```

---

### 2. Event Sourcing 详解

#### 2.1 核心概念

```
CRUD 方式（存状态）：
┌─────────────────────────────────────┐
│ orders table                        │
│ id | user_id | status  | balance    │
│ 1  | u-123   | active  | $100.00    │
│                                      │
│ 更新余额：UPDATE orders SET balance = $150 WHERE id = 1
│ ❌ 丢失了"之前是$100"的历史信息
└─────────────────────────────────────┘

Event Sourcing 方式（存事件）：
┌─────────────────────────────────────────────────────────┐
│ order_events stream                                     │
│                                                         │
│ #1: OrderCreated(order_id="1", user_id="u-123")        │
│ #2: MoneyDeposited(order_id="1", amount=$100)           │
│ #3: ItemAdded(order_id="1", item_id="item-A", qty=1)    │
│ #4: ItemAdded(order_id="1", item_id="item-B", qty=2)    │
│ #5: OrderConfirmed(order_id="1")                        │
│                                                         │
│ 当前状态 = replay(all events)                           │
│ balance = $100                                          │
│ items = [{A:1}, {B:2}]                                  │
│ status = confirmed                                      │
│                                                         │
│ ✅ 完整审计追踪                                         │
│ ✅ 可以跳到任意时间点重建状态                             │
│ ✅ 事件不可变（append only），天然安全                     │
└─────────────────────────────────────────────────────────┘
```

#### 2.2 Go 中的 Event Sourcing 实现

```go
package es

import (
    "context"
    "encoding/json"
    "fmt"
    "time"
)

// Event 是所有事件的基础接口
type Event interface {
    EventType() string
    Timestamp() time.Time
}

// AggregateRoot 是聚合根基类
type AggregateRoot struct {
    ID                string
    Version           int64
    uncommittedEvents []Event
}

func (a *AggregateRoot) Apply(event Event) {
    // 应用事件到内部状态
    switch e := event.(type) {
    case *OrderCreated:
        a.onCreated(e)
    case *MoneyDeposited:
        a.onDeposited(e)
    // ... 其他事件类型
    }
    a.uncommittedEvents = append(a.uncommittedEvents, event)
}

// OrderAggregate 是一个具体的聚合根
type OrderAggregate struct {
    AggregateRoot
    UserID      string
    Items       map[string]int
    Balance     float64
    Status      string
}

// 每个事件类型都对应一个方法，由 AggregateRoot.Apply 分发调用
func (o *OrderAggregate) onCreated(e *OrderCreated) {
    o.UserID = e.UserID
    o.Items = make(map[string]int)
    o.Balance = 0
    o.Status = "pending"
}

func (o *OrderAggregate) onDeposited(e *MoneyDeposited) {
    o.Balance += e.Amount
}

// RaiseEvent 是业务逻辑入口
func (o *OrderAggregate) PlaceOrder(userID string, items []Item) {
    if o.Status != "pending" {
        panic("order already placed")
    }
    
    for _, item := range items {
        o.Items[item.ID] = item.Quantity
    }
    o.Status = "placed"
    
    // 生成事件（不持久化！只是标记为"待提交"）
    event := &OrderPlaced{
        OrderID: o.ID,
        Items:   items,
        Time:    time.Now(),
    }
    o.Apply(event)
}

// UncommittedEvents 返回待持久化的事件
func (a *AggregateRoot) UncommittedEvents() []Event {
    return a.uncommittedEvents
}

func (a *AggregateRoot) MarkCommitted() {
    a.uncommittedEvents = nil
    a.Version++
}
```

#### 2.3 事件存储与重放

```go
// EventStore 管理事件的持久化和重放
type EventStore struct {
    db *sql.DB
}

// Append 追加新事件
func (s *EventStore) Append(ctx context.Context, aggregateID string, expectedVersion int64, events []Event) error {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // 检查版本（乐观锁）
    var currentVersion int64
    err = tx.QueryRow("SELECT version FROM aggregates WHERE id = ?", aggregateID).Scan(&currentVersion)
    if err != nil && !errors.Is(err, sql.ErrNoRows) {
        return err
    }
    if currentVersion != expectedVersion {
        return fmt.Errorf("version conflict: expected %d, got %d", expectedVersion, currentVersion)
    }
    
    // 插入所有事件
    insertStmt, _ := tx.Prepare("INSERT INTO events (aggregate_id, version, type, data, timestamp) VALUES (?, ?, ?, ?, ?)")
    defer insertStmt.Close()
    
    version := expectedVersion
    for _, event := range events {
        data, _ := json.Marshal(event)
        version++
        _, err := insertStmt.Exec(aggregateID, version, event.EventType(), data, event.Timestamp())
        if err != nil {
            return err
        }
    }
    
    // 创建或更新聚合元数据
    _, err = tx.Exec(`INSERT INTO aggregates (id, version) VALUES (?, ?) 
                       ON CONFLICT(id) DO UPDATE SET version = excluded.version`,
        aggregateID, version)
    if err != nil {
        return err
    }
    
    return tx.Commit()
}

// Load 从重放事件恢复聚合状态
func (s *EventStore) Load(ctx context.Context, aggregateID string) ([]Event, error) {
    rows, err := s.db.QueryContext(ctx,
        "SELECT type, data FROM events WHERE aggregate_id = ? ORDER BY version ASC",
        aggregateID,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var events []Event
    for rows.Next() {
        var eventType string
        var data []byte
        rows.Scan(&eventType, &data)
        
        var event Event
        switch eventType {
        case "OrderCreated":
            event = &OrderCreated{}
        case "MoneyDeposited":
            event = &MoneyDeposited{}
        // ...
        }
        json.Unmarshal(data, event)
        events = append(events, event)
    }
    
    return events, nil
}
```

---

### 3. 快照优化

当事件历史过长时，每次从头重放所有事件太慢了。引入**快照（Snapshot）**机制：

```
无快照的重放（10000个事件）：
Event #1 ─▶ Event #2 ─▶ ... ─▶ Event #10000
时间：约 2~5 秒（取决于事件数量和处理逻辑）

带快照的重放：
Snapshot (v9000) ─▶ Event #9001 ─▶ ... ─▶ Event #10000
              ▲
         之前的事件跳过！
时间：约 50~100ms（只需重放最后1000个事件）

快照策略：
• 每 N 个事件存一次快照（如 1000 个）
• Snapshot 存储当前的完整状态（而不是事件）
• 加载时先还原快照，再重放快照之后的事件
```

```go
// 带快照的 LoadWithSnapshot
func (s *EventStore) LoadWithSnapshot(ctx context.Context, aggregateID string) ([]Event, error) {
    // 1. 尝试加载最近的快照
    snapshot, snapVersion, found := s.loadLatestSnapshot(ctx, aggregateID)
    if !found {
        return s.Load(ctx, aggregateID) // 没有快照就全量重放
    }
    
    // 2. 从快照之后加载新事件
    rows, err := s.db.QueryContext(ctx,
        "SELECT type, data FROM events WHERE aggregate_id = ? AND version > ? ORDER BY version ASC",
        aggregateID, snapVersion,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    // 3. 从快照状态开始，重放后续事件
    agg := NewAggregateFromSnapshot(snapshot)
    for rows.Next() {
        var eventType string
        var data []byte
        rows.Scan(&eventType, &data)
        
        var event Event
        // 反序列化...
        agg.Apply(event)
    }
    
    return agg.UncommittedEvents(), nil
}
```

---

### 4. 何时使用？（最重要的一道题）

```
应该使用 Event Sourcing 的场景：
✅ 需要完整的审计追踪（金融、医疗合规）
✅ 需要回溯到任意时间点查看状态（数据事故排查）
✅ 有复杂的领域规则，事件的完整性很重要
✅ 需要构建实时仪表盘（基于事件流的实时计算）
✅ CQRS 架构下的 Write Side（天然适合事件驱动）

不应该使用的场景：
❌ 简单的 CRUD 业务（库存商品列表、文章发布）
❌ 团队不了解 ES 概念（学习成本高）
❌ 需要频繁更新实体（ES 本质是 append-only，不适合 UPDATE）
❌ 对实时性要求不高且数据量小（过度设计）
❌ 需要灵活的多维查询（Event Store 通常只有一个查询视角）
```

**经典面试金句**：
> "Event Sourcing 不是银弹。如果你的业务只需要简单的增删改查，不要用 Event Sourcing。但如果你的业务需要审计追踪、合规审查、或者复杂的领域逻辑验证——那它就是值得投入的技术债。"

---

## 面试话术

**Q：CQRS 和 Event Sourcing 的区别是什么？**

> 这是两个独立的设计模式，虽然经常被一起使用：
> 
> **CQRS 解决的是"读写分离"问题**——写用独立的模型保证一致性，读用独立的模型追求查询性能。它关注的是系统的结构分层。
> 
> **Event Sourcing 解决的是"状态存储"问题**——不存当前状态，只存导致状态变化的事件序列。它关注的是数据的表示形式。
> 
> 你可以只用 CQRS 不用 ES（读模型放 ES，写模型放 MySQL），也可以只用 ES 不用 CQRS（事件流同时用于写和读）。但把它们组合使用时效果最好：ES 作为 CQRS 的写模型，投影自动构建读模型。
> 
> 🗣️ **记忆口诀**：**"CQRS分读写，ES存事件，组合起来更强大"**

---

*参考文档：[Greg Young - Event Sourcing](https://cqrs.ninja/) · [Threedots/cqrs-example](https://github.com/ThreeDotsLabs/cqrs-example) · [Event Store DB](https://eventstore.org/)*

🏠 [首页](../../../README.md) · 📦 [分布式系统](../README.md)
