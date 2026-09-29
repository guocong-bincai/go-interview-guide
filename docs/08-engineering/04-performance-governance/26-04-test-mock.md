# 单元测试 Mock 工程化：gomock / mockgen 与可测性设计

> 考察频率：★★★★☆  优先级：P1
> 关键词：gomock、mockgen、testify、测试替身、fake vs mock、接口设计、可测性

## 面试官考察意图

"你这个函数怎么测的？"——如果候选人回答"数据不好造，所以没测"或"我 mock 了 8 个依赖"，都不算好答案。面试官想透过 Mock 看三件事：

1. 你**分得清测试替身**（Dummy / Stub / Fake / Spy / Mock）的用途，而不是无脑 mock；
2. 你**知道 Mock 的成本**：Mock 越多，测试越脆（和实现耦合），最后测试变成"验证你有没有按我猜的顺序调用依赖"；
3. 你**能从设计层面提高可测性**：接口切分、依赖注入、时间/随机可注入——而不是靠打桩掩盖设计问题。

**核心立场：能用一个 Fake 就别用一堆 Mock；能用真实内存实现（如内存版 Repository）就别 mock 框架。Mock 是工具，不是目标。**

---

## 核心答案（30 秒版）

五种测试替身（Martin Fowler 分类）：

| 替身 | 作用 | Go 里的写法 |
|------|------|-------------|
| **Dummy** | 只为填参数，不被使用 | 传 `nil` / 空 struct |
| **Stub** | 返回预设值 | 手写 func 字段的 struct |
| **Fake** | 有真实逻辑的简化实现 | 内存版 repo、map 版 cache |
| **Spy** | 记录调用，供断言 | 记录调用参数的 struct |
| **Mock** | 预设期望 + 校验交互 | `gomock` / `mockgen` |

**选型优先级**：`Fake > Stub/Spy > Mock`。**Mock 只在"必须验证副作用是否发生"时用**（如"支付成功后必须调了一次发券接口"）。

Go 生态必备：

- **`testify/assert` / `require`**：断言库。`require` 失败即 `t.FailNow`，后续不再执行（适合前置条件）；`assert` 失败继续（适合批量断言）；
- **`gomock` + `mockgen`**：基于接口生成 mock；
- **`httptest`**：测 HTTP 依赖；**`testcontainers-go`**：起真实 DB/Redis，做集成测试。

**可测性的三条设计原则**：

1. **依赖接口，不依赖具体类型**（这样能替换）；
2. **副作用通过构造函数注入**（时间、随机数、ID 生成器）；
3. **业务逻辑与 IO 分离**（纯函数最好测）。

---

## 深度展开

### 一、手写 Fake 优先：一个例子说明为什么

假设要测订单服务里的"库存不足要报错"：

```go
type InventoryRepo interface {
    Deduct(ctx context.Context, sku string, n int) error
}

// ✅ Fake：用 map 实现真实逻辑，几乎零成本
type fakeInventory struct {
    stock map[string]int
}

func (f *fakeInventory) Deduct(_ context.Context, sku string, n int) error {
    if f.stock[sku] < n {
        return ErrInsufficientStock
    }
    f.stock[sku] -= n
    return nil
}

func TestCreateOrder_InsufficientStock(t *testing.T) {
    inv := &fakeInventory{stock: map[string]int{"A": 1}}
    svc := NewOrderService(inv)
    _, err := svc.Create(context.Background(), Order{SKU: "A", Qty: 5})
    require.ErrorIs(t, err, ErrInsufficientStock) // 断言业务结果
}
```

对比 gomock 版本：

```go
// ⚠️ Mock：断言的是"调用行为"，不是业务结果
ctrl := gomock.NewController(t)
defer ctrl.Finish()
inv := NewMockInventoryRepo(ctrl)
inv.EXPECT().Deduct(gomock.Any(), "A", 5).Return(ErrInsufficientStock)

_, err := svc.Create(context.Background(), Order{SKU: "A", Qty: 5})
require.ErrorIs(t, err, ErrInsufficientStock)
```

