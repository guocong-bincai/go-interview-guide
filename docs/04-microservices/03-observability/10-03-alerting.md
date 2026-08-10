# 告警系统设计与告警疲劳治理

> 考察频率：★★★☆☆  难度：★★★★☆
> 重点：能讲清楚告警规则设计、收敛、分级、告警疲劳治理

## 告警系统架构

### 核心组件

```
Metrics/Logs/Traces
        │
        ▼
Prometheus / Loki / Jaeger（采集）
        │
        ▼
AlertManager（告警规则 + 聚合 + 分发）
        │
        ├──→ Slack / 飞书（通知）
        ├──→ PagerDuty / 电话（紧急）
        └──→ 钉钉 / 微信（普通）
```

---

## Prometheus 告警规则设计

### 核心指标分类

| 类别 | 指标示例 | 告警场景 |
|------|---------|---------|
| **SLO** | 请求成功率、延迟 P99 | SLA 低于 99.9% |
| **资源** | CPU > 80%、内存 > 85%、磁盘 > 90% | 资源紧张 |
| **业务** | 订单失败率、支付成功率 | 业务异常 |
| **基础设施** | Pod 重启次数、Kafka lag | 依赖服务异常 |

### Go 服务告警规则示例

```yaml
groups:
  - name: go-service-alerts
    rules:
      # 1. 服务不可用告警（核心）
      - alert: ServiceDown
        expr: up{job="myapp"} == 0
        for: 1m
        labels:
          severity: critical
          team: infra
        annotations:
          summary: "服务 {{ $labels.instance }} 不可用"
          description: "服务已下线超过 1 分钟，当前值: {{ $value }}"

      # 2. 高错误率告警
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service) > 0.05
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "服务 {{ $labels.service }} 5xx 错误率 > 5%"
          description: "5 分钟内 5xx 占比: {{ $value | humanizePercentage }}"

      # 3. 接口延迟 P99 告警
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 1.0
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "服务 {{ $labels.service }} P99 > 1s"
          description: "当前 P99: {{ $value }}s"

      # 4. Goroutine 泄漏告警
      - alert: GoroutineLeak
        expr: |
          go_goroutines{job="myapp"} 
          > on(instance) go_goroutines{job="myapp"} offset 10m * 1.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Goroutine 数量异常增长"
          description: "当前: {{ $value }}，10分钟前: {{ $value | ...

      # 5. 内存告警（OOM 前兆）
      - alert: HighMemoryUsage
        expr: |
          go_memstats_heap_inuse_bytes{job="myapp"} 
          / go_memstats_heap_sys_bytes{job="myapp"} > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "堆内存使用率 > 85%"
```

---

## 告警分级

### 分级原则

| 级别 | 定义 | 响应时间 | 通知方式 |
|------|------|---------|---------|
| **P0 - Critical** | 服务完全不可用，数据丢失风险 | < 5 分钟 | 电话 + 短信 + 飞书 |
| **P1 - Warning** | 部分功能受损，SLO 降级 | < 15 分钟 | 飞书 + 短信 |
| **P2 - Info** | 潜在风险，不影响服务 | < 2 小时 | 飞书 |

### 分级示例

```yaml
# Critical：立即响应
- alert: ServiceDown
  labels:
    severity: critical
  # for: 1m（等 1 分钟再告警，避免抖动）

# Warning：关注但不必立即处理
- alert: HighErrorRate
  labels:
    severity: warning
  # for: 2m

# Info：趋势性告警
- alert: DiskUsageGrowing
  labels:
    severity: info
```

---

## 告警收敛与聚合

### AlertManager 路由配置

```yaml
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s          # 等待 30s 聚合同组告警
  group_interval: 5m       # 发送间隔
  repeat_interval: 4h      # 重复告警间隔
  receiver: 'default'
  
  routes:
    # Critical 直接电话通知
    - match:
        severity: critical
      receiver: 'critical-receiver'
      group_wait: 10s      # Critical 更快聚合
      repeat_interval: 1h
    
    # Warning 飞书通知
    - match:
        severity: warning
      receiver: 'warning-receiver'
    
    # 按服务分组
    - match:
        service: order
      receiver: 'order-team'
      continue: true      # 继续匹配后续规则
```

### 告警聚合策略

**1. 按 instance 聚合（同服务多实例，只发一条）**
```yaml
- alert: ServiceDown
  expr: up{job="order"} == 0
  # Prometheus 会自动聚合所有 down 的 instance
  # 告警内容：3 个实例不可用
```

