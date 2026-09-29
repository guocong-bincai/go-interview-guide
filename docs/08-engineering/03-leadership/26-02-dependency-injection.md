# Go 依赖注入工程化：手工装配 / wire / fx / dig 怎么选

> 考察频率：★★★★☆  优先级：P1
> 关键词：依赖注入、DI、wire 代码生成、uber-go/fx、dig、Provider、生命周期、可测性

## 面试官考察意图

"你项目里 service 怎么 new 出来的？"——这是高级工程师面试里区分度很高的一问。小项目 `main` 里手工 `new` 没问题；但一个 20+ 依赖的中台服务，如果还在 `NewOrderService(NewUserRepo(db), NewCache(rdb), NewLogger(), ...)` 里层层手写，改动一个构造参数就要牵动一长串调用点，测试时也没法替换实现。

面试官想确认三件事：

1. 你**懂不懂 DI 的本质**（控制反转 = 谁负责组装、对象不自己去找依赖），而不是把它当成"框架魔法"；
2. 你**知不知道 Go 生态的三条路线**：手工装配 / 编译期代码生成（wire）/ 运行时反射容器（dig、fx），以及各自的代价；
3. 你**有没有生产经验**：循环依赖怎么破、启动顺序怎么控、优雅关闭怎么接、测试怎么替换。

**核心立场：DI 的目标是「可测 + 可换 + 装配集中」，不是「少写几行 new」。如果没有这些收益，手工装配就是最正确的选择。**

---

## 核心答案（30 秒版）

Go 的 DI 分三代：

| 方案 | 原理 | 优点 | 代价 |
|------|------|------|------|
| **手工装配** | `main` 里显式 new，构造函数注入 | 零依赖、可读、编译期可见 | 依赖一多装配代码冗长，改动牵连广 |
| **wire**（google/wire） | **编译期代码生成**，读 `wire.Build` 声明生成 `wire_gen.go` | 无反射、零运行时开销、错误在编译期 | 需要 `go generate`，循环依赖直接编译失败 |
| **fx / dig**（uber-go） | **运行时反射容器**，`fx.Provide` 注册、自动解析依赖图 | 生命周期 Hook、模块化、动态组装 | 反射有启动开销，错误推迟到运行时（`fx.ValidateApp` 可提前） |

**选型一句话**：中小项目 / 强确定性 → **手工装配**；追求零运行时开销 + 编译期保障 → **wire**；需要生命周期管理（OnStart/OnStop）+ 插件化模块 → **fx**。三者可以混用：`main` 手工搭骨架，业务用 wire/fx 组装。

**无论用哪种，"构造函数注入"永远优于"全局变量 / 包内单例"**——因为只有前者能测。

---

## 深度展开

### 一、先搞清楚 DI 到底解决什么

反例（反面教材，面试里千万别这么答）：

```go
// ❌ 依赖在函数体内部自己构造，无法替换、无法测试
func (s *OrderService) Create(o Order) error {
    db, _ := sql.Open("mysql", os.Getenv("DSN")) // 自己找依赖
    // ... 业务逻辑
    _ = db
    return nil
}
```

这带来三个致命问题：

1. **不可测**：单测时无法注入 mock DB，只能连真库；
2. **不可换**：想换 Redis 实现、加装饰器，要改所有调用点；
3. **隐式耦合**：依赖关系藏在函数体里，`main` 看不出一张完整的依赖图。

正例——**构造注入 + 面向接口**：

```go
type UserRepo interface {
    Get(ctx context.Context, id int64) (*User, error)
}

type OrderService struct {
    users  UserRepo   // 依赖接口，不依赖实现
    cache  Cache
    logger *slog.Logger
}

func NewOrderService(users UserRepo, cache Cache, logger *slog.Logger) *OrderService {
    return &OrderService{users: users, cache: cache, logger: logger}
}
```

这一步之后，`OrderService` 就可以对着 `UserRepo` 的 fake 做纯单测。**DI 的收益在这一行就已经兑现了，用不用框架是第二层问题。**

### 二、wire：编译期代码生成，零反射

`wire` 的思路是：你只声明"我要什么"（`wire.Build`），工具帮你生成"怎么造"（`wire_gen.go`），生成的是**普通的 Go 代码**，运行时没有任何魔法。

