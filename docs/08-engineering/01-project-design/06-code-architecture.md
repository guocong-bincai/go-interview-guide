# Go 服务架构设计：分层、整洁架构与六边形架构

> 考察频率：★★★★☆  优先级：P1
> 关键词：分层架构、Clean Architecture、六边形架构、依赖注入、接口设计

---

## 面试官考察意图

这道题考的是**你能否设计一个可维护、可测试、可扩展的 Go 服务架构**。面试官想看到的是：

- 你对**常见架构模式的理解**（分层 vs 整洁 vs 六边形）
- 你能否**根据业务场景选择合适的架构**
- 你是否有**架构演进的思维**（不是一步到位，而是随业务生长）

**高级工程师的回答** vs **初级工程师的回答**：

| 维度 | 初级 | 高级 |
|------|------|------|
| 架构选择 | "大家都这么写我也这么写" | 根据团队规模、业务复杂度、变更频率选择 |
| 分层边界 | "handler 调 service 调 repo" | 清晰的接口契约、依赖单向、领域逻辑内聚 |
| 可测试性 | "service 没法测因为依赖 DB" | 用依赖注入 + mock，service 层可独立单测 |
| 扩展性 | "加新功能就改老代码" | 用接口隔离变化，新增实现不影响已有代码 |
| 演进思维 | 一步到位画大图 | 渐进式演进：小团队先用简单架构，业务复杂了再拆分 |

---

## 核心答案（30秒版）

Go 服务架构三条路：**简单分层**（handler → service → repo，适合小团队快速迭代）→ **整洁架构**（领域层独立，依赖倒置，适合业务复杂的中型团队）→ **六边形架构**（端口适配器分离，适合多团队协作和长期演进）。选型关键看：团队规模、变更频率、是否需要多端适配。

---

## 深度展开

### 一、Go 服务的典型分层架构

#### 标准三层（最常用）

```
┌─────────────────────────────────────────────┐
│  Handler层（接口层）                           │
│  - 参数解析、参数校验、响应组装                  │
│  - HTTP路由、Middleware                        │
├─────────────────────────────────────────────┤
│  Service层（业务逻辑层）                        │
│  - 业务规则、事务控制、跨服务调用               │
│  - 纯业务逻辑，不直接依赖 HTTP/DB              │
├─────────────────────────────────────────────┤
│  Repo层（数据访问层）                           │
│  - DB操作、Redis操作、第三方API调用            │
│  - 对下层基础设施的封装                        │
└─────────────────────────────────────────────┘
```

**典型代码实现**：

```go
// handler/order_handler.go
type OrderHandler struct {
    svc orderService
}

func (h *OrderHandler) CreateOrder(c *gin.Context) {
    var req CreateOrderReq
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    
    // 业务调用
    order, err := h.svc.CreateOrder(c.Request.Context(), req)
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(200, order)
}

// service/order_service.go
type orderService interface {
    CreateOrder(ctx context.Context, req CreateOrderReq) (*Order, error)
}

type OrderService struct {
    repo     orderRepo
    producer KafkaProducer  // 依赖接口而非具体实现
}

func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderReq) (*Order, error) {
    // 业务校验
    if req.TotalAmount <= 0 {
        return nil, ErrInvalidAmount
    }
    
    // 事务控制
    order, err := s.repo.CreateOrderTx(ctx, func(tx *sql.Tx) (*Order, error) {
        // 创建订单
        order := &Order{
            UserID:      req.UserID,
            TotalAmount: req.TotalAmount,
            Status:      StatusPending,
        }
        return s.repo.Create(ctx, order, tx)
    })
    if err != nil {
        return nil, err
    }
    
    // 发消息（异步，不影响主流程）
    go s.producer.Send("order.created", order)
    
    return order, nil
}

// repo/order_repo.go
type orderRepo interface {
    Create(ctx context.Context, order *Order, tx *sql.Tx) (*Order, error)
    CreateOrderTx(ctx context.Context, fn func(*sql.Tx) (*Order, error)) (*Order, error)
}

type OrderRepo struct {
    db *sql.DB
}

func (r *OrderRepo) Create(ctx context.Context, order *Order, tx *sql.Tx) (*Order, error) {
    // 实现...
}
```

**三层架构的优点**：
- 简单直观，团队共识度高
- 适合小团队（<10人）快速迭代

**三层架构的缺点**：
- 业务逻辑散落在 Service 层，随着业务增长 Service 会变成"上帝类"
- 对数据库的依赖导致 Service 难以独立测试
- 换存储层（如从 MySQL 换到 TiDB）要改 Service 层

### 二、整洁架构（Clean Architecture）

#### 核心理念

```
         ┌─────────────────────────────────────┐
         │        Interface Adapter 层          │
         │   (Handler / Presenter / Gateway)    │
         ├─────────────────────────────────────┤
         │          Ports 层                    │
         │   (入站接口：UseCase                │
         │    出站接口：Repo/Publisher)         │
         ├─────────────────────────────────────┤
         │         Domain 层（核心）             │
         │   (Entity / Value Object / Domain   │
         │    Service / 业务规则)               │
         └─────────────────────────────────────┘
              ↑ 所有依赖指向内部
```