**2. 时间窗口聚合（避免抖动）**
```yaml
- alert: HighErrorRate
  expr: error_rate > 0.05
  for: 2m  # 持续 2 分钟才告警，避免瞬时抖动
```

**3. 变化率告警（优于绝对值）**
```yaml
# ❌ 不好：基于绝对值，阈值难定
- alert: HighOrderCount

# ✅ 好：基于变化率，异常更明显
- alert: OrderCountSpike
  expr: |
    rate(orders_total[5m]) 
    > rate(orders_total[5m] offset 1h) * 1.5
```

---

## 告警疲劳治理

### 问题现象

- 告警太多 → 工程师麻木 → 真正告警被忽视
- 告警不准 → "狼来了" → 告警被静音
- 告警没有上下文 → 排查慢 → 告警堆积

### 治理策略

**1. 告警分级 + 静默规则**

```yaml
# 周末低峰期静默普通告警
- name: weekend-silence
  match:
    severity: warning
  time_intervals:
    - times:
        - start_time: '20:00'
          end_time: '09:00'
      weekdays: ['saturday', 'sunday']
```

**2. 告警质量评审（每周）**

```
告警健康度指标：
├── 触发率：每天告警数 / 服务数
├── 准确率：真告警 / 总告警（需人工标记）
├── 响应时间：告警到首次处理的时间
└── 解决率：当天解决的告警 / 总告警

目标：
- 每天每个服务 < 5 条告警
- 准确率 > 80%
- P0 响应时间 < 5 分钟
```

**3. 告警内容规范化**

```yaml
# ❌ 不好的告警
summary: "Error"

# ✅ 好的告警（包含上下文）
annotations:
  summary: "订单服务 5xx 错误率 > 5%"
  description: |
    服务: order-service
    环境: production
    当前错误率: {{ $value | humanizePercentage }}
    过去 1 小时错误数: {{ $values }}
    关联 TraceID: {{ $labels.trace_id }}
    排查链接: http://grafana/dashboard?var-service=order
```

**4. On-Call 轮值制度**

```go
// 告警分配算法
type OnCallScheduler struct {
    engineers []Engineer
    current   int
}

func (s *OnCallScheduler) GetOnCall() *Engineer {
    return s.engineers[s.current%len(s.engineers)]
}

func (s *OnCallScheduler) Rotate() {
    s.current++
}
```

---

## SLO/SLI 与告警设计

### 核心概念

| 概念 | 说明 | 示例 |
|------|------|------|
| **SLI** | 服务质量指标 | 请求成功率 99.9% |
| **SLO** | 目标值 | 月错误预算 0.1% |
| **Error Budget** | 允许的错误量 | 月 43 分钟 |

### 基于 Error Budget 的告警

```yaml
# 基于 burn rate 的告警（比传统阈值更准确）
# 1 小时内消耗 100% 日错误预算 → 立即告警
- alert: ErrorBudgetBurn
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[1h])) / 3600
    > 0.1 * 0.001 / 24  # 1h burn rate
  labels:
    severity: critical

# 6 小时内消耗 100% 日错误预算 → 告警
- alert: ErrorBudgetSlowBurn
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[6h])) / 21600
    > 0.1 * 0.001 / 24
  labels:
    severity: warning
```

---

## 常见问题

### Q：告警 "for" 参数设置多少合适？

> `for` 参数控制告警触发延迟。生产建议：
> - Critical：`for: 1m`（等 1 分钟避免抖动）
> - Warning：`for: 2-5m`（给服务恢复时间）
> - 资源类：`for: 5-10m`（资源变化较慢）

### Q：如何避免告警风暴？

> **收敛**：AlertManager 按服务+告警类型聚合；**分级**：Critical 立即通知，Warning 汇总通知；**静默**：维护窗口期间自动静默；**质量评审**：定期清理无效告警。

### Q：告警和监控的区别？

> **监控**是持续采集数据（what is happening），**告警**是触发阈值后的通知（when to act）。好的监控是告警的基础，没有准确的 SLI/SLO 定义，告警就是瞎告。

### Q：Go 服务特有的告警指标有哪些？

> Goroutine 数量（泄漏检测）；GC 频率和耗时（stop-the-world）；内存分配速率；P99 延迟（`histogram_quantile`）；连接池使用率（DB、Redis）；goroutine blocked 数量。