```go
// main.go —— 手写声明文件，带 build tag 或放在单独 inject.go
//go:build wireinject

package main

import "github.com/google/wire"

func InitApp(cfg *Config) (*App, error) {
    wire.Build(
        NewApp,        // *App
        NewOrderService,
        NewUserRepo,   // 提供 UserRepo 具体实现
        NewDB,         // 提供 *sql.DB
        NewRedisClient,// 提供 *redis.Client
        wire.Bind(new(Cache), new(*redisCache)), // 接口绑定到实现
    )
    return nil, nil // 生成后会被替换
}
```

```go
// Provider 就是返回依赖的构造函数
func NewUserRepo(db *sql.DB) *mysqlUserRepo { return &mysqlUserRepo{db: db} }
func NewDB(cfg *Config) (*sql.DB, error)    { return sql.Open("mysql", cfg.DSN) }
```

执行 `go generate ./...`（或 `wire ./...`）生成 `wire_gen.go`：

```go
func InitApp(cfg *Config) (*App, error) {
    db, err := NewDB(cfg)
    if err != nil { return nil, err }
    rdb := NewRedisClient(cfg)
    repo := NewUserRepo(db)
    svc := NewOrderService(repo, rdb, ...)
    return NewApp(svc), nil
}
```

wire 的关键特性：

- **依赖图在编译期解析**，少写一个 Provider 直接编译失败；**循环依赖直接报错**，不给运行期埋雷；
- **参数名 / 类型 / `wire.Bind` 决定注入谁**，类型相同可以靠 `wire.Struct`、`wire.Value`、`wire.InterfaceValue` 消歧；
- 生成文件提交进仓库，**CI 校验 `wire diff`**，避免有人改了声明没重新生成。

### 三、fx：运行时容器 + 生命周期

`fx` 建立在 `dig` 之上，主打**模块化 + 生命周期管理**，适合服务型应用：

```go
func main() {
    fx.New(
        fx.Provide(
            NewDB,
            NewRedisClient,
            NewUserRepo,
            NewOrderService,
            NewHTTPServer, // 返回 (*http.Server, error)
        ),
        fx.Invoke(RegisterRoutes), // 显式触发依赖，做副作用注册
        fx.Invoke(func(lc fx.Lifecycle, srv *http.Server) {
            lc.Append(fx.Hook{
                OnStart: func(ctx context.Context) error {
                    go srv.ListenAndServe() // 启动
                    return nil
                },
                OnStop: func(ctx context.Context) error {
                    return srv.Shutdown(ctx) // 优雅关闭，按注册逆序执行
                },
            })
        }),
    ).Run()
}
```

fx 的价值点：

- **`fx.Lifecycle` 管理启停顺序**：`OnStop` 按 `OnStart` 的**逆序**执行，天然符合"后启动的先关闭"；DB 连接池会自动在 service 停止后关闭；
- **`fx.Module` 分模块**：`fx.Module("order", fx.Provide(...), fx.Invoke(...))`，大项目按领域切分；
- **`fx.Annotate`** 给同一类型打标签（`fx.ResultTags`/`fx.ParamTags`），解决"两个 `*sql.DB`"的歧义；
- **`fx.ValidateApp()`** 在测试里预跑一次依赖图校验，把"少 Provide 一个"的错误提前到单测阶段。

### 四、dig：只要容器，不要生命周期

`dig` 是更底层的库，只做依赖解析，不管启停：

```go
c := dig.New()
_ = c.Provide(NewDB)
_ = c.Provide(NewOrderService)
err := c.Invoke(func(svc *OrderService) {
    // 使用 svc
})
```

适合"只想要自动装配、不需要 fx 那套 Hook/Module"的场景。`fx` 用户 99% 场景不需要直接用 `dig`。

### 五、选型矩阵（面试直接背这张表）

| 维度 | 手工装配 | wire | fx / dig |
|------|----------|------|----------|
| 运行时开销 | 无 | 无 | 有（反射 + 容器构建） |
| 错误发现时机 | 编译期 | **编译期** | 运行期（可 ValidateApp 提前） |
| 学习成本 | 最低 | 中（要理解 Bind/Provider） | 中高（生命周期/Hook/Module） |
| 生命周期管理 | 自己写 | 自己写 | **内置** |
| 循环依赖 | 自己发现 | **编译期报错** | 运行期报错 |
| 适合规模 | 小 / 中 | 中 / 大 | 中 / 大（服务型） |

