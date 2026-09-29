# 特性开关（Feature Flag）：渐进式发布的工程化落地

> 考察频率：★★★★☆  优先级：P1
> 关键词：Feature Flag、特性开关、渐进式发布、灰度、Trunk-Based Development、开关腐化、Go 实现

## 面试官考察意图

"你做过灰度发布吗？"——如果候选人只会答"K8s 按权重切流量"，那只是**部署级灰度**。更高级的答案是**特性开关（Feature Flag）**：代码已经全量上线，但功能对新用户不可见，靠一个开关在**运行时**控制谁看到什么。

这是高级工程师的强区分点，因为 Feature Flag 串起了三件事：

1. **发布解耦**：部署（Deploy）与发布（Release）分离，代码合并进主干 ≠ 用户可见；
2. **Trunk-Based Development 的前提**：没有开关，长分支就会一直存在；有了开关，半成品也能安全合入主干；
3. **开关治理**：开关是**技术债**，用完不删就是"僵尸开关"，面试官想听你有没有治理机制。

**核心立场：Feature Flag 是一把双刃剑——它让发布变安全，但如果不做生命周期管理，一年后你的代码会被几百个开关的 `if/else` 淹没。**

---

## 核心答案（30 秒版）

Feature Flag = **一个运行时可变的配置**，包裹一段新逻辑，用来控制"谁、在什么条件下、看到哪条代码路径"。

按用途分四类（这是最关键的分类，面试必答）：

| 类型 | 生命周期 | 用途 | 例子 |
|------|----------|------|------|
| **Release Flag** | 短（天～周） | 渐进式放量、A/B | `checkout_v2_enabled` |
| **Experiment Flag** | 短（周） | A/B 实验、指标对比 | `new_ranking_algorithm` |
| **Ops Flag** | 中 / 长 | 运维降级开关、熔断 | `disable_recommend` |
| **Permission Flag** | 长（永久） | 付费权限、企业版功能 | `feature_export_excel` |

Go 侧最小实现：**配置中心（或 DB）存开关值 → 进程内 `atomic` 缓存 → 请求级求值（用户维度分流）**。关键设计点：

- **求值要快**：走内存，绝不每次请求查 DB / 打远程；
- **分流要稳定**：同一用户永远落在同一桶（`hash(userID) % 100`），否则体验闪烁；
- **要有兜底**：配置中心挂了就用本地快照 / 默认值，开关服务本身不能成为单点。

---

## 深度展开

### 一、为什么不能只靠"部署灰度"

部署级灰度（K8s 权重、Ingress 分流）有两个天花板：

1. **只能按实例分流量**，粒度粗。想做"只有 iOS 14 以上用户看到新结算页"——部署灰度做不到，必须靠开关；
2. **回滚要重新部署**。出问题时，改一个开关值（秒级）远比重新发布（分钟级 + 全量重启）快。

因此成熟团队是**双层灰度**：

```
部署灰度（Pod 权重 10%） → 特性开关（用户白名单 → 1% → 10% → 100%）
```

部署灰度卡住"机器维度"的风险，开关卡住"用户维度"的体验。

### 二、Go 实现：一个够用的 Feature Flag 引擎

```go
package flag

import (
    "context"
    "encoding/json"
    "hash/fnv"
    "sync/atomic"
)

// Rule 描述一个开关的求值规则
type Rule struct {
    Enabled  bool              `json:"enabled"`   // 总开关
    Rollout  int               `json:"rollout"`   // 放量百分比 0-100
    AllowIDs []int64           `json:"allow_ids"` // 白名单（内部测试）
    Attrs    map[string]string `json:"attrs"`     // 附加条件，如 platform=ios
}

type Engine struct {
    snapshot atomic.Pointer[map[string]Rule] // 无锁读
}

// Apply 由配置中心回调 / 定时拉取触发，整体替换快照
func (e *Engine) Apply(raw []byte) error {
    var m map[string]Rule
    if err := json.Unmarshal(raw, &m); err != nil {
        return err // 解析失败保留旧快照，避免"配置炸了全站关闭"
    }
    e.snapshot.Store(&m)
    return nil
}

// Eval 请求级求值：同一 userID 结果稳定
func (e *Engine) Eval(ctx context.Context, key string, userID int64) bool {
    m := e.snapshot.Load()
    if m == nil {
        return false // 没有快照 → 安全默认值（关）
    }
    r, ok := (*m)[key]
    if !ok || !r.Enabled {
        return false
    }
    for _, id := range r.AllowIDs { // 白名单直通
        if id == userID {
            return true
        }
    }
    if r.Rollout >= 100 {
        return true
    }
    return bucket(userID, key) < uint32(r.Rollout) // 稳定分桶
}

// bucket: 把 (userID, key) 映射到 [0,100) 的稳定桶
func bucket(userID int64, key string) uint32 {
    h := fnv.New32a()
    _, _ = h.Write([]byte(key))
    var b [8]byte
    for i := 0; i < 8; i++ {
        b[i] = byte(userID >> (8 * i))
    }
    _, _ = h.Write(b[:])
    return h.Sum32() % 100
}
```

三个细节值得在面试里点出来：

- **`atomic.Pointer` 无锁读**：求值在热路径（每个请求都过），绝不能用 `sync.RWMutex` 抢锁；
- **解析失败保留旧快照**：配置中心推来个坏 JSON，最坏结果是"开关不生效"，而不是"全站功能被关掉"；
- **`hash(key+userID)` 而不是 `hash(userID)`**：不同开关用不同 hash 种子，避免"凡是新功能都命中同一批天选之子"（相关性偏差）。

