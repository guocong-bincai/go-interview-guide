# Seata 分布式事务实战

> 考察频率：★★★☆☆  难度：★★★★☆
> 重点：理解 AT/TCC/Saga 三种模式的适用场景，能在面试中讲清楚选型依据

## 概述

Seata（Simple Extensible Autonomous Transaction Architecture）是阿里巴巴开源的分布式事务解决方案，提供 AT、TCC、Saga、XA 四种事务模式。

### 四种模式对比

| 模式 | 隔离级别 | 性能影响 | 侵入性 | 适用场景 |
|------|---------|---------|--------|----------|
| **AT** | 读已提交 | 低（本地事务） | 低（自动回滚） | Oracle/MySQL，强一致性场景 |
| **TCC** | 业务保证 | 中（多阶段） | 高（写三个方法） | 跨服务、跨库，强一致性 |
| **Saga** | 最终一致 | 高（长事务） | 中 | 长链路、多服务，最终一致 |
| **XA** | 强一致 | 低（2PC阻塞） | 低 | 跨数据库，强一致性 |

---

## AT 模式（Automatic Transaction）

### 核心原理

AT 模式基于 **本地事务 + 全局锁** 实现，对业务零侵入。

```
TM（事务管理器）发起全局事务
        │
        ▼
    TC（协调者）生成 XID
        │
        ▼
各分支事务执行 + 记录 undo_log
        │
        ▼
    TC 注册分支，锁定资源
        │
        ▼
全局提交 → 异步删除 undo_log
全局回滚 → 根据 undo_log 逆向生成补偿 SQL
```

### undolog 表结构

```sql
CREATE TABLE `undo_log` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `branch_id` bigint NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longtext NOT NULL,  -- 存储前置镜像
  `log_status` int NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1;
```

### AT 模式执行流程

```go
// AT 模式业务代码示例（对业务无侵入）
func TransferAccounts(from, to string, amount float64) error {
    // Seata DataSource Proxy 自动处理
    _, err := db.Exec("UPDATE account SET balance = balance - ? WHERE id = ?", amount, from)
    if err != nil {
        return err
    }
    _, err = db.Exec("UPDATE account SET balance = balance + ? WHERE id = ?", amount, to)
    return err
}
// Seata 自动生成 undo_log，自动回滚
```

### AT 模式优缺点

**优点：**
- 对业务代码零侵入
- 自动生成回滚 SQL，执行效率高
- 支持复杂 SQL（UPDATE、批量操作）

**缺点：**
- 需要支持本地事务的数据库（MySQL/PostgreSQL/Oracle）
- 全局锁降低并发性能
- 不支持跨语言

---

## TCC 模式（Try-Confirm-Cancel）

### 三阶段设计

```
Try  →  资源预留/锁定
Confirm → 确认执行（Try 成功后才调用）
Cancel → 资源释放（Try 失败或超时）
```

### Go 实现示例

```go
type AccountTCCService struct{}

// Try：预留资源，冻结金额
func (s *AccountTCCService) Try(ctx context.Context, req *TryRequest) (bool, error) {
    // 冻结金额：update set frozen = frozen + amount where balance - frozen >= amount
    affected, err := db.ExecContext(ctx,
        "UPDATE account SET frozen = frozen + ? WHERE id = ? AND balance >= ?",
        req.Amount, req.AccountID, req.Amount)
    if err != nil {
        return false, err
    }
    rows, _ := affected.RowsAffected()
    return rows > 0, nil  // 余额不足返回 false
}

// Confirm：真正扣款
func (s *AccountTCCService) Confirm(ctx context.Context, req *TryRequest) (bool, error) {
    // 冻结金额转实际扣款：update set frozen = frozen - amount, balance = balance - amount
    _, err := db.ExecContext(ctx,
        "UPDATE account SET frozen = frozen - ?, balance = balance - ? WHERE id = ?",
        req.Amount, req.Amount, req.AccountID)
    return err == nil, err
}

// Cancel：释放冻结
func (s *AccountTCCService) Cancel(ctx context.Context, req *TryRequest) (bool, error) {
    // 释放冻结：update set frozen = frozen - amount
    _, err := db.ExecContext(ctx,
        "UPDATE account SET frozen = frozen - ? WHERE id = ?",
        req.Amount, req.AccountID)
    return err == nil, err
}
```

### TCC 空回滚与悬挂

**空回滚：** Try 超时未执行，但 Cancel 被调用
```go
func (s *AccountTCCService) Cancel(ctx context.Context, req *TryRequest) (bool, error) {
    // 检查是否执行过 Try
    var frozen float64
    db.QueryRowContext(ctx, "SELECT frozen FROM account WHERE id = ?", req.AccountID).Scan(&frozen)
    if frozen <= 0 {
        return true, nil  // 空回滚，直接返回成功
    }
    // 正常取消...
}
```

**悬挂：** Cancel 先执行，Try 后到
```go
// 解决方案：记录事务状态
type TCCState struct {
    XID        string
    AccountID  string
    State      int  // 0=none, 1=tryed, 2=confirmed, 3=cancelled
}
// Try 执行前检查状态
```

### TCC 与 AT 对比

| 维度 | TCC | AT |
|------|-----|-----|
| 侵入性 | 高（需写 Try/Confirm/Cancel） | 低（自动处理） |
| 性能 | 较高（资源锁定时间短） | 较低（全局锁） |
| 适用数据库 | 任何数据库 | 支持本地事务的数据库 |
| 并发度 | 高（应用层控制锁） | 较低（Seata 全局锁） |
| 灵活性 | 高（业务自定义） | 低（SQL 解析） |

