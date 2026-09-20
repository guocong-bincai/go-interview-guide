# Flaky Test 治理：从"重跑一次就过了"到 CI 结果可信

> 考察频率：★★★★☆  优先级：P1
> 关键词：不稳定测试、-count / -shuffle、testing/synctest、时间与随机依赖、测试隔离、quarantine 策略

## 面试官考察意图

"测试挂了，重跑一下就过了"——这句话在高级工程师面试里是**减分项**。Flaky Test（不稳定测试）的破坏力在于：**它让团队不再相信 CI**，然后真实的失败也被当成噪声忽略。

面试官想确认：

- 你是否知道 flaky 的**五类根因**和对应的**确定性修法**
- 你是否有**检测机制**（不是靠人碰运气）
- 你是否知道 **Go 官方给了哪些工具**（`-count` / `-shuffle` / `testing/synctest` / `-race`）
- 你有没有**治理策略**（quarantine 隔离 vs 一刀切重试）

**核心立场：重试是止血，不是治疗。允许长期重试的 flaky 测试，等于允许 CI 说谎。**

---

## 核心答案（30 秒版）

Flaky Test = 同一份代码、同一个测试，**多次运行结果不一致**。五类根因：

1. **时间依赖**（`time.Sleep`、真实时钟、超时边界）
2. **随机/顺序依赖**（未播种的 `math/rand`、map 遍历顺序、测试间共享状态）
3. **并发**（goroutine 未同步、共享变量、`-race` 才暴露的数据竞争）
4. **外部依赖**（网络、DB、容器、第三方 API 的抖动）
5. **资源泄漏**（未关闭的文件/连接/goroutine，导致后续测试受影响）

修法对应五条：**注入时钟**（`clock.Clock` 接口）→ **固定种子 + `t.Setenv`/`t.Cleanup` 隔离** → **用 channel/WaitGroup 代替 sleep** → **testcontainers 起真实依赖 + 超时宽松** → **`defer` 关闭一切**。

工具链：
- 检测：`go test -count=20 -shuffle=on ./...`、`go test -race`
- 修时间问题：**`testing/synctest`**（Go 1.24 实验引入，**Go 1.25 转正为标准库**，用虚拟时钟 + bubble 隔离）
- 治理：CI 记录 flaky 率，标记为 flaky 的测试**先隔离（quarantine）再修**，禁止无限重试

---

## 深度展开

### 一、Flaky 的成本模型（面试可量化说）

```text
假设：1000 个测试，单个 flaky 概率 0.5%
  单次 CI 通过率 = 0.995^1000 ≈ 0.0067   ← 几乎必然失败！

即使每个测试只有 0.1% 的抖动：
  单次通过率 = 0.999^1000 ≈ 0.37        ← 63% 概率失败

后果：
  → 大家默认"CI 挂了重跑"
  → 真实回归被淹没在噪声里
  → 平均 2~3 次重跑才能拿到绿灯，CI 时间成本 ×2.5
  → 最终形态：CI 被设为 non-blocking（不阻断合并）——质量门禁彻底失效
```

**这就是"1% 的 flaky 也能毁掉 CI 可信度"的数学原因**：通过率是连乘的。

### 二、根因 1：时间依赖（最常见）

```go
// ❌ flaky：依赖真实时钟 + sleep，机器慢一点就挂
func TestCacheExpiry(t *testing.T) {
    c := NewCache(50 * time.Millisecond)
    c.Set("k", "v")
    time.Sleep(60 * time.Millisecond)   // CI 负载高时 Set 和 Sleep 之间就被 GC 打断
    if _, ok := c.Get("k"); ok {
        t.Fatal("期望过期，但仍命中")
    }
}
```

**修法 A：注入时钟接口**

```go
// 接口化时钟，生产用真实时钟，测试用假时钟
type Clock interface {
    Now() time.Time
    After(d time.Duration) <-chan time.Time
}

type realClock struct{}
func (realClock) Now() time.Time { return time.Now() }
func (realClock) After(d time.Duration) <-chan time.Time { return time.After(d) }

// 测试里推进时间，零等待、完全确定
type fakeClock struct{ now time.Time }
func (c *fakeClock) Now() time.Time { return c.now }
func (c *fakeClock) Advance(d time.Duration) { c.now = c.now.Add(d) }
```

**修法 B：用 `testing/synctest`（Go 1.25 标准库）——更彻底**

```go
// Go 1.24: 需 GOEXPERIMENT=synctest，API 为 synctest.Run
// Go 1.25+: 标准库直接可用，API 为 synctest.Test
func TestCacheExpiryWithSynctest(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        c := NewCache(50 * time.Millisecond)
        c.Set("k", "v")

        time.Sleep(60 * time.Millisecond) // ← 虚拟时间，瞬时完成！

        if _, ok := c.Get("k"); ok {
            t.Fatal("期望过期，但仍命中")
        }
    })
}
```

`synctest` 的两个核心能力：