**与三层的本质区别**：领域层（Domain）不依赖任何外部基础设施，依赖方向统一向内。

#### Go 整洁架构实现

```go
// ==================== Domain 层（最核心，完全独立）====================

// domain/order/order.go
// 领域实体：只包含业务规则，不含框架依赖
type Order struct {
    id          OrderID
    customerID  CustomerID
    lineItems   []LineItem
    totalAmount Money
    status      OrderStatus
    createdAt   time.Time
}

func (o *Order) CanCancel() bool {
    return o.status == StatusPending || o.status == StatusConfirmed
}

func (o *Order) Cancel(reason string) error {
    if !o.CanCancel() {
        return ErrCannotCancel
    }
    o.status = StatusCancelled
    return nil
}

func (o *Order) AddItem(item LineItem) error {
    if item.Quantity <= 0 {
        return ErrInvalidQuantity
    }
    o.lineItems = append(o.lineItems, item)
    o.recalculateTotal()
    return nil
}

func (o *Order) recalculateTotal() {
    var total Money
    for _, item := range o.lineItems {
        total = total.Add(item.Subtotal())
    }
    o.totalAmount = total
}

// 领域服务：跨实体业务规则
type OrderDomainService struct{}

func (s *OrderDomainService) ValidateOrder(o *Order) error {
    if len(o.lineItems) == 0 {
        return ErrEmptyOrder
    }
    if o.totalAmount.IsNegative() {
        return ErrNegativeTotal
    }
    return nil
}

// ==================== Application 层（用例）====================

// usecase/order_usecase.go
// 用例：编排领域对象和应用服务，不含业务规则

type OrderUseCase interface {
    CreateOrder(ctx context.Context, cmd CreateOrderCmd) (*Order, error)
    CancelOrder(ctx context.Context, cmd CancelOrderCmd) error
}

type CreateOrderCmd struct {
    CustomerID  CustomerID
    LineItems   []LineItemDTO
}

type CancelOrderCmd struct {
    OrderID     OrderID
    Reason      string
}

type orderUseCase struct {
    repo         repository.OrderRepository
    publisher    event.Publisher
    domainSvc    *OrderDomainService
}

func (uc *orderUseCase) CreateOrder(ctx context.Context, cmd CreateOrderCmd) (*Order, error) {
    // 1. 构造领域对象
    order := NewOrder(cmd.CustomerID)
    for _, item := range cmd.LineItems {
        order.AddItem(item.ToDomain())
    }
    
    // 2. 领域规则校验
    if err := uc.domainSvc.ValidateOrder(order); err != nil {
        return nil, err
    }
    
    // 3. 持久化
    if err := uc.repo.Save(ctx, order); err != nil {
        return nil, err
    }
    
    // 4. 发布领域事件（解耦）
    uc.publisher.Publish(event.OrderCreated{OrderID: order.ID()})
    
    return order, nil
}

// ==================== Infrastructure 层（适配器）====================

// adapter/repository/mysql_order_repo.go
// 实现 repository.OrderRepository 接口

type MySQLOrderRepo struct {
    db *sql.DB
}

func (r *MySQLOrderRepo) Save(ctx context.Context, order *Order) error {
    // MySQL 特有实现
    // MySQL → Domain 转换
}

func (r *MySQLOrderRepo) Find(ctx context.Context, id OrderID) (*Order, error) {
    // ...
}

// adapter/messaging/kafka_publisher.go
// 实现 event.Publisher 接口

type KafkaPublisher struct {
    producer *kafka.Producer
}

func (p *KafkaPublisher) Publish(evt event.Event) error {
    // Kafka 发布实现
}

// ==================== 接口定义（Ports）====================

// repository/order_repository.go
// 定义仓库接口（属于 Ports 层）
// 应用层依赖接口，基础设施层实现接口

package repository

type OrderRepository interface {
    Save(ctx context.Context, order *Order) error
    Find(ctx context.Context, id OrderID) (*Order, error)
    FindByCustomer(ctx context.Context, customerID CustomerID) ([]*Order, error)
}

// event/publisher.go
type Publisher interface {
    Publish(evt event.Event) error
}
```

**整洁架构的优点**：
- 领域逻辑完全独立，可以不依赖任何框架做单元测试
- 换存储层/换消息队列只需新增适配器，不改业务代码
- 便于多团队协作（接口契约先行）

**整洁架构的缺点**：
- 代码量大（需要大量接口和转换代码）
- 初期学习成本高，团队需要有 DDD 基础

### 三、六边形架构（Ports & Adapters）

六边形架构和整洁架构本质相同，只是命名不同：