---

## Saga 模式

### 适用场景

长链路、多服务协作的业务流程，如：
- 订单 → 库存 → 支付 → 物流（4 个服务调用）
- 金融交易（不支持回滚的业务）

### 两种实现方式

**编排型（Choreography）：** 各服务监听事件自协调
```
OrderService → [创建订单] → 发送 order.created
    ↓
InventoryService → [扣库存] → 发送 inventory.reserved
    ↓
PaymentService → [扣款] → 发送 payment.completed
    ↓
ShippingService → [发货] → 发送 shipping.delivered
```

**控制型（Orchestration）：** 单一编排器控制全局
```
SagaOrchestrator
    ├── 调用 OrderService.create()
    ├── 调用 InventoryService.reserve()
    ├── 调用 PaymentService.charge()
    └── 调用 ShippingService.ship()
    如果失败 → 调用各服务的 compensate() 逆向补偿
```

### Go 编排器实现

```go
type OrderSaga struct {
    orderSvc     *OrderService
    inventorySvc *InventoryService
    paymentSvc   *PaymentService
}

type SagaStep struct {
    Name        string
    Execute     func(ctx context.Context, req interface{}) error
    Compensate  func(ctx context.Context, req interface{}) error
}

func (s *OrderSaga) ExecuteOrder(ctx context.Context, order *Order) error {
    steps := []SagaStep{
        {
            Name: "createOrder",
            Execute: func(ctx context.Context, req interface{}) error {
                return s.orderSvc.Create(ctx, order)
            },
            Compensate: func(ctx context.Context, req interface{}) error {
                return s.orderSvc.Cancel(ctx, order.ID)
            },
        },
        {
            Name: "reserveInventory",
            Execute: func(ctx context.Context, req interface{}) error {
                return s.inventorySvc.Reserve(ctx, order.Items)
            },
            Compensate: func(ctx context.Context, req interface{}) error {
                return s.inventorySvc.Release(ctx, order.Items)
            },
        },
        {
            Name: "chargePayment",
            Execute: func(ctx context.Context, req interface{}) error {
                return s.paymentSvc.Charge(ctx, order.UserID, order.Amount)
            },
            Compensate: func(ctx context.Context, req interface{}) error {
                return s.paymentSvc.Refund(ctx, order.UserID, order.Amount)
            },
        },
    }

    // 正向执行，失败则逆向补偿
    executed := []int{}
    for i, step := range steps {
        if err := step.Execute(ctx, order); err != nil {
            // 回滚已执行的步骤
            for j := len(executed) - 1; j >= 0; j-- {
                steps[executed[j]].Compensate(ctx, order)
            }
            return fmt.Errorf("saga failed at step %d: %w", i, err)
        }
        executed = append(executed, i)
    }
    return nil
}
```

---

## Seata 配置与部署

### AT 模式配置（application.yml）

```yaml
seata:
  enabled: true
  application-id: business-app
  tx-service-group: my_test_tx_group
  vgroup-mapping:
    my_test_tx_group: default
  config:
    type: nacos
    nacos:
      server-addr: nacos-server:8848
      namespace: dev
      group: SEATA_GROUP
  registry:
    type: nacos
    nacos:
      server-addr: nacos-server:8848
      application: seata-server
```

### 全局事务注解

```go
// TM（事务发起者）
@GlobalTransactional(name = "createOrder", rollbackFor = Exception.class)
func CreateOrderHandler(ctx context.Context, req *OrderReq) (*OrderResp, error) {
    // 所有参与的 RM 都在这个方法内
    order := orderSvc.Create(ctx, req)
    inventorySvc.Deduct(ctx, order.Items)   // Seata 自动注册为 RM
    paymentSvc.Charge(ctx, order)           // Seata 自动注册为 RM
    return order, nil
}
```

### RM 端配置（数据源代理）

```go
// AT 模式只需配置数据源代理，Seata 自动处理
func InitSeata() {
    // 使用 Seata DataSource Proxy 包装原生数据源
    db, _ := gorm.Open(mysql.Open("root:password@tcp(db:3306)/test"))
    // Seata 自动拦截 SQL，记录 undo_log
}
```

---

## 常见面试问题

### Q：Seata 和 ShardingSphere 的区别？

> ShardingSphere 主要做**数据分库分表**，解决数据库水平扩展问题；Seata 做**分布式事务**，解决跨库操作的一致性问题。两者可结合使用：ShardingSphere 管分片，Seata 管事务。

### Q：AT 模式的全局锁和本地锁有什么区别？

> 本地锁只能保证单库并发安全；Seata 全局锁在 TC（协调者）层面控制，确保跨库操作的序列化。当全局锁被其他事务占用时，当前事务会等待，不会脏读/脏写。

### Q：TCC 的 Confirm/Cancel 失败了怎么办？

> Seata 会不断重试（默认 10 次），间隔指数增长。如果最终失败，会记录状态，人工介入处理。这是 TCC 的"最大努力通知"策略。

### Q：Saga 和 TCC 的区别？

> TCC 是"资源预留"模式，Try 阶段冻结资源，Confirm/Cancel 明确释放；Saga 没有预留，直接正向执行，失败后逆向补偿。Saga 更适合长链路、低并发、无法冻结资源的场景（如外部支付系统）。

### Q：分布式事务的选型原则？

> **强一致选 AT 或 XA**，但要接受性能损耗；**跨服务/跨库选 TCC**，需要业务配合写补偿逻辑；**长链路/多服务选 Saga**，最终一致即可；**纯业务协调选本地消息表 + 定时任务**。
