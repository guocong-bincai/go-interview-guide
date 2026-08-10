# DDD 战略设计：领域、限界上下文、聚合根

> 考察频率：★★★☆☆  难度：★★★★★
> 重点：能讲清楚 DDD 的核心概念，理解限界上下文的划分方法

## DDD 是什么

DDD（Domain-Driven Design）是一种以**业务领域**为核心的软件设计方法，通过深入理解业务语言来构建软件模型。

### 传统分层 vs DDD 分层

**传统三层：**
```
Controller → Service → DAO → Database
（数据为主，功能分散）
```

**DDD 四层：**
```
User Interface（用户界面）
        ↓
Application（应用服务/用例编排）
        ↓
Domain（核心业务逻辑：实体、值对象、聚合、领域服务）
        ↓
Infrastructure（持久化、消息、第三方服务）
```

---

## 核心概念

### 实体（Entity）

有唯一标识，且标识在生命周期内不变的对象。

```go
type Order struct {
    ID        string    // 唯一标识，不变
    Status    OrderStatus
    items     []OrderItem
    createdAt time.Time
}

func (o *Order) AddItem(item *OrderItem) error {
    if o.Status != OrderStatusPending {
        return ErrOrderNotModifiable  // 业务规则
    }
    o.items = append(o.items, *item)
    return nil
}
```

### 值对象（Value Object）

无唯一标识，由属性值定义，不可变。

```go
// 地址是值对象（两个地址只要属性相同就相等）
type Address struct {
    City    string
    District string
    Street  string
}

// Money 是值对象
type Money struct {
    Amount   decimal.Decimal
    Currency string
}

func (m *Money) Add(other *Money) (*Money, error) {
    if m.Currency != other.Currency {
        return nil, ErrCurrencyMismatch
    }
    return &Money{
        Amount:   m.Amount.Add(other.Amount),
        Currency: m.Currency,
    }, nil
}
```

### 聚合（Aggregate）

一组相关对象的集合，**作为数据修改的单元**。外部对象只能引用聚合根。

```go
// Order 是聚合根，管理整个订单的生命周期
type Order struct {
    id      OrderID
    items   []OrderItem
    status  OrderStatus
    userID  UserID
    
    // 业务方法在聚合根上
}

func (o *Order) AddItem(product *Product, qty int) error {
    if o.status != OrderStatusPending {
        return ErrCannotModify  // 业务规则
    }
    // ...
}

func (o *Order) Cancel() error {
    if o.status == OrderStatusShipped {
        return ErrCannotCancelShipped  // 业务规则
    }
    o.status = OrderStatusCancelled
    return nil
}

// 外部只能通过 Order（聚合根）操作，不能直接操作 OrderItem
```

---

## 限界上下文（Bounded Context）

### 概念

限界上下文是业务领域的**边界**，每个上下文有自己**独立的领域模型**。

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  电商上下文     │    │   仓库上下文     │    │   财务上下文     │
│                 │    │                 │    │                 │
│  Order          │    │  Inventory      │    │  Account         │
│  Customer       │    │  Warehouse      │    │  Ledger          │
│  Payment        │    │  Stock           │    │  Invoice         │
│                 │    │                 │    │                 │
│  库存：库存服务  │    │  库存：真实库存   │    │  库存：财务库存   │
│  (预留/冻结)     │    │  (物理实物)      │    │  (资产统计)       │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
         └──────────────────────┴──────────────────────┘
                          上下文映射
              （通过防腐层 / 消息 / API 交互）
```

### 上下文划分原则

| 原则 | 说明 |
|------|------|
| **语义边界** | 同一业务概念在不同上下文含义不同（如"库存"在仓库=实物，在财务=资产） |
| **团队边界** | 不同团队负责不同上下文，独立演进 |
| **技术边界** | 不同技术栈、数据存储 |
| **变更频率** | 一起变化的东西放一起 |

---

## 防腐层（Anti-Corruption Layer, ACL）

不同上下文交互时，用**防腐层**隔离，防止外部模型污染内部模型。

```go
// 仓库上下文的库存聚合
type WarehouseInventory struct {
    Available int
    Reserved  int
    Physical  int  // 物理库存
}

