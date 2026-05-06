# DDD 战术设计：实体、聚合、值对象与工程落地

> 考察频率：★★★★★  优先级：P0
> 关键词：DDD、聚合根、值对象、实体、领域事件、Repository、工程落地

## DDD 战术设计是什么

DDD（Domain-Driven Design）分为**战略设计**和**战术设计**：
- **战略设计**：限界上下文（Bounded Context）、上下文映射（Context Mapping）
- **战术设计**：如何在代码层面落地 DDD，包括**聚合、实体、值对象、领域事件、Repository**

---

## 1. 核心概念：实体 vs 值对象

### 实体（Entity）

**实体 = 有唯一标识 + 生命周期 + 可变性**

```go
// 实体：Order 有全局唯一 ID，状态可变（pending → paid → shipped）
type Order struct {
    ID        string        // 全局唯一标识（不能重复）
    UserID    string
    Status    OrderStatus   // 可变：pending → paid → shipped → completed
    Items     []OrderItem   // 可变：可以增删商品
    CreatedAt time.Time
    UpdatedAt time.Time
}

type OrderStatus string

const (
    StatusPending   OrderStatus = "pending"
    StatusPaid      OrderStatus = "paid"
    StatusShipped   OrderStatus = "shipped"
    StatusCompleted OrderStatus = "completed"
)

// 实体相等性判断：只看 ID，不看其他字段
func (o *Order) Equals(other *Order) bool {
    return o.ID == other.ID  // 即使其他字段不同，只要 ID 相同就是同一个订单
}
```

**实体的特点：**
- 有全局唯一标识（订单号、用户 ID）
- 可以跨系统存在（订单创建后，可能存在于支付系统、物流系统）
- 同一 ID 的两个实例代表同一个实体

### 值对象（Value Object）

**值对象 = 无唯一标识 + 不可变 + 可复用**

```go
// 值对象：Money 代表金额，不关心谁付的，只关心金额本身
// 两个 Money{100, "CNY"} 和 Money{100, "CNY"} 是相等的（不需要 ID）
type Money struct {
    Amount   decimal.Decimal  // 金额
    Currency string           // 币种（CNY/USD）
}

// 值对象：Address（门牌号、城市、街道），描述性，不关心谁住
type Address struct {
    Province string
    City     string
    District string
    Detail   string
}

// 值对象必须是不可变的（修改时返回新对象）
func (a Address) WithDetail(newDetail string) Address {
    a.Detail = newDetail  // 值对象修改只能这样：新建一个
    return a
}
```

**值对象 vs 实体的核心区别：**

| 维度 | 实体 | 值对象 |
|------|------|--------|
| 唯一标识 | ✅ 有 | ❌ 无 |
| 可变性 | 可变 | 不可变 |
| 相等性判断 | ID 相同 = 同一实体 | 所有字段相同 = 相等 |
| 生命周期 | 跨系统持久化 | 依附实体存在 |

---

## 2. 聚合（Aggregate）与聚合根（Aggregate Root）

### 为什么需要聚合

**问题：订单和订单明细，修改时需要一起成功或一起失败。**

```
场景：创建订单时，订单和订单明细必须同时创建
      取消订单时，订单和订单明细必须同时取消
      如果分开保存，订单创建成功但明细创建失败 → 数据不一致

解决方案：聚合
- 把密切相关的实体和值对象打包成一个聚合
- 聚合内保证数据一致性（原子性）
- 聚合之间通过 ID 引用，不直接引用内部对象
```

### 聚合设计原则

```
聚合设计黄金法则：
1. 一个聚合只有一个聚合根（Aggregate Root）
2. 聚合外部对象不能直接修改聚合内部状态
3. 聚合之间只能通过聚合根 ID 引用
4. 聚合边界内保持事务一致性
```

```go
// Order（聚合根） + OrderItem（内部实体） + Money（值对象） = 订单聚合

// ✅ 正确的聚合设计：
// 1. 聚合根是 Order，只有 Order 能修改内部状态
type Order struct {
    ID        string        // 聚合根 ID
    userID    string        // 私有字段（外部不可直接修改）
    status    OrderStatus
    items     []OrderItem   // 内部实体，通过聚合根方法访问
}

// ✅ 外部只能通过聚合根方法修改
func (o *Order) AddItem(productID string, price Money, qty int) error {
    if o.status != StatusPending {
        return errors.New("only pending order can add item")
    }
    // 业务不变式：订单未支付才能添加商品
    o.items = append(o.items, OrderItem{...})
    return nil
}

// ❌ 错误的设计：外部直接修改内部对象
// o.items[0].Qty = 100  // 绕过了聚合的业务不变式！

// ✅ 正确的修改方式：
func (o *Order) UpdateItemQty(itemID string, qty int) error {
    for i := range o.items {
        if o.items[i].ID == itemID {
            // 可以在此处添加业务规则验证
            o.items[i].Qty = qty
            return nil
        }
    }
    return errors.New("item not found")
}
```

