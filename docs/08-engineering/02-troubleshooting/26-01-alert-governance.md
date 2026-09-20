# 告警治理与 On-Call 工程化：从"每天 800 条告警"到"每条都值得看"

> 考察频率：★★★★☆  优先级：P1
> 关键词：告警风暴、告警分级、告警疲劳、SLO 燃烧率、MTTD/MTTR、Runbook、值班轮转

## 面试官考察意图

这道题是**高级工程师（Owner 视角）的分水岭**。中级工程师会配 Prometheus 告警规则；高级工程师要回答的是：

- 告警**配多少合适**？多了会怎样？
- 告警风暴（一次故障刷 800 条）怎么收敛？
- 怎么衡量"告警体系本身健不健康"？
- On-Call 值班被叫醒，**多久能定位**？有没有 Runbook？

面试官最想听到的一句话是：**"告警的目标不是覆盖所有异常，而是让每条被推送的告警都可执行（actionable）。"**

---

## 核心答案（30 秒版）

告警治理的核心指标是三个：**信噪比**（有效告警 / 总告警，健康值 > 50%）、**MTTD**（发现时间）、**MTTR**（恢复时间）。治理手段分四层：

1. **规则层**：所有告警加 `for:` 持续时间抑制瞬时抖动；阈值基于 SLO 而非拍脑袋（如"错误预算燃烧率 > 14.4"而非"错误率 > 1%"）
2. **收敛层**：AlertManager 的 `group_by`（按服务聚合）+ `inhibit_rules`（根因抑制衍生告警）+ `silence`（维护窗口）
3. **分级层**：P0 电话/P1 群消息/P2 工单/P3 日报，**只有 P0 允许半夜叫人**
4. **闭环层**：每条告警必须有 Runbook 链接，值班复盘产出改进项，把"人肉能处理的告警"逐步自动化或降级

**判定标准：如果一条告警值班同学看到后的动作是"再看一眼，没事"，那它就不该存在。**

---

## 深度展开

### 一、告警疲劳的数学

```text
假设一个 30 人的研发团队：
  平均每人参与 5 个服务   → 150 个服务维度
  每个服务配 20 条告警    → 3000 条规则
  每条每天误报 0.2 次     → 每天 600 条无效告警
  分散到 2 个值班同学     → 每人每天 300 条

结果：值班同学在 30 分钟内学会"看到就关掉"——告警体系彻底失效。
```

**告警疲劳不是态度问题，是设计问题。** 人对重复无效刺激的脱敏是生理性的。

### 二、第一层：规则设计——只在真的需要人介入时告警

```yaml
groups:
  - name: checkout-slo
    rules:
      # ✅ 好告警：基于错误预算燃烧率，直接对应"用户正在难受"
      # 含义：1 小时窗口内烧掉了 30 天预算的 14.4 倍速度 → 2 天内会烧完
      - alert: CheckoutErrorBudgetBurnFast
        expr: |
          (
            sum(rate(http_requests_total{service="checkout",code=~"5.."}[1h]))
            / sum(rate(http_requests_total{service="checkout"}[1h]))
          ) > (14.4 * (1 - 0.999))
        for: 2m                     # ← 关键：持续 2 分钟才告警，过滤瞬时抖动
        labels:
          severity: critical
          team: checkout
        annotations:
          summary: "结算服务错误预算燃烧过快（1h 窗口）"
          description: "当前错误率 {{ $value | humanizePercentage }}，2 天内将耗尽月度预算"
          runbook_url: "https://wiki.internal/runbooks/checkout-5xx"

      # ❌ 坏告警：CPU 高
      # CPU 80% 不等于用户受影响，它可能只是这台机器在满负荷做有用功
      # - alert: HighCPU
      #   expr: rate(process_cpu_seconds_total[5m]) > 0.8
      #   for: 1m
```

**规则设计三原则**：

| 原则 | 说明 | 反例 |
|------|------|------|
| **面向用户影响** | 告警必须对应 SLI 变差 | "内存使用率 75%" |
| **可执行** | 收到后有明确动作 | "Kafka lag 增长"（没有阈值和动作） |
| **有归属** | 必须带 `team` 和 `runbook_url` | 无 label 的告警涌进公共群 |

