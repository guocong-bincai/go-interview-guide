[🏠 首页](../../../README.md) · [📦 工程素养](../README.md)

---

# Code Review 规范：什么值得 Block，什么只是建议

## 面试官考察意图

考察候选人的工程素养和团队协作能力。
高级工程师不仅写出正确代码，还要能**在 Code Review 中有效发现风险、传授知识、维护代码一致性**，理解 Block 和 Suggestion 的边界，以及如何在 Review 中保持建设性沟通。

---

## 核心答案（30 秒版）

Code Review 的核心原则：**安全 > 正确 > 性能 > 可维护性 > 风格**。

- **必须 Block**：安全漏洞、正确性 bug、数据损坏风险、严重性能退化
- **建议类（Comment）**：命名优化、重复代码提取、注释补充、测试覆盖不足
- **Reviewer 心态**：对事不对人，给方案而非只提问题，重大变更先设计后代码

---

## 深度展开

### 1. Block 标准：必须纠正的问题

#### 1.1 安全风险（P0 — 立即修复）

```go
// ❌ 典型 SQL 注入（Block）
func getUser(query string) *User {
    sql := "SELECT * FROM users WHERE name = '" + query + "'"
    db.Query(sql)
}

// ✅ 正确做法：参数化查询
func getUser(query string) *User {
    sql := "SELECT * FROM users WHERE name = ?"
    db.Query(sql, query)
}

// ❌ 硬编码密钥（Block）
apiKey := "sk-abc123xxxx" // 必须从环境变量或 Secret 获取

// ❌ 敏感信息日志（Block）
log.Printf("用户登录失败: password=%s", password) // 密码泄露

// ❌ 权限检查缺失（Block）
func deleteOrder(ctx context.Context, orderID string) error {
    // ❌ 没有检查当前用户是否有权限删除
    return db.DeleteOrder(orderID)
}
```

**典型安全 Block 项：**
- SQL 注入、XSS、命令注入
- 硬编码密钥/Token/Secret
- 敏感数据写入日志（密码、Token、PII）
- 缺少权限校验（接口越权访问）
- 整数溢出可能导致的安全问题（Crypto、支付相关）
- 依赖已知 CVE 漏洞库（`go mod tidy` + `govulncheck`）

#### 1.2 正确性 / 数据损坏（P0 — 立即修复）

```go
// ❌ 并发 map 读写（运行时 panic）（Block）
var m = make(map[string]int)
// goroutine A
go func() { m["a"] = 1 }()
// goroutine B
go func() { fmt.Println(m["a"]) }()

// ✅ 使用 sync.RWMutex 或 sync.Map
var mu sync.RWMutex
var m = make(map[string]int)

// ❌ 整数溢出风险（支付场景）（Block）
func calcAmount(cents int, quantity int) int {
    return cents * quantity // quantity 极大时溢出
}

// ✅ 使用 big.Int 或提前检查
func calcAmount(cents int64, quantity int64) int64 {
    product := cents * quantity
    if product < 0 { /* 溢出处理 */ }
    return product
}

// ❌ 缺少 context 超时（Block）
func SendEmail(ctx context.Context, to string) error {
    // ❌ 如果 DB 或网络 hang，goroutine 永久泄漏
    result := db.QueryContext(context.Background(), ...) // 忽略传入的 ctx
}

// ✅ 正确传递 context
func SendEmail(ctx context.Context, to string) error {
    result, err := db.QueryContext(ctx, ...) // 使用传入的 ctx
}

// ❌ 资源泄漏（未关闭连接）（Block）
func readData() {
    resp, _ := http.Get(url)
    defer resp.Body.Close()
    // 正常情况 ok，但错误路径 resp=nil 时 defer 也会执行
}

// ✅ 正确写法
func readData() error {
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    return nil
}
```

#### 1.3 严重性能退化（P1 — 强烈建议修复）

```go
// ❌ O(n²) 查询（Block 大数据量场景）
func getAllUserEmails(userIDs []int64) []string {
    var emails []string
    for _, id := range userIDs {
        // ❌ N+1 查询：10万用户 = 10万次 DB 查询
        email := db.QueryScalar("SELECT email FROM users WHERE id = ?", id)
        emails = append(emails, email)
    }
    return emails
}

// ✅ 批量查询
func getAllUserEmails(userIDs []int64) []string {
    if len(userIDs) == 0 {
        return nil
    }
    // 一次 IN 查询
    placeholders := strings.Repeat("?,", len(userIDs)-1) + "?"
    rows, _ := db.Query("SELECT email FROM users WHERE id IN ("+placeholders+")", userIDs...)
    // ...
}

// ❌ 大对象拷贝进 channel（Block 高并发）
ch := make(chan []byte)
go func() {
    hugeData := loadHugeData() // 10MB
    ch <- hugeData             // 每次拷贝 10MB，高并发时内存暴涨
}()

// ✅ 传指针或用池
ch := make(chan []byte, 100)
go func() {
    data := getFromPool() // 从 sync.Pool 获取
    ch <- data            // 用完放回池
}()
```

#### 1.4 缺少关键测试（Block）

```go
// ❌ 支付、权限、金额计算等核心逻辑无测试（Block）
// 生产核心路径必须有单测
func CalculateDiscount(price int64, coupon *Coupon) int64 {
    if coupon == nil {
        return price
    }
    // 复杂折扣逻辑，无测试 = 高风险
}
```

---

### 2. 建议类：可以先合并，后续优化

#### 2.1 命名与可读性