// 电商上下文，通过防腐层转换
type InventoryACL struct {
    remote *WarehouseServiceClient  // 依赖外部上下文
}

func (a *InventoryACL) GetECommerceStock(productID string) (*Stock, error) {
    // 调用外部服务
    resp, err := a.remote.GetInventory(ctx, productID)
    if err != nil {
        return nil, err
    }
    
    // 转换：外部模型 → 内部模型
    // 电商的"可用库存" = 仓库的 Physical - Reserved
    return &Stock{
        Available: resp.Physical - resp.Reserved,
        Reserved:  resp.Reserved,
    }, nil
}
```

---

## DDD 在 Go 中的落地

### 目录结构

```
internal/
├── domain/           # 领域层（核心）
│   ├── order/         # Order 限界上下文
│   │   ├── entity/    # 实体
│   │   ├── vo/        # 值对象
│   │   ├── aggregate/ # 聚合根
│   │   ├── service/   # 领域服务
│   │   └── events/    # 领域事件
│   └── user/          # User 限界上下文
├── application/      # 应用层（用例）
│   ├── order/         # Order 应用服务
│   │   ├── command/   # 命令
│   │   └── query/     # 查询
│   └── user/
├── infrastructure/   # 基础设施层
│   ├── persistence/   # 持久化
│   ├── messaging/    # 消息队列
│   └── acl/          # 防腐层
└── interface/        # 接口层
    ├── http/          # HTTP Controller
    └── grpc/          # gRPC Server
```

### 聚合根实现

```go
// domain/order/aggregate/order.go
type Order struct {
    id        OrderID
    customerID CustomerID
    items      []OrderItem
    status     OrderStatus
    totalAmount Money
    
    // 私有构造函数，只能通过工厂方法创建
    newOrder func() *Order  // 工厂
}

func NewOrder(cmd *CreateOrderCommand) (*Order, error) {
    if len(cmd.Items) == 0 {
        return nil, ErrEmptyOrder
    }
    
    order := &Order{
        id:         NewOrderID(),
        customerID: cmd.CustomerID,
        items:      []OrderItem{},
        status:     OrderStatusPending,
    }
    
    for _, item := range cmd.Items {
        if err := order.addItem(item); err != nil {
            return nil, err
        }
    }
    
    // 发布领域事件
    order.recordEvent(&OrderCreated{OrderID: order.id})
    
    return order, nil
}

// 业务方法（聚合根内部）
func (o *Order) addItem(item *OrderItem) error {
    if o.status != OrderStatusPending {
        return ErrCannotModify
    }
    o.items = append(o.items, *item)
    return nil
}

// 禁止外部直接修改 items
func (o *Order) Items() []OrderItem {
    result := make([]OrderItem, len(o.items))
    copy(result, o.items)  // 返回副本，防止外部修改
    return result
}
```

---

## 常见问题

### Q：DDD 和微服务的关系？

> DDD 的**限界上下文**往往就是微服务的边界。DDD 帮助识别业务边界，Microservices 负责技术边界。好的 DDD 设计 → 好的微服务拆分。

### Q：DDD 适合什么规模的系统？

> DDD 有一定复杂度，适合**中大型系统**（5 人以上团队，复杂业务逻辑）。简单 CRUD 系统用 DDD 反而过度设计。

### Q：聚合根设计的原则？

> 1. **聚合内数据一致**：聚合内所有对象同时修改；2. **聚合间最终一致**：跨聚合通过事件异步同步；3. **聚合要小**：太大的聚合会导致并发冲突；4. **聚合根是唯一入口**：外部只能持有聚合根引用。

### Q：DDD 的核心价值？

> 把**业务知识**沉淀到代码中，而不是散落在 SQL/注释/文档；通过限界上下文划分，**降低系统复杂度**；通用语言（Ubiquitous Language）让技术团队和业务团队**说同一种话**。