两者都能测，但 **Fake 版本重构友好**：如果 `Deduct` 改签名或改成"预留-提交"两步，Fake 只需要改一处，Mock 则要改所有 `EXPECT()`。**这就是"过度 mock 导致测试脆"的本质**。

### 二、gomock / mockgen 的正确打开方式

**生成方式**（推荐 `source` 模式，无需编译）：

```bash
#go:generate mockgen -source=repo.go -destination=mock_repo.go -package=mocks
```

```go
// repo.go
//go:generate mockgen -source=repo.go -destination=mock_repo.go -package=mocks
type UserRepo interface {
    Get(ctx context.Context, id int64) (*User, error)
    Save(ctx context.Context, u *User) error
}
```

**Go 1.24+ 的 `go tool` 方式**（避免 `go.mod` 里带工具依赖）：

```bash
go get -tool go.uber.org/mock/mockgen
go tool mockgen -source=repo.go -destination=mock_repo.go -package=mocks
```

**常用匹配器**：

```go
inv.EXPECT().Get(gomock.Any(), int64(42)).Return(&User{ID: 42}, nil)
inv.EXPECT().Save(gomock.Any(), gomock.Any()).DoAndReturn(
    func(_ context.Context, u *User) error {
        require.Equal(t, "bob", u.Name) // 在 Do 里做参数校验
        return nil
    })
inv.EXPECT().Get(gomock.Any(), int64(99)).Return(nil, ErrNotFound).Times(1)
```

⚠️ **gomock 最大的坑**：`EXPECT().Times(1)` 默认是"恰好一次"。**过度使用会导致测试与实现强耦合**——重构把两次调用合并成一次，测试就红。**能用 `AnyTimes()` 就用，只对"必须发生"的副作用做精确断言。**

### 三、什么必须 mock / 什么千万别 mock

**必须替换（不可控 / 慢 / 有副作用）**：

- 网络：外部 HTTP API（用 `httptest.Server` 比 mock 客户端更好）；
- DB / Redis：单测用 Fake，集成测试用 `testcontainers`；
- 时间：`time.Now()`、`time.Sleep` —— 注入 `Clock` 接口；
- 随机 / UUID：注入 `func() string`；
- 消息队列 / 文件系统。

**千万别 mock 的**：

- 自己写的纯业务逻辑（直接 new 出来测）；
- 标准库（`sort`、`strings` —— 直接测）；
- 同包内的私有函数（间接通过公开入口测）。

### 四、可测性设计：把"难测"变成"好测"

**技巧 1：注入时钟，消灭 `time.Sleep`**

```go
type Clock interface{ Now() time.Time }
type RealClock struct{}
func (RealClock) Now() time.Time { return time.Now() }

// 生产
svc := NewCouponService(RealClock{})
// 测试：可精确控制时间前进，测试从秒级降到毫秒级
type fakeClock struct{ t time.Time }
func (f *fakeClock) Now() time.Time { return f.t }
```

**技巧 2：接口按"使用方需要"定义（Interface Segregation）**

```go
// ❌ 一个巨大的 Repo 接口，mock 要生成几十个方法
type Repo interface { Get; Save; Delete; List; Count; ... }

// ✅ 使用方声明自己需要的最小接口（Go 的鸭子类型让这很自然）
type userGetter interface{ Get(ctx context.Context, id int64) (*User, error) }
```

**技巧 3：表驱动 + 子测试 + 并行**

```go
func TestDeduct(t *testing.T) {
    tests := []struct {
        name    string
        stock   int
        n       int
        wantErr error
    }{
        {"enough", 10, 3, nil},
        {"exact", 3, 3, nil},
        {"insufficient", 1, 5, ErrInsufficientStock},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            inv := &fakeInventory{stock: map[string]int{"A": tt.stock}}
            err := inv.Deduct(context.Background(), "A", tt.n)
            require.ErrorIs(t, err, tt.wantErr)
        })
    }
}
```

⚠️ `t.Parallel()` 要求所有子测试**互不共享可变状态**——Fake 必须每个 `t.Run` 里新建。

### 五、Mock 的边界：什么时候该转向集成测试