### 三、第二层：收敛——AlertManager 三大武器

```yaml
route:
  receiver: default
  group_by: [alertname, service, cluster]   # ← 同一服务同一告警合并成一条
  group_wait: 30s                            # 首个告警等 30s，同类一起发
  group_interval: 5m                         # 聚合组内新增告警的发送间隔
  repeat_interval: 4h                        # 未恢复告警重复提醒间隔（别设 1h！）
  routes:
    - matchers: [severity="critical"]
      receiver: pagerduty          # P0 → 电话叫人
      continue: false
    - matchers: [severity="warning"]
      receiver: feishu-warning     # P1 → 群消息，不叫人
      continue: false
    - matchers: [severity="info"]
      receiver: ticket-system      # P2 → 建工单，每天汇总

inhibit_rules:
  # 抑制规则：根因告警触发时，静音它的衍生告警
  - source_matchers: [alertname="ServiceDown", service="mysql"]
    target_matchers: [service="checkout"]      # MySQL 挂了，checkout 报错很正常
    equal: [cluster]
  - source_matchers: [alertname="NodeNotReady"]
    target_matchers: [alertname=~"Pod.*|Container.*"]
    equal: [node]
```

**抑制规则的价值**：一次 MySQL 主库故障，如果没配抑制，会同时触发 checkout / order / payment / inventory 的 5xx 告警 + 数据库连接池耗尽告警 + Pod 重启告警，**值班同学面对 40 条告警，第一反应是"哪个是根因"而不是"怎么修"**。配上抑制后只剩 1 条 MySQL 告警，直接指向根因。

### 四、第三层：分级——什么级别配什么通道

| 级别 | 判定标准 | 通知方式 | 响应要求 | 允许半夜叫醒 |
|------|----------|----------|----------|--------------|
| **P0** | 核心链路不可用 / 数据丢失风险 | 电话 + 群 + 短信 | 5 分钟内响应 | ✅ |
| **P1** | 核心链路降级 / 错误预算燃烧过快 | 群消息 @ 值班 | 30 分钟内响应 | ❌（工作时间） |
| **P2** | 非核心功能异常 / 容量预警 | 工单 / 日报汇总 | 1 个工作日 | ❌ |
| **P3** | 趋势性异常、技术债信号 | 看板 / 周报 | 纳入迭代 | ❌ |

**只有 P0 允许半夜叫人**——这是最需要坚持的纪律。一旦 P1 也开始半夜打电话，值班同学就会把所有通知静音，P0 也会被漏掉。

### 五、第四层：闭环——Runbook 与复盘

```markdown
# Runbook: checkout-5xx

## 告警含义
结算接口 5xx 错误率超过 SLO 阈值，可能影响用户下单。

## 影响范围
- 用户侧：无法下单，转化率下降
- 依赖侧：支付网关、库存服务可能受影响

## 快速定位（按顺序执行）
1. 看错误类型分布
   `sum by (code, path) (rate(http_requests_total{service="checkout",code=~"5.."}[5m]))`
2. 看下游依赖
   `sum by (upstream) (rate(upstream_errors_total{service="checkout"}[5m]))`
3. 看最近变更
   `kubectl rollout history deployment/checkout -n prod | tail -5`

## 已知处置方案
| 错误特征 | 根因 | 处置 |
|----------|------|------|
| `context deadline exceeded` 集中在支付调用 | 支付网关超时 | 联系支付团队值班 / 降级到备用通道 |
| `too many connections` | DB 连接池耗尽 | 临时提高 max_open_conns 并扩容 |
| `panic: runtime error` | 新版本代码缺陷 | `kubectl rollout undo` 回滚 |

## 升级路径
15 分钟未恢复 → 升级给 checkout owner（电话：xxx）

## 历史案例
- 2026-05-12：营销活动大 payload 导致反序列化超时 → 已增加字段大小限制
```

**Runbook 的验收标准**：一个没参与过这个服务的值班同学，照着 Runbook 能完成前两步定位。

### 六、SLO 燃烧率告警（Google SRE 多窗口法）