**决策树**：

```
依赖数量 < 10 且无复杂启停 → 手工装配（别过度设计）
需要零运行时开销 + 编译期保障 → wire
需要 OnStart/OnStop / 模块化插件 → fx
```

### 六、五个生产级坑（面试的加分项）

1. **循环依赖**：`OrderService` 依赖 `PayService`，`PayService` 又依赖 `OrderService`。wire 直接编译失败。正确做法是**抽出第三个接口**（如 `OrderQuery`）打破环，或引入事件解耦——**不要用 `Set`/延迟注入去"绕过"环，那只是把设计问题藏起来**。
2. **全局变量 / init 里自建单例**：`var db, _ = sql.Open(...)` 会让 DI 失效、测试无法替换。**除纯常量外，禁止包级可变全局**。
3. **构造函数里做 IO**：`NewDB` 直接 `Ping` 还好（要 `PingContext` + 超时），但别在构造函数里跑迁移、预热大缓存——**容器构建期应该只做"接线"，副作用放到 `OnStart`/`Invoke`**。
4. **接口绑定搞错实现**：wire 里 `wire.Bind(new(Cache), new(*redisCache))`，`*redisCache` 必须实现 `Cache`，否则编译错；fx 里要靠 tag 消歧，否则解析出错误实现（运行时才炸）。
5. **测试替换**：DI 的终极收益在测试。wire/fx 都支持在测试里换 Provider：

```go
// wire: 在测试目录再写一个 InitTestApp，提供 fake 实现
// fx: 在测试里覆盖 Provide
app := fx.New(
    fx.Provide(NewFakeUserRepo), // 覆盖真实 repo
    fx.NopLogger,                // 测试里别刷日志
    fx.ValidateApp(),
)
```

---

## 高频追问

**Q1：DI 和"工厂模式""单例模式"是什么关系？**
DI 是**原则**（依赖由外部提供）；工厂是**创建对象的手段**；单例是**作用域**。DI 容器本质是"带依赖图的工厂"。Go 里 `sync.Once` + 包级变量是一种失控的单例，而 DI 容器通常默认单例（每次 `Invoke` 复用同一实例）。

**Q2：wire 生成的文件要不要提交？**
要。提交后 CI 才能校验一致性（改了声明没重新生成会 diff 出差异）。生成命令固定为 `go generate ./...`，并在 CI 里跑 `git diff --exit-code`。

**Q3：fx 启动慢怎么办？**
fx 反射构建容器在**毫秒级**，真正慢的是 `OnStart` 里的 IO（连 DB、拉配置）。排查方法：在 `OnStart` 里打点计时，找出耗时最长的 Hook；连接类依赖改为**惰性建连**（`sql.DB` 本身就是懒连接池）。

**Q4：一个类型要被注入多个实例怎么办？**
wire 用 `wire.Struct` 传入不同字段或 `wire.Value` 显式赋值；fx 用 `fx.Annotate` + `fx.ResultTags("name:\"primary\"")` / `fx.ParamTags`。

**Q5：不用框架，怎么保证手工装配不出错？**
把装配集中到一个 `NewApp(cfg)`，配合 `fx.ValidateApp` 思路做**编译期检查**（返回 `error` 链式构造），并为 `NewApp` 写一个"只构建不启动"的测试。规模再大就换 wire。

---

## 面试话术

> "我们项目用的是**构造函数注入 + 接口**，装配集中在 `main` 一个 `NewApp` 里。规模变大后引入了 **wire**，因为它编译期生成代码、零反射、循环依赖直接报错，比运行时容器更可靠。需要生命周期管理（优雅关闭、后台任务启停）的服务用 **fx**，靠 `fx.Lifecycle` 按逆序 Stop。我的原则是：**DI 的收益是可测和可换，如果没这两个需求，手工装配就别上框架。**"

---

## 延伸阅读

- [google/wire 官方文档](https://github.com/google/wire)
- [uber-go/fx 官方文档](https://github.com/uber-go/fx)
- [uber-go/dig](https://github.com/uber-go/dig)
- 本仓库：[Go 服务架构设计：分层、Clean Architecture、Hexagonal](08-06-code-architecture.md)