**单测（Fake/Mock）** 保证：业务分支、边界条件、错误处理。
**集成测试（testcontainers）** 保证：SQL 真的能跑、序列化真的对、事务真的回滚。

```go
func TestUserRepo_Integration(t *testing.T) {
    if testing.Short() { t.Skip("skip integration") }
    ctx := context.Background()
    pg, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image: "postgres:16-alpine",
            ExposedPorts: []string{"5432/tcp"},
        },
        Started: true,
    })
    require.NoError(t, err)
    defer pg.Terminate(ctx)
    // ... 跑真实 SQL
}
```

**面试表达**：**"逻辑用 Fake 快测，持久层用 testcontainers 慢测，两者的比例大概是测试金字塔的 70/20/10。"**

### 六、常见坑

1. **`controller.Finish()` 忘了调**：Go 1.14+ 用 `gomock.NewController(t)` 会自动注册 `t.Cleanup`，**不需要再手动 `defer ctrl.Finish()`**（旧教程还在教，属过时写法）。
2. **Mock 掉不该 mock 的东西**：mock 了 `sort.Ints`，测试变成"验证我调用了 sort"——没有任何价值。
3. **断言交互顺序**：用 `gomock.InOrder` 会强绑定实现顺序，**重构即红**。只在顺序**语义上重要**时才用。
4. **Fake 和真实实现行为漂移**：Fake 实现了简化逻辑，时间久了和真实 DB 行为不一致，单测绿、集成红。**Fake 要尽量保持与真实实现的语义一致，并靠集成测试兜底。**
5. **测试共享可变状态**：包级 `var repo = &fakeInventory{}` 被多个测试改 → 偶发失败（Flaky）。每个测试独立构造。

---

## 高频追问

**Q1：Mock 和 Stub 的区别？**
Stub **只提供返回值**，不关心调用；Mock **预设期望并校验交互**（调了没、调了几次、参数对不对）。过度用 Mock 会测"实现细节"，Stub/Fake 测"行为结果"。

**Q2：为什么不用 monkey patch 打桩私有函数？**
`monkey`/`gomonkey` 依赖运行时内存改写，**在 Go 里既不稳定（编译器内联优化会让打桩失效）又脆弱**，还会破坏 `-race`。**正确做法是通过接口注入依赖，而不是打桩。**

**Q3：覆盖率 100% 就代表测试好吗？**
不代表。覆盖率只说明"代码被执行过"，不说明"断言正确"。**覆盖率盲区是"分支未覆盖"而不是"行未覆盖"**，应关注分支覆盖率 + 变异测试（如 `go-mutesting`）来评估测试质量。

**Q4：`httptest.Server` 和 mock HTTP client 哪个好？**
`httptest.Server` 更好——它测的是**真实的 HTTP 交互**（序列化、header、状态码、超时），而 mock client 只是验证"我调了它"。

**Q5：如何测并发安全代码？**
单测 + `-race`：`go test -race -count=10 ./...`。再配合 `testing/synctest`（Go 1.25 标准库）做虚拟时钟并发测试，见 [Flaky Test 治理](../04-performance-governance/26-03-flaky-test.md)。

---

## 面试话术

> "我的测试策略是**优先 Fake、谨慎 Mock**：持久层用内存 Fake，外部 HTTP 用 `httptest.Server`，端到端用 testcontainers。只有需要验证副作用（比如支付成功必须发券）时才上 **gomock**，而且尽量 `AnyTimes()`，避免和实现顺序强耦合。可测性靠设计保证：依赖接口、注入时钟和 ID 生成器、业务逻辑与 IO 分离——把 `time.Now()` 抽成 Clock 接口后，涉及时间的测试从秒级降到毫秒级。**我的经验是：一个测试里 mock 超过 3 个依赖，通常说明这个函数职责太重，该拆了。**"

---

## 延伸阅读

- [uber-go/mock (gomock 官方维护版)](https://github.com/uber-go/mock)
- [testify](https://github.com/stretchr/testify)
- [testcontainers-go](https://golang.testcontainers.org/)
- [Martin Fowler: Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
- 本仓库：[测试策略：单测、集成测试、契约测试](../04-performance-governance/14-03-testing-strategy.md)
