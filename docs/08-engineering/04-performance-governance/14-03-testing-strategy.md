# 测试策略：单测、集成测试、契约测试

> 考察频率：★★★★☆  优先级：P1

## 面试官考察意图

考察候选人的**质量保障意识和测试体系设计能力**。

初级工程师：写单测只是为了覆盖率达标。
高级工程师：**设计适合团队的测试金字塔，知道什么该测、什么不该测、怎么让测试真正有效**。

面试官想看：你不只是能写测试，而是理解测试背后的工程哲学。

---

## 核心答案（30 秒版）

测试策略的核心是**分层测试 + 快速反馈**：

| 测试类型 | 占比 | 运行速度 | 覆盖范围 |
|---------|------|---------|---------|
| 单元测试 | 70% | ms 级 | 函数/方法逻辑 |
| 集成测试 | 20% | 秒级 | 模块间交互 |
| E2E 测试 | 10% | 分钟级 | 关键业务流程 |

**测试金字塔 vs 测试钻石：**
- 金字塔（推荐）：底层多、上层少，快速反馈
- 钻石（危险）：底层少、E2E 多，ci 运行极慢，反馈周期长

**一句话：** 好的测试策略是"让每一次代码提交都能快速知道有没有破坏核心功能"。

---

## 深度展开

### 1. 测试金字塔的正确理解

#### 金字塔各层职责

```
         ┌─────────┐
         │   E2E   │  ← 少量，测关键路径（登录、下单）
        ┌──────────┴──────────┐
        │     集成测试        │  ← 测模块间交互（Service + DB）
       ┌┴───────────────────┴─┐
       │      单元测试         │  ← 最多，测每个函数的边界条件
       └─────────────────────┘
```

#### 为什么要这样分层？

| 层 | 优点 | 缺点 | 适合测什么 |
|----|------|------|-----------|
| 单测 | 快、稳、定向 | 不覆盖跨模块 | 函数逻辑、边界条件、异常处理 |
| 集成 | 覆盖真实交互 | 慢、需要环境 | DB 查询、缓存交互、消息发送 |
| E2E | 最真实 | 极慢、不稳定 | 关键业务流程、跨系统链路 |

**真实案例：**
> "我之前带的项目 E2E 测试占了 60%，CI 运行要 40 分钟，团队怨声载道。后来我们重构了测试策略：把 50% 的 E2E 场景改成单测 + 集成测试，E2E 只保留 10 个核心场景，CI 压缩到 8 分钟。"

---

### 2. 单元测试、集成测试、端到端测试如何分层

#### 单元测试（Unit Test）

**Go 单测最佳实践：**

```go
// 原则：测试行为，不测试实现
// Bad: 测试内部变量
func TestAdd(t *testing.T) {
    s := &Stack{maxSize: 3}  // 暴露内部结构
    s.push(1)
    if s.top != 1 {  // 测试内部状态，不推荐
        t.Error()
    }
}

// Good: 测试公开行为
func TestStack_Push(t *testing.T) {
    s := NewStack(3)
    s.Push(1)
    got, err := s.Pop()
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if got != 1 {
        t.Errorf("got %d, want 1", got)
    }
}
```

**单测的 3A 原则：**
```go
func TestAdd(t *testing.T) {
    // Arrange：准备数据
    a, b := 2, 3
    
    // Act：执行被测操作
    result := Add(a, b)
    
    // Assert：验证结果
    if result != 5 {
        t.Errorf("Add(%d, %d) = %d, want 5", a, b, result)
    }
}
```

**单测覆盖哪些场景：**
- 正常路径
- 边界条件（nil、空数组、0、最大值）
- 错误路径（传入非法参数、网络超时）
- 并发安全（如果是并发模块）

**单测不该测什么：**
- 第三方库的实现细节（用 fake/mock）
- 简单的 getter/setter（没有业务逻辑的）

#### 集成测试（Integration Test）

**Go 集成测试的常用方法：**

```go
// 使用 testcontainers 测真实的数据库
func TestUserRepository_Create(t *testing.T) {
    // 启动真实的 MySQL 容器
    mysql, err := tc.GenericContainer(ctx, tc.ContainerRequest{
        Image: "mysql:8",
        ...
    })
    defer mysql.Terminate(ctx)
    
    repo := NewUserRepository(db) // 真实 DB 连接
    user := &User{Name: "test"}
    
    err = repo.Create(ctx, user)
    // 真实写入，验证 SQL 是否正确
}

// 使用 mock 测外部依赖
func TestOrderService_PlaceOrder(t *testing.T) {
    mockRedis := &RedisMock{GetFn: func(key string) (string, error) {
        return "100", nil  // 固定返回
    }}
    svc := NewOrderService(mockRedis)
    
    // 不依赖真实 Redis
}
```