### 聚合边界如何划分

**划分的核心标准：高内聚 + 事务一致性 + 边界清晰**

```
❌ 错误划分：把所有东西放一个聚合
Order ← OrderItem ← Product ← Category ← Seller ← ...
问题：任何一个对象变化都导致整个聚合重新加载/保存

✅ 正确划分：按事务边界划分
Order 聚合：Order（根）+ OrderItem（子实体）+ Money（值对象）
Product 聚合：Product（根）+ Category（子实体）
User 聚合：User（根）+ Address（值对象）

聚合间通过 ID 引用：
type Order struct {
    ID           string
    userID       string  // ✅ 引用用户 ID，而非 User 实体
    productIDs   []string  // ✅ 引用商品 ID 列表
}
```

---

## 3. 领域服务 vs 应用服务

### 领域服务（Domain Service）

**处理跨聚合的业务逻辑，或不适合放在任何单一聚合内的操作**

```go
// 领域服务：订单扣减库存（涉及 Order 聚合 + Inventory 聚合）
type OrderDomainService struct {
    orderRepo      OrderRepository
    inventoryRepo  InventoryRepository
}

// 扣减库存并创建订单（在同一个事务中）
func (s *OrderDomainService) CreateOrderAndDeductStock(
    ctx context.Context,
    userID string,
    items []OrderItemRequest,
) (*Order, error) {

    // 1. 检查库存（跨聚合操作）
    for _, item := range items {
        ok, err := s.inventoryRepo.Deduct(ctx, item.ProductID, item.Qty)
        if err != nil {
            return nil, err
        }
        if !ok {
            return nil, fmt.Errorf("库存不足: product=%s", item.ProductID)
        }
    }

    // 2. 创建订单
    order, err := s.orderRepo.Create(ctx, userID, items)
    if err != nil {
        // 回滚库存（需要补偿逻辑或 TCC）
        for _, item := range items {
            s.inventoryRepo.Restore(ctx, item.ProductID, item.Qty)
        }
        return nil, err
    }
    return order, nil
}
```

### 应用服务（Application Service）

**编排领域对象和领域服务，处理用例（Use Case），不包含业务逻辑**

```go
// 应用服务：处理"用户下单"这个用例
type OrderAppService struct {
    orderDomainService *OrderDomainService
    eventBus           EventBus  // 发布领域事件
}

type CreateOrderRequest struct {
    UserID  string
    Items   []OrderItemRequest
}

func (s *OrderAppService) CreateOrder(
    ctx context.Context,
    req CreateOrderRequest,
) (*CreateOrderResponse, error) {

    // 1. 参数校验（应用层校验）
    if len(req.Items) == 0 {
        return nil, errors.New("订单不能为空")
    }

    // 2. 调用领域服务（业务逻辑在领域层）
    order, err := s.orderDomainService.CreateOrderAndDeductStock(ctx, req.UserID, req.Items)
    if err != nil {
        return nil, err
    }

    // 3. 发布领域事件（通知其他限界上下文）
    s.eventBus.Publish(ctx, OrderCreatedEvent{
        OrderID: order.ID,
        UserID:  order.UserID,
        Amount:  order.TotalAmount(),
        Items:   order.Items,
    })

    // 4. 返回结果（DTO，不暴露内部结构）
    return &CreateOrderResponse{
        OrderID:      order.ID,
        Status:       order.Status,
        CreatedAt:    order.CreatedAt,
    }, nil
}
```

**领域服务 vs 应用服务对比：**

| 维度 | 领域服务 | 应用服务 |
|------|---------|---------|
| 所在层 | 领域层（Domain）| 应用层（Application）|
| 业务逻辑 | ✅ 包含真正的业务规则 | ❌ 不包含（只编排）|
| 跨聚合协调 | ✅ 典型场景 | ✅ 也可能涉及 |
| 事务边界 | 负责开启事务 | 调用领域服务 |
| 是否发布事件 | ❌ 不发布 | ✅ 发布领域事件 |

---

## 4. Repository 模式在 Go 中的落地

### Repository 接口定义（领域层）

```go
// Repository 接口放在领域层（domain 包）
// 实现放在 infrastructure 层

type OrderRepository interface {
    FindByID(ctx context.Context, id string) (*Order, error)
    FindByUserID(ctx context.Context, userID string, offset, limit int) ([]*Order, error)
    Save(ctx context.Context, order *Order) error   // Create + Update 合一
    Delete(ctx context.Context, id string) error

    // 批量操作（聚合根级别的事务）
    SaveAll(ctx context.Context, orders []*Order) error
}
```