### 三、渐进式发布的正确节奏

```
1. 代码合入主干，开关默认关（Dark Launch / 暗发布）
2. 白名单：内部员工 → Beta 用户
3. 小流量：1% → 5% → 10%（每档停留 ≥ 1 个业务高峰）
4. 观测：盯错误率、P99、核心业务指标（转化率/下单量）
5. 全量：100%，开关保留一周（随时可关）
6. 清理：删除开关与旧代码路径 ← 最容易忘的一步
```

**关键纪律：每一档都要有明确的"观测指标 + 回滚阈值"**。比如"错误率 > 基线 0.5% 或 P99 > 500ms 立即回退到上一档"。

### 四、开关腐化（Flag Debt）：面试官最想听的点

反面案例：

```go
// 😱 一年后的代码：没人知道 checkout_v2 是什么，也没人敢删
if flag.On("checkout_v2") {
    return newFlow()
} else if flag.On("checkout_v2_beta") {
    return betaFlow()
} else if flag.On("checkout_v2_fix") {
    return fixedFlow()
} else {
    return oldFlow()
}
```

治理手段（有这些才叫"工程化"）：

1. **开关必须登记元数据**：owner、创建时间、**过期时间**、上线 issue 链接；
2. **自动告警**：超过过期时间仍为 100% 的 release flag → 自动开清理 ticket；
3. **CI 静态检查**：`golangci-lint` + 自定义 linter / `grep` 扫描已下线的 flag key，禁止新代码引用；
4. **开关数量设上限**：release flag 常驻数量超过阈值就停止新增（倒逼清理）；
5. **区分"永久开关"和"临时开关"**：permission flag 允许永久，release flag 必须有 TTL。

### 五、和"配置中心"的关系

Feature Flag 是配置中心的**一种特化**。区别在于：

- 普通配置是**全局静态**的（超时时间、连接数）；
- 特性开关是**请求级动态求值**的（带用户上下文、分桶、白名单）。

所以不要用"全局配置 + 重启生效"的方式做开关——**开关必须支持热更新 + 请求级求值**。放在配置中心里、用 `atomic` 快照做本地缓存，是标准解法。

### 六、常见坑

1. **开关嵌套**：`if A { if B { if C {} } }`，组合爆炸且无法推理。**禁止嵌套开关**，改成一次性求值出"决策"再 `switch`。
2. **默认值方向搞错**：新功能默认应该**关**（fail-closed）。但**降级类 Ops Flag 要 fail-open**（比如"禁用推荐"开关读不到就按"禁用"走，保护主链路）——**方向取决于"坏掉的代价"**。
3. **只在入口求值一次**：一个请求内多处求值可能因配置热更导致前后不一致，应在中间件里求值后写入 `context`，全链路复用。
4. **测试不可控**：开关让"同一份代码有两种行为"，测试必须显式设定开关状态（`engine.Apply([]byte(...))`），否则测试变成掷骰子。
5. **把它当权限系统用**：Feature Flag 不该承载复杂 RBAC，权限应该走独立的鉴权体系。

---

## 高频追问

**Q1：Feature Flag 和 A/B Test 的区别？**
A/B Test 是**实验**，要统计显著性、有明确的假设和指标；Feature Flag 是**机制**。A/B 通常用 Feature Flag 做流量切分，但 Flag 不关心统计——它只负责"把流量分给谁"。

**Q2：放量百分比怎么保证稳定？**
对 `(flagKey, userID)` 做一致性哈希取模。**千万不能**用 `rand()` 或 `time.Now()`，否则同一用户刷新页面就在新旧两版之间闪。

**Q3：开关配置多久拉一次？**
推送（长轮询 / WebSocket）为主 + 定时兜底（如 30s）。只靠长轮询可能因网络问题丢更新，纯定时又不够实时。**双通道 + 版本号（旧版本丢弃）**最稳。

**Q4：怎么保证删开关不出事？**
先改成"恒定关闭"跑一个版本（观察无异常），再删代码。一次性删代码 + 换逻辑风险最高。

**Q5：多实例间开关不一致怎么办？**
这是最终一致问题——推送有毫秒级窗口。对强一致要求高的场景，把 `flagKey+version` 写进请求上下文，同请求内只用一次求值结果；跨实例的一致性靠配置中心的版本号收敛。

---

## 面试话术

> "我们用**特性开关做渐进式发布**，和部署灰度配合：部署灰度控机器维度，开关控用户维度。实现是配置中心推 `atomic.Pointer` 快照，请求中间件里求值一次写进 context，用 `hash(flagKey+userID)` 稳定分桶。发布节奏是**暗发布 → 白名单 → 1% → 10% → 100%**，每档盯错误率和 P99。开关按用途分 release / experiment / ops / permission 四类，**release flag 强制带 TTL**，过期不删会被自动告警，避免开关腐化。"

---

## 延伸阅读

- [Martin Fowler: Feature Toggles (aka Feature Flags)](https://martinfowler.com/articles/feature-toggles.html)
- [OpenFeature 标准](https://openfeature.dev/)
- 本仓库：[金丝雀发布：K8s 灰度部署 + 优雅关闭 + 快速回滚](../04-performance-governance/25-05-canary-release.md)