单窗口燃烧率告警有矛盾：窗口太长（如 6h）反应慢，窗口太短（如 5m）太敏。标准解法是**多窗口多燃烧率**：

```yaml
# 1h 窗口 + 5m 窗口同时满足才告警 → 既快速又抗抖动
- alert: ErrorBudgetBurnFast
  expr: |
    (
      sum(rate(http_requests_total{service="checkout",code=~"5.."}[1h]))
      / sum(rate(http_requests_total{service="checkout"}[1h]))
      > (14.4 * 0.001)
    )
    and
    (
      sum(rate(http_requests_total{service="checkout",code=~"5.."}[5m]))
      / sum(rate(http_requests_total{service="checkout"}[5m]))
      > (14.4 * 0.001)
    )
  labels:
    severity: critical
```

配合的燃烧率分级（SLO = 99.9%，30 天窗口）：

| 燃烧率 | 剩余时间 | 动作 |
|--------|----------|------|
| 14.4× | 2 天烧完 | 页面告警（P0） |
| 6× | 5 天烧完 | 工单（P1） |
| 1× | 30 天烧完 | 日报（P3） |

---

## 高频追问

**Q1：怎么衡量告警体系本身健不健康？**
四个指标：① **信噪比**（被人工标注为"有效"的告警 / 总告警），目标 > 50%，低于 30% 说明体系失效；② **MTTD**（故障发生到告警触发），目标 < 5 分钟；③ **MTTR**（告警到恢复）；④ **告警重复率**（同根因告警的条数），一故障 > 10 条说明抑制没配好。**每季度跑一次"告警复盘"，把 P2/P3 里永远没人看的告警直接删掉。**

**Q2：`repeat_interval` 设太短有什么问题？**
默认 4h 是有道理的。设成 10 分钟会导致未恢复的告警每 10 分钟刷屏一次，值班同学会把它静音——然后真正的恢复通知也看不到了。**判断标准：如果告警没恢复，值班同学应该在看 Runbook，而不是在等第 8 条重复通知。**

**Q3：告警风暴时值班同学应该做什么？**
三步：① **先静音衍生告警**（`amtool silence add` 按服务静音），清空噪声；② **只看 P0 里的根因告警**，直接翻 Runbook；③ **开故障群同步信息**，把"我在查什么、查到什么"写进去，避免多人重复排查。**切忌在风暴中逐条点开告警**——告警风暴本身就是"你在看告警而不是在修问题"的陷阱。

**Q4：值班被叫醒了但发现是误报，怎么办？**
必须当场做两件事：① 记录到"误报台账"（告警名 + 原因）；② 当天提 PR 修规则（加 `for:`、调阈值、加抑制）。**不修的误报会在三个月后变成"狼来了"，让值班同学漏掉真故障。** 这是我们团队硬性要求：误报当天必须闭环。

**Q5：Go 服务有什么特有的告警项？**
几个易漏的点：① `go_goroutines` 持续单调上升（goroutine 泄漏，见 05-goroutine-leak）；② `go_gc_duration_seconds` P99 突增（GC 压力，通常是分配暴增的前兆）；③ `go_memstats_heap_inuse_bytes` 逼近容器 limit 但 `OOMKilled` 还没触发（提前 10 分钟预警）；④ **`container_cpu_cfs_throttled_seconds_total`**——CPU 被限流，这个指标比 CPU 使用率更能反映"服务在难受"。

---

## 面试话术

> "告警治理我们看三个指标：信噪比、MTTD、MTTR。手段是四层：规则上所有告警基于 SLO 燃烧率而不是拍脑袋的阈值，并且统一加 `for:` 过滤抖动；收敛上用 AlertManager 的 group_by 聚合 + inhibit_rules 做根因抑制——之前一次 MySQL 故障刷了 40 条告警，配了抑制以后只剩 1 条指向根因；分级上严格只有 P0 允许半夜打电话，P1 以下走群和工单；闭环上每条告警必须带 Runbook 链接，误报当天必须提 PR 修掉。我们的信噪比从原来的不到 20% 提到了 60% 以上，值班同学被叫醒的次数下降了一个数量级。"

---

[🏠 首页](../../../README.md) · [📦 工程素养](../README.md) · [🩺 线上问题排查](./README.md)