**集成测试的覆盖重点：**
- Repository 层（SQL 正确性）
- Service 层（业务逻辑 + 多模块交互）
- HTTP Handler（路由、中间件、请求解析）

#### 端到端测试（E2E）

**E2E 测试的使用原则：**
1. **只覆盖核心业务流程**（登录、支付、下单）
2. **CI/CD 中用，但不阻断日常开发**
3. **用真实环境（或接近真实）**

```go
// Go E2E 测试示例（使用 gocypress 或自行封装）
func TestUserLogin_E2E(t *testing.T) {
    // 启动完整服务
    app := app.NewApp()
    go app.Run()
    defer app.Close()
    
    // 调真实 HTTP 接口
    resp, err := http.Post("http://localhost:8080/login", 
        "application/json", 
        strings.NewReader(`{"username":"test","password":"123"}`))
    
    if resp.StatusCode != 200 {
        t.Errorf("login failed: %s", resp.Status)
    }
}
```

**E2E 的常见问题：**
- 慢（分钟级），不能加到每次 CI
- 不稳定（flaky），网络、环境都会影响
- 难以定位问题（不知道是哪一层出错）

**解法：E2E 测试放在每日 CI 或发布前 CI，而不是每次 PR CI**

---

### 3. 契约测试适合什么场景

#### 什么是契约测试

**契约测试（Contract Testing）** 是 Consumer-Driven Contract（CDC）的实践：

```
Consumer（消费者）：我需要什么数据格式
Provider（提供者）：我能提供什么数据格式
契约测试：验证 Provider 提供的接口是否满足 Consumer 的需求
```

**典型场景：微服务间接口变更**

> "A 服务调用 B 服务的 /user 接口，B 服务要改 /user 的返回字段。如果没有契约测试，A 不知道 B 改了，可能会悄悄上线后才发现数据格式不对。"

#### Go 契约测试工具

**Pact（推荐）：**
```go
// Consumer 端定义契约
func TestUserService_ConsumesUserAPI(t *testing.T) {
    mock := pact.VerifyMockProvider(&MockProvider{
        "/users/1": map[string]interface{}{
            "name": "test",
            "age":  25,
        },
    })
    
    user, err := userService.GetUser(1)
    // 验证 GetUser 能正确处理返回的数据
}

// Provider 端验证契约
func TestUserAPI_Provider(t *testing.T) {
    pact.VerifyProvider(t, &ProviderVerification{
        ProviderBaseURL: "http://user-service:8080",
        PactFiles:       []string{"./pacts/user-service.json"},
    })
}
```

**契约测试的使用建议：**

| 适用 ✅ | 不适用 ❌ |
|--------|---------|
| 微服务间接口 | 单体应用内部 |
| 接口变更频繁的服务 | 稳定的内部接口 |
| 多 Consumer 的共享服务 | 只有单 Consumer 的服务 |

---

### 4. 如何保证重构安全

#### 测试是重构的保险

**重构的铁律：** 没有测试覆盖的重构 = 裸奔

```
重构前：代码有测试
重构后：测试全部通过
→ 重构是安全的
```

**Case Study：老系统重构如何加测试**

Step 1：用 E2E 测试覆盖核心流程（不改代码，先加测试）
Step 2：跑通 E2E，确认覆盖了核心功能
Step 3：逐步把 E2E 拆成单元 + 集成测试（逐层替换）
Step 4：重构时，单元测试保证每个模块的行为不变
Step 5：E2E 验证整体业务流程不变

#### 黄金路径

```
测试先行（TDD）→ 重构时测试保证安全 → CI 防止回归
```

**推荐的重构流程：**

1. **加测试（不改业务逻辑）：**
   > "先写测试，确保当前行为被测试覆盖。不要改代码，只加测试。"

2. **小步重构 + 频繁运行测试：**
   > "每次只改一个很小的地方，运行测试，确认通过后再改下一个。"

3. **CI 门禁：**
   > "PR 必须所有测试通过才能合并，包括单测 + 集成测试。"

---

### 5. Mock 过多的坏处

#### Mock 地狱

```go
// Mock 层层嵌套，测试变成了"Mock 配置"测试
func TestOrderService(t *testing.T) {
    mockRedis := NewMockRedis()
    mockDB := NewMockDB()
    mockKafka := NewMockKafka()
    mockConfig := NewMockConfig()
    
    // 这些 Mock 的配置对不对？谁知道
    svc := NewOrderService(mockRedis, mockDB, mockKafka, mockConfig)
}
```