- **虚拟时钟**：bubble 内的 `time.Sleep`/`time.After` **不真的等待**，测试跑完 50ms 逻辑只需微秒
- **`synctest.Wait()`**：等待 bubble 内所有 goroutine 进入"稳定"（全部阻塞）状态，**替代 `time.Sleep` 做同步点**

这解决了并发测试里最难的两件事：**又要等 goroutine 跑，又不想睡固定时间**。

**注意点**：
- bubble 内的 goroutine 不能做真实 IO（网络/文件），否则 panic
- Go 1.24 的 `synctest.Run()` 在 1.25 已废弃，用 `synctest.Test()`（`GOEXPERIMENT=synctest` 时旧 API 才存在，**Go 1.26 将移除**）

**修法 C：如果实在要用真实时间，放大裕度并重试**

```go
// 最后手段：明确标注 + 放大裕度（不推荐作为长期方案）
if testing.Short() {
    t.Skip("skipping timing-sensitive test in short mode")
}
deadline := time.Now().Add(5 * time.Second)   // 从 60ms 放大到秒级
for time.Now().Before(deadline) {
    if _, ok := c.Get("k"); !ok {
        return // 已过期，通过
    }
    time.Sleep(10 * time.Millisecond)
}
t.Fatal("超时：缓存始终未过期")
```

### 三、根因 2：顺序与共享状态

```go
// ❌ 共享的包级变量被多个测试改
var globalConn = mustConnectDB()   // 包级初始化，所有测试共用

func TestA(t *testing.T) { globalConn.Exec("DELETE FROM orders") }
func TestB(t *testing.T) { /* 依赖 orders 表里有数据 —— 取决于 A 是否先跑 */ }

// ✅ 每个测试独立依赖，用 t.Cleanup 保证回收
func TestA(t *testing.T) {
    db := mustConnectDB(t)
    t.Cleanup(func() { db.Close() })
    // 用事务包裹，测试结束自动回滚
    tx := db.Begin()
    t.Cleanup(func() { tx.Rollback() })
}
```

**Go 提供的检测工具**：

```bash
# -shuffle 打乱测试顺序，暴露顺序依赖（默认用时间做种子）
go test -shuffle=on ./...

# 固定种子以便复现失败
go test -shuffle=12345 ./...

# -count 重复跑，暴露概率性失败
go test -count=50 ./pkg/cache

# -p 1 串行跑包，排除包间干扰（定位用，不是长期方案）
go test -p 1 ./...
```

**map 遍历顺序**是 Go 里最经典的隐性随机源：

```go
// ❌ 输出顺序随机（Go 故意随机化 map 遍历）
for k, v := range m {
    out = append(out, k+v)
}
// ✅ 需要确定顺序时，先排序 key
keys := slices.Sorted(maps.Keys(m))
```

### 四、根因 3：并发（`-race` 是底线）

```go
// ❌ 数据竞争：结果取决于调度
func TestCounterConcurrent(t *testing.T) {
    var count int
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() { defer wg.Done(); count++ }()  // race！
    }
    wg.Wait()
    if count != 100 { t.Fatalf("got %d", count) } // 有时 98，有时 100
}
```

```bash
# CI 必备：-race 能直接指出竞争位置
go test -race ./...
```

**竞态测试的正确写法**：不要用 sleep 等 goroutine，用 **channel 或 WaitGroup 建立明确的同步点**；需要"等它开始"时用 `started := make(chan struct{})` 显式通知。

### 五、根因 4：外部依赖

| 外部依赖 | 修法 |
|----------|------|
| 数据库 | **testcontainers-go** 起真实容器（比 sqlite mock 更准），或用事务回滚隔离 |
| Redis | miniredis（纯内存实现）或 testcontainers |
| HTTP 第三方 | `httptest.NewServer` 起本地假服务；断言请求内容而非依赖真实 API |
| 时间/时区 | `t.Setenv("TZ", "UTC")`（`t.Setenv` 会自动恢复） |
| 文件系统 | `t.TempDir()`（自动清理，且每个测试独立目录） |
| 端口 | 用 `:0` 让内核分配，不要硬编码 `:8080` |

```go
// httptest 是解决"外部服务抖动"最有效的工具
func TestPaymentClient(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(503)                 // 精确模拟故障
    }))
    defer srv.Close()
    t.Setenv("PAYMENT_URL", srv.URL)
    // ...断言客户端对 503 的处理
}
```

### 六、检测与治理机制（这是高级工程师的答案）

#### 检测：把 flaky 变成数字

```bash
# CI 夜间任务：每个包跑 20 次，输出失败率
for pkg in $(go list ./...); do
  pass=0
  for i in $(seq 1 20); do
    go test -count=1 -timeout=60s "$pkg" >/dev/null 2>&1 && pass=$((pass+1))
  done
  rate=$(( pass * 100 / 20 ))
  [ "$rate" -lt 100 ] && echo "FLAKY $pkg pass-rate=${rate}%"
done
```