### Repository 实现（基础设施层）

```go
// infrastructure/persistence/order_repository.go

type OrderRepositoryImpl struct {
    db *sqlx.DB
}

func (r *OrderRepositoryImpl) FindByID(ctx context.Context, id string) (*Order, error) {
    var order Order
    err := r.db.GetContext(ctx, &order, "SELECT * FROM orders WHERE id = ?", id)
    if err == sql.ErrNoRows {
        return nil, ErrOrderNotFound
    }
    if err != nil {
        return nil, err
    }

    // 加载聚合内所有子实体
    var items []OrderItem
    r.db.SelectContext(ctx, &items, "SELECT * FROM order_items WHERE order_id = ?", id)
    order.items = items

    return &order, nil
}

func (r *OrderRepositoryImpl) Save(ctx context.Context, order *Order) error {
    return r.db.TX(ctx, func(tx *sqlx.Tx) error {
        // 先 upsert 聚合根
        _, err := tx.ExecContext(ctx, `
            INSERT INTO orders (id, user_id, status, created_at, updated_at)
            VALUES (?, ?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE status = ?, updated_at = ?`,
            order.ID, order.UserID, order.Status, order.CreatedAt, order.UpdatedAt,
            order.Status, order.UpdatedAt)
        if err != nil {
            return err
        }

        // 删除旧子实体，重新插入（简化处理）
        tx.ExecContext(ctx, "DELETE FROM order_items WHERE order_id = ?", order.ID)
        for _, item := range order.items {
            _, err := tx.ExecContext(ctx, `
                INSERT INTO order_items (id, order_id, product_id, qty, price)
                VALUES (?, ?, ?, ?, ?)`,
                item.ID, order.ID, item.ProductID, item.Qty, item.Price)
            if err != nil {
                return err
            }
        }
        return nil
    })
}
```

---

## 5. 如何避免 DDD 过度设计

### DDD 适用场景判断

```
✅ 需要 DDD 的场景：
- 业务复杂（金融、订单、库存、物流）
- 团队 > 10 人，多个团队协作
- 需要长期维护和演进

❌ 不需要 DDD 的场景：
- CRUD 为主（用户管理、新闻系统）
- 业务简单，贫血模型足够
- 团队 < 5 人，快速迭代阶段
```

### 实用主义 DDD（不执着于术语）

```go
// ❌ 过度设计：所有实体都要搞聚合根、值对象、领域服务
type UserAggregateRoot struct {
    id       string
    email    EmailValueObject
    profile  ProfileValueObject
    address  AddressValueObject
    // 5 层包装，调用时要：user.Profile.Address.City
}

// ✅ 实用主义：简单场景用简单方式
type User struct {
    ID     string
    Email  string  // Email 直接用 string，不需要单独类型
    Name   string
    City   string  // 直接字段，不用包装
}

// 只有在需要封装业务规则的地方才用 DDD 概念
// 例如：金额计算需要封装（Currency + Amount），用 Money 值对象
```

### 过渡策略：从简单到复杂

```
第一阶段（贫血模型，快速上线）：
User{id, name, email} + UserService{}
      ↓ 重构
第二阶段（引入聚合）：
User{id, email, Profile{}} + ProfileService{}
      ↓ 演进
第三阶段（完整 DDD）：
User（聚合根）+ Profile（值对象）+ Address（值对象）
+ UserDomainService + UserRepository
```

---

## 6. 完整示例：订单 + 库存 DDD 实践

### 限界上下文划分

```
┌─────────────────────────────────────┐
│         订单上下文（Order Context）   │
│  聚合：Order（聚合根）               │
│  实体：Order, OrderItem              │
│  值对象：Money, Address              │
│  仓储：OrderRepository              │
└─────────────────────────────────────┘
           ↑ 领域事件
┌─────────────────────────────────────┐
│        库存上下文（Inventory Context）│
│  聚合：Inventory（聚合根）           │
│  实体：Inventory                    │
│  值对象：SKU                        │
│  仓储：InventoryRepository          │
└─────────────────────────────────────┘
```

### 核心代码

```go
// ========== 订单聚合 ==========
type Order struct {
    id      string
    userID  string
    status  OrderStatus
    items   []OrderItem
    total   Money  // 值对象
}

func (o *Order) AddItem(productID string, price Money, qty int) error {
    if o.status != StatusPending {
        return errors.New("只能向待支付订单添加商品")
    }
    o.items = append(o.items, OrderItem{
        ID:    uuid.New().String(),
        PID:   productID,
        Price: price,
        Qty:   qty,
    })
    return nil
}

func (o *Order) Pay() error {
    if len(o.items) == 0 {
        return errors.New("订单不能为空")
    }
    o.status = StatusPaid
    return nil
}

// ========== 库存聚合 ==========
type Inventory struct {
    sku        string
    available  int   // 可用库存
    reserved   int   // 预留库存（支付中）
    sold       int   // 已售
}

func (i *Inventory) Reserve(qty int) error {
    if i.available < qty {
        return errors.New("库存不足")
    }
    i.available -= qty
    i.reserved += qty
    return nil
}

func (i *Inventory) ConfirmSold(qty int) {
    i.reserved -= qty
    i.sold += qty
}

// ========== 跨聚合协调（领域服务）==========
type PlaceOrderService struct {
    orderRepo     OrderRepository
    inventoryRepo InventoryRepository
    eventBus      DomainEventBus
}

func (s *PlaceOrderService) PlaceOrder(ctx context.Context, req PlaceOrderReq) (*Order, error) {
    // 1. 预留库存（先扣库存，防止超卖）
    inventory, err := s.inventoryRepo.FindBySKU(ctx, req.ProductID)
    if err != nil {
        return nil, err
    }
    if err := inventory.Reserve(req.Qty); err != nil {
        return nil, err
    }
    s.inventoryRepo.Save(ctx, inventory)

    // 2. 创建订单
    order := NewOrder(req.UserID)
    order.AddItem(req.ProductID, req.Price, req.Qty)
    if err := order.Pay(); err != nil {
        // 释放预留库存
        inventory.RestoreReserved(req.Qty)
        s.inventoryRepo.Save(ctx, inventory)
        return nil, err
    }

    orderRepo.Save(ctx, order)

    // 3. 发布领域事件（通知库存确认）
    s.eventBus.Publish(ctx, OrderPaidEvent{
        OrderID:   order.ID,
        ProductID: req.ProductID,
        Qty:       req.Qty,
    })

    return order, nil
}
```

---

## 高频追问

### Q1：聚合边界如何划分？

**核心原则：事务一致性边界内，边界外最终一致性**

划分步骤：
1. **找不变式（Invariant）**：哪些操作必须同时成功/失败？
2. **找高频操作**：哪些数据总是被一起访问？
3. **避免跨聚合引用**：聚合 A 不能直接持有聚合 B 的内部对象

**实操经验：**
- 订单 + 订单明细 → 同一聚合（明细不能脱离订单存在）
- 用户 + 用户地址 → 同一聚合（地址属于用户）
- 商品 + 分类 → 不同聚合（分类可以被多个商品引用）

### Q2：DDD 和传统三层架构如何结合？

```
DDD 分层（与三层架构对应）：

DDD 应用层  ←→  三层：Service 层（用例编排）
DDD 领域层  ←→  三层：Domain 层（业务逻辑）
DDD 基础设施层 ←→ 三层：DAO/Repository 层（数据访问）

Go 项目推荐目录结构：
├── domain/          # 领域层（实体、值对象、聚合、领域服务）
│   ├── order/
│   │   ├── order.go        # 聚合根
│   │   ├── order_item.go   # 子实体
│   │   ├── money.go        # 值对象
│   │   └── repository.go   # 仓储接口（领域层定义）
│   └── inventory/
├── application/     # 应用层（用例、DTO、应用服务）
│   └── order_service.go
├── infrastructure/ # 基础设施层（仓储实现、MQ、第三方服务）
│   └── persistence/
│       ├── order_repo.go
│       └── inventory_repo.go
└── interfaces/      # 接口层（HTTP、gRPC）
    └── order_handler.go
```

### Q3：Go 里贫血模型 vs 充血模型怎么选？

**贫血模型（Anemic Domain Model）：**
```go
// 贫血：所有逻辑在 Service 层，对象只是数据结构
type Order struct { ID string; UserID string; Amount float64 }
func OrderService_CreateOrder() { /* 所有逻辑在 service */ }
```
**缺点：** 业务逻辑散落在各处，难以维护；对象失去语义

**充血模型（Rich Domain Model）：**
```go
// 充血：业务逻辑在聚合根和实体内部，对象自己知道怎么玩
func (o *Order) Pay() error { /* 业务逻辑在聚合根 */ }
```
**缺点：** 对团队 DDD 能力要求高

**Go 实战建议：**
- 简单 CRUD：贫血模型足够，不需要强行 DDD
- 复杂业务：充血模型 + 聚合根 + 领域服务
- 过渡方案：从贫血逐步演进到充血，先把业务逻辑收到 domain 对象里