**Mock 地狱的症状：**
- 测试代码比业务代码还长
- 改一个接口，所有 Mock 都要改
- Mock 配置错误导致测试假阳性/假阴性

#### 解法：Mock 最小化原则

| 原则 | 说明 |
|------|------|
| 只 Mock 外部依赖（DB、外部 API）| 内部模块之间不要 Mock |
| Mock 接口，不 Mock 实现 | 依赖接口而非具体类型 |
| 优先用真实对象 + 测试数据库 | 比 Mock 更可靠 |
| Mock 层超过 3 层 → 考虑集成测试 | 说明模块划分有问题 |

**最佳实践：用测试数据库替代 Mock**

```go
// Bad: Mock 数据库
func TestWithMock(t *testing.T) {
    mockDB := &MockDB{QueryFn: func(sql string) {...}}
    repo := NewUserRepo(mockDB)
}

// Good: 用真实测试数据库（testcontainers）
func TestWithRealDB(t *testing.T) {
    mysql := startTestMySQL(t)  // 真实 MySQL
    defer mysql.Close()
    
    repo := NewUserRepo(mysql.Conn())  // 真实连接
    // 测试真实 SQL 是否正确
}
```

---

### 6. 高级工程师如何推动测试体系落地

#### 现实问题：团队不爱写测试

**常见阻力：**
- "项目紧，没时间写测试"
- "测试是质量保障的事，不是开发的事"
- "写了测试，CI 还是跑不过，没人管"

**推动策略：**

**① 用覆盖率做参考，不要做门禁**

> "我们不强制覆盖率，但覆盖率低于 30% 的模块不允许发版（容易出线上问题）。"
> → 不是考核工具，是风险提示

**② 从关键路径开始**

> "不要求大家把所有代码都写单测，但从今天开始，所有新写的 Service 层必须有单测。"

**③ 单测写得好有奖励**

> "代码 review 时，如果单测写得好，我会点名表扬。连续 3 个月单测质量好的同学，给他机会做架构设计。"

**④ 降低写测试的门槛**

- 提供单测模板（3A 原则、常见场景覆盖）
- 提供 Mock 工具库（不要让每个人自己造轮子）
- 提供测试最佳实践文档

**⑤ CI 中展示测试趋势**

```
测试覆盖率趋势图：
50% ████████████
40% ██████████
30% ████████
20% ██████
10% ████
    -----
    Month
```
→ 让团队看到进步，比强制要求更有效

---

## 高频追问

### Q1：单测覆盖率越高越好吗？

**参考答案：**
不是。覆盖率是手段，不是目的。

> "覆盖率反映的是'测了哪些代码'，而不是'测得有多好'。100% 覆盖率的测试可以全是无效测试。
> 
> 更重要的是：
> 1. 核心业务逻辑有没有被测到（尤其是边界条件和错误处理）
> 2. 测试是否稳定（flaky test 比没测试更糟糕）
> 3. 测试是否可维护（改代码时是否要大量改测试）"

### Q2：已经上线的项目怎么补测试？

**参考答案：**
不要想着"一次性补完"，而是"增量补充"：

1. **新增代码必须有测试**（不再积累技术债）
2. **出线上问题后，先补测试再修 bug**（防止复现）
3. **重构时补测试**（为重构买保险）
4. **核心模块优先补**（用风险评估决定优先级）

### Q3：如何平衡测试质量和开发速度？

**参考答案：**
- 核心业务流程：**测试优先**（写完代码先写测试再提交）
- 辅助模块：可以降低单测要求，但必须有基本覆盖
- 临时脚本/一次性的代码：不强制测试

> "不是所有代码都需要同等质量的测试。根据业务影响和变更频率，决定测试投入。"

### Q4：契约测试和端到端测试的区别是什么？

**参考答案：**

| | 契约测试 | 端到端测试 |
|--|---------|-----------|
| 范围 | 单个接口的请求/响应格式 | 整个业务流程 |
| 速度 | 快（秒级）| 慢（分钟级）|
| 稳定性 | 高 | 低 |
| 定位 | Consumer 和 Provider 之间的接口契约 | 整体业务流程是否正常 |
| 谁来做 | Consumer + Provider 共同维护 | E2E 测试团队 |

---

## 延伸阅读

- [测试金字塔详解](https://martinfowler.com/articles/practical-test-pyramid.html)
- [Google Testing Blog](https://testing.googleblog.com/)
- [Go Testing Best Practices](https://ieftimov.com/posts/testing-in-go/)
- [Pact 契约测试 Go 实践](https://docs.pact.io/)