进阶做法：CI 里给每个测试用例打标签上报（如 `gotestsum --format=json` 解析），在 Grafana 里画"测试用例 flaky 率排行"，**每周治理 top 10**。

#### 治理：quarantine（隔离）策略

```go
// 方式 1：Go 官方支持的跳过标记（可被 CI 统计）
func TestFlakyOne(t *testing.T) {
    if os.Getenv("CI") != "" {
        t.Skip("flaky: 见 JIRA-1234，正在修复")   // ← 必须带 issue 号！
    }
    // ...
}
```

**隔离策略必须配三条纪律**：

1. **必须关联 issue**，否则会永远隔离下去
2. **有修复 SLA**（比如 2 周），到期未修则升级为 P0 技术债
3. **隔离的测试计入技术债看板**，不让它消失在视野外

#### 重试的边界（面试高频追问）

| 允许 | 禁止 |
|------|------|
| 集成/E2E 测试因外部依赖抖动重试 1 次，且**重试结果上报** | 单元测试无脑重试直到过 |
| 重试仅作为**临时止血**，同时开修复 issue | 靠重试掩盖真实回归 |
| 重试必须**留痕**（记录到 flaky 台账） | 重试后悄悄通过，没人知道 |

---

## 高频追问

**Q1：`testing/synctest` 有什么限制？**
三个主要限制：① **bubble 内的 goroutine 不能做真实 IO**（网络、文件、`os.Exit`），否则测试 panic——它只能测纯内存/纯计算的并发逻辑；② 需要用 `synctest.Wait()` 建立同步点，如果忘了，虚拟时间和真实调度会不一致；③ Go 1.24 里需要 `GOEXPERIMENT=synctest` 且 API 是 `Run()`，**Go 1.25 起转正为标准库、API 改为 `Test()`**，Go 1.26 将移除旧 API。所以跨版本代码要小心。

**Q2：`-race` 能发现所有并发问题吗？**
不能。`-race` 是**动态检测**，只能发现**本次运行中实际发生的**竞争。两个后果：① 没跑到的路径即使有竞争也报不出来（所以 CI 要跑 `-count` 多次）；② `-race` 会让程序变慢 5~20 倍、内存增长 5~10 倍，不适合在性能测试时开。**它是底线不是保证**——必须配合精心设计的并发测试用例。

**Q3：为什么 `t.Parallel()` 会让 flaky 变多？**
因为并行测试共享进程级资源：**环境变量**、**当前工作目录**、**包级变量**、**全局单例连接**、**CPU/内存竞争导致的超时**。规则：用了 `t.Parallel()`，就只能依赖 `t.TempDir()` 和显式注入的依赖，绝不能碰包级可变状态。另外 `t.Parallel()` 的测试与串行测试的执行顺序在 Go 里是确定的（并行测试会等所有串行测试结束后一起跑），但**它们之间**顺序不保证。

**Q4：`-shuffle=on` 会暴露什么问题？**
最典型的是**测试间的隐式顺序依赖**：TestA 写了一条数据，TestB 恰好依赖它存在。打乱后随机失败。另一个常见的是**包级 `init()` 或 `TestMain` 里的全局状态**被多个测试累积修改。修复方向是让每个测试**自给自足**（自己造数据、自己清理），而不是依赖执行顺序。

**Q5：容器化的集成测试会不会引入新的 flaky？**
会，主要是三类：① **容器启动竞态**——端口还没 ready 就开始测，必须做健康检查轮询（testcontainers-go 的 `WaitFor` 策略）；② **资源不足导致超时**——CI 机器负载高，测试超时要设得宽松（如 60s 而不是 5s）；③ **镜像拉取失败**——用内网镜像代理，并给拉取设重试。**这三类都是"外部环境"问题，可以用重试缓解，但真正的逻辑 flaky 必须修。**

---

## 面试话术

> "Flaky test 处理我们的立场很明确：重试是止血不是治疗，允许长期重试等于允许 CI 说谎。检测上 CI 夜间任务用 `-count=20 -shuffle=on -race` 跑全量，把每个用例的通过率上报，每周治理 flaky 率 top 10。修法按根因分五类：时间依赖用 fake clock 或者 Go 1.25 转正的 `testing/synctest`——虚拟时钟让原本要睡 50 毫秒的测试瞬间跑完，还完全确定；状态污染用 `t.TempDir` 和 `t.Cleanup`；并发用 channel 建同步点而不是 sleep；外部依赖用 httptest 和 testcontainers 起真实容器。治理上被判定 flaky 的测试走 quarantine，但必须挂 issue 号并且有 2 周修复 SLA，隔离项计入技术债看板。我们治理了两轮，CI 的一次通过率从 60% 提到了 98% 以上。"

---

[🏠 首页](../../../README.md) · [📦 工程素养](../README.md) · [📈 质量治理与交付](./README.md)