```
                    ┌──────────────────┐
    HTTP/REST  ──→  │   Inbound Ports   │──→  (Primary/Driving Adapters)
    gRPC        ──→  │  (Input Ports)    │
    GraphQL    ──→  │                   │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │    Application    │
                    │    (Core Logic)   │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
    MySQL          │   Outbound Ports  │──→  (Secondary/Driven Adapters)
    Redis          │  (Output Ports)   │
    Kafka          │                   │
                    └──────────────────┘
```

**六边形在 Go 中的简化实现**：

```go
// cmd/server/main.go - 入口，组装所有组件
func main() {
    // 1. 初始化基础设施
    db, _ := NewDB()
    kafka, _ := NewKafkaProducer()
    
    // 2. 用 wire/dig 做依赖注入
    container := wire.New(
        repository.NewMySQLOrderRepo,  // 绑定实现
        event.NewKafkaPublisher,
        usecase.NewOrderUseCase,
        handler.NewOrderHandler,
    )
    
    // 3. 启动 HTTP server
    httpServer := container.GetHTTPHandler()
    httpServer.Run(":8080")
}
```

### 四、架构选型决策树

```
业务场景 → 架构选择
────────────────────────────────────────────────────────────
小团队（<5人）、业务简单、快速迭代
→ 简单三层架构（handler → service → repo）

中型团队（5-20人）、业务复杂度中等
→ 领域驱动分层（Domain 层 + 应用层 + 基础设施层）

大型团队（>20人）、业务复杂、长期演进
→ 六边形架构 / 整洁架构（明确的限界上下文）

需要同时服务 Web / Mobile / 第三方 API
→ 六边形架构（端口隔离，适配器可替换）

业务逻辑经常变更，需要高度可测试
→ 整洁架构（核心逻辑完全独立）
```

### 五、架构演进实战案例

**阶段1：MVP 阶段（小团队，3个月）**

```
// 简单三层，一个文件打天下
// allinone.go - 300行
// Handler + Service + Repo 全在一起
```

**阶段2：增长期（6-12个月，团队扩到8人）**

```
// 拆分为标准三层
// internal/
//   handler/   - HTTP 入口
//   service/   - 业务逻辑
//   repo/      - DB/Redis 操作
```

**阶段3：平台期（12个月+，团队20人+）**

```
// 按限界上下文拆分
// internal/
//   order/         - 订单上下文（有自己的 domain/service/repo）
//   inventory/     - 库存上下文
//   payment/       - 支付上下文
// pkg/
//   shared/        - 跨上下文共享的领域概念
```

**演进原则**：
- 不要过度设计：阶段1不需要六边形
- 按需拆分：当某个模块 >2000 行、或 >3 人并行开发时拆分
- 架构演进要有理由：不是"这个更先进"，而是"业务需要"

### 六、Go 架构设计常见反模式

| 反模式 | 问题 | 正确做法 |
|--------|------|---------|
| 在 Service 里直接 `sql.Open` | 依赖具体实现，难以 mock 测试 | 通过接口注入 `*sql.DB` |
| 全局变量做 Repo/Svc 单例 | 难以在测试中替换 mock | 用依赖注入 |
| Handler 写业务逻辑 | Controller 膨胀 | 业务逻辑下沉 Service |
| 循环依赖（pkg A → B，B → A） | 编译不过 | 用接口解环，或合并为一个包 |
| Repository 直接返回 `map[string]any` | 丧失类型安全 | 返回强类型领域对象 |
| 用 JSON struct tag 做领域模型 | 领域模型被 DB/JSON 污染 | 分层：Domain struct / DB struct / DTO |

---

## 高频追问

**Q1：分层架构下，Service 层如何做单元测试？**

用依赖注入 + mock：
```go
// Service 依赖接口而非具体实现
type OrderService struct {
    repo     OrderRepo
    producer MessagePublisher
}

// 测试时注入 mock
func TestOrderService_Create(t *testing.T) {
    mockRepo := &MockOrderRepo{}
    mockPub := &MockPublisher{}
    
    svc := NewOrderService(mockRepo, mockPub)
    // 测试逻辑，不涉及任何外部依赖
}
```

**Q2：Go 的接口和 Java/C# 的接口有什么不同？**

- Go 是**隐式接口**（implement by method，不需显式声明 implement）
- Go 接口是**小而专**的（通常 1-3 个方法），Java 接口倾向大而全
- Go 鼓励**面向接口编程**，但不要过度设计（YAGNI）

**Q3：如何判断一个模块该不该拆分？**

三个信号，满足任一就应该拆：
1. **代码量**：单文件 > 500 行，或单 package > 3000 行
2. **团队**：> 2 人同时修改同一个 package
3. **变更频率**：该模块变更频率明显高于其他模块

---

## 延伸阅读

- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Go Clean Architecture 实践 - 煎鱼博客](https://eddycjy.com/posts/go/clean-architecture/2020-02-09-go-clean-architecture/)
- [Dependency Injection in Go](https://quii.gitbook.io/learn-go-with-go-tests/dependency-injection)
- [Hexagonal Architecture in Go](https://medium.com/@matiasvarela/hexagonal-architecture-in-go-2dc3d3af05e8)