```go
// ⚠️ 建议：变量命名可以更清晰
for i, v := range data {
    process(v)  // v 语义不明
}
for i, user := range users {
    process(user)  // 更好
}

// ⚠️ 建议：魔法数字用常量
if retry > 3 {   // 3 是什么意思？
    return err
}

// ✅ 建议写法
const maxRetries = 3
if retry > maxRetries {
    return ErrMaxRetriesExceeded
}
```

#### 2.2 重复代码

```go
// ⚠️ 建议：提取公共函数
func processOrder(ctx context.Context, o *Order) error {
    // 校验订单
    if o.Total <= 0 {
        return ErrInvalidTotal
    }
    if o.Status != StatusPending {
        return ErrInvalidStatus
    }
    // 处理...
    return nil
}

func validateOrder(o *Order) error {
    if o.Total <= 0 {
        return ErrInvalidTotal
    }
    if o.Status != StatusPending {
        return ErrInvalidStatus
    }
    return nil
}
// ⚠️ 建议：抽取 validateOrder，其他地方复用
```

#### 2.3 注释与文档

```go
// ⚠️ 建议：复杂逻辑加注释说明 WHY
// 为什么用 BTree 而非 Hash？
// BTree 支持范围查询，且迭代器稳定；Hash 在大量范围查询时需重建迭代器
var index = btree.New(32, /* degree, 控制树高 */)

/// ⚠️ 建议：导出的 API 写 godoc
// SendOrder 将订单推送到消息队列，重试最多 3 次。
// 返回错误时订单已在队列中，最终由消费者保证幂等处理。
func SendOrder(ctx context.Context, order *Order) error
```

---

### 3. Review 流程规范

#### 3.1 作者职责

| 要求 | 说明 |
|------|------|
| **PR 越小越好** | 单次 Review < 400 行，超过则拆 |
| **自审再提交** | `git diff --cached` 确认改动符合预期 |
| **描述清楚 WHY** | PR 说明改了什么、为什么改、怎么验证 |
| **标注风险项** | 重大变更在 PR 描述中说明回滚方案 |
| **关联 Issue** | PR 关联对应需求/缺陷 Issue |

#### 3.2 Reviewer 职责

```markdown
# PR Review Comment 模板

## 🔴 Must Fix（阻塞合并）
[安全/正确性问题，必须修复后再合并]

## 🟡 Should Fix（强烈建议）
[性能/可维护性问题，建议修复，可以例外合并但需备注]

## 🔵 Nit（次要建议）
[命名格式、代码风格，不阻塞合并]

## 💬 Question（疑问）
[理解代码逻辑的疑问，不是必须回答但建议澄清]

## ✅ Approve（通过）
[满足合并标准]
```

#### 3.3 常见误判

| ❌ 误判 | ✅ 正确做法 |
|--------|------------|
| 用自己的风格偏好 Block 别人的等效写法 | 风格问题 → 用 `gofmt` + `golangci-lint` 自动约束，非人工 Review |
| 过度追求完美，单次 Review 提 50+ 条意见 | 优先级排序，聚焦 P0/P1，其余开后续 Issue |
| 看到不熟悉的技术就要求改成熟悉方案 | 询问作者理由，尊重技术选型自由 |
| 拒绝任何例外（"我们一直这么做"） | 例外是合理的，但需记录到规范文档 |

---

### 4. Go 项目常见检查项清单

```bash
# Review 前可先跑这些工具
golangci-lint run ./...      # 综合 linter
govulncheck ./...            # CVE 检查
go test -race -cover ./...   # 单测 + 竞态检测
go mod tidy && go mod verify # 依赖规范性
```

| 分类 | 检查点 |
|------|--------|
| 并发安全 | `sync` 原语使用、channel 是否有泄漏、Goroutine 是否受控 |
| 错误处理 | 错误是否 wrap、是否丢失上下文、是否暴露内部错误给用户 |
| 资源管理 | defer 在错误路径是否安全、HTTP response body 是否关闭 |
| 性能 | N+1 查询、循环内查询、内存分配热点、大对象拷贝 |
| 可测性 | 核心逻辑是否依赖全局状态、是否可 mock |

---

## 高频追问

**Q：Leader 要求强行合并你的 Review 意见，但你认为有风险怎么办？**

两条路：① 记录在案，让 Leader 书面确认决策（邮件/IM），保护自己；② 如果是 P0 安全问题，有权拒绝合并并上报技术委员会。核心原则：**技术风险不能因为业务压力而被掩盖**。

**Q：如何 Review 自己写的代码？**

① 放下自我视角，用"接手别人代码"的视角重读；② 用 `git blame` 看自己的 PR-diff 而不是全量代码；③ 用 Claude/CodeRabbit 等 AI Review 工具做初步筛查；④ 重点检查边界条件、错误路径、资源释放。

**Q：Go 项目 Review 一定要用 golangci-lint 吗？**

不是必须，但非常推荐。golangci-lint 可以统一团队规范（40+ linter 规则），减少 Review 中的风格之争。团队应维护自己的 `.golangci.yml`，将高优先级规则（如 `errcheck`、`goconst`）开启，其他作为 Suggestion。

---

## 延伸阅读

- [Google Engineering Practices](https://google.github.io/eng-practices/review/)
- [The Code Review Pyramid](https://www.smartsheet.com/community/article/got-bugs-llc/code-review-pyramid)
- [Practical Go](https://dave.cheney.net/practical-go/handbook)

---

**[← 上一篇：Goroutine 泄漏排查](./05-goroutine-leak/05-goroutine-leak.md)**
