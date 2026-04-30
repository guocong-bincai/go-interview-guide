# 灰度发布实战

> 考察频率：★★★★★  优先级：P0
> 关键词：金丝雀、蓝绿、灰度流量、ABTest、Istio、Database Schema 升级、快速回滚

[🏠 首页](../../../README.md) · [📚 微服务模块](../../04-microservices/04-deployment/README.md)

---

## 面试官考察意图

这道题考察候选人**软件交付风险控制**的综合能力，是区分"发布只是改配置"和"发布是可控制的工程过程"的关键。
- 初级工程师只会讲"蓝绿部署"，讲不清灰度策略的边界（数据库变更怎么办？）
- 高级工程师要能讲清楚 **灰度策略分类、金丝雀流量控制、数据库灰度升级、ABTest 分流、快速回滚**，并能结合生产事故讲清楚为什么灰度策略设计失误会导致故障
- 加分项：能讲清楚如何用 **Service Mesh（Istio）实现无代码侵入灰度**

---

## 核心答案（30 秒版）

灰度发布的核心是**将新版本风险的爆炸半径控制住**：

| 策略 | 原理 | 适用场景 | RTO |
|------|------|---------|-----|
| **蓝绿** | 两套环境切流量 | 大版本、风险高 | 秒级（切 DNS）|
| **金丝雀** | 切 1%~5% 流量到新版本 | 大规模系统、高风险 | 分钟级 |
| **灰度** | 按用户/地区/功能逐步放量 | 中高风险 | 分钟级 |
| **ABTest** | 不同用户看到不同版本 | 功能验证 | 小时级 |

**关键工程能力：** 灰度不只是一段代码，而是流量 + 监控 + 回滚三位一体。

---

## 深度展开

### 1. 灰度策略全分类

#### 1.1 蓝绿部署（Blue-Green）

```
  流量 ──→ [负载均衡] ──→ 蓝（v1） 生产实例
                      └──→ 绿（v2） 待命实例
                          
  切换：LB 指向绿，v2 全量上线
  回滚：LB 指回蓝
```

**优点：** 切换快，环境隔离，RTO 秒级
**缺点：** 资源翻倍，两套环境成本高

**Go 生产示例（蓝绿切换管理器）：**
```go
type BlueGreenSwitch struct {
    blue *url.URL // v1
    green *url.URL // v2
    active atomic.String // "blue" | "green"
}

func (b *BlueGreenSwitch) Switch(target string) error {
    // 1. 验证目标环境健康
    if !b.healthCheck(target) {
        return fmt.Errorf("target %s health check failed", target)
    }
    // 2. 切换负载均衡指向
    if err := b.updateLBTarget(target); err != nil {
        return err
    }
    b.active.Store(target)
    return nil
}

func (b *BlueGreenSwitch) Rollback() error {
    current := b.active.Load()
    target := map[string]string{"blue": "green", "green": "blue"}[current]
    return b.Switch(target)
}
```

#### 1.2 金丝雀部署（Canary）

```
  99% 流量 ──→ v1 实例组
   1% 流量 ──→ v2 金丝雀实例
  
  监控：v2 错误率 < v1 → 逐步放量 5% → 10% → 100%
```

**金丝雀核心指标：**
```go
// 金丝雀监控指标（简化版）
type CanaryMetrics struct {
    v1ErrorRate   float64
    v2ErrorRate   float64
    v1P99Latency  time.Duration
    v2P99Latency  time.Duration
}

func (m *CanaryMetrics) IsHealthy() bool {
    // v2 错误率不能比 v1 高太多（+0.5%）
    if m.v2ErrorRate > m.v1ErrorRate+0.005 {
        return false
    }
    // v2 P99 延迟不能比 v1 慢太多（+20%）
    if m.v2P99Latency > m.v1P99Latency*1.2 {
        return false
    }
    return true
}
```

#### 1.3 灰度策略进阶

| 灰度维度 | 实现方式 | 示例 |
|---------|---------|------|
| 按用户 ID | Header/Cookie + 路由规则 | 员工内测 100% 先看新版本 |
| 按地区 | 地理位置路由 | 北京先发，确认后扩全量 |
| 按功能 | Feature Flag | 灰度某个功能按钮（不开通则隐藏）|
| 按流量百分比 | 随机 hash | 随机 5% 用户走 v2 |

**Go Feature Flag 简单实现：**
```go
type FeatureFlag struct {
    enabledUsers map[string]struct{} // 白名单用户
    percent      int                  // 百分比 0-100
}

func (f *FeatureFlag) IsEnabled(userID string) bool {
    if _, ok := f.enabledUsers[userID]; ok {
        return true // 白名单优先
    }
    h := fnv32(userID) % 100
    return h < f.percent
}

// 使用：业务代码中判断
if featureFlag.IsEnabled(r.Header.Get("X-User-ID")) {
    // 新逻辑
} else {
    // 老逻辑
}
```

---

### 2. Istio 灰度：Service Mesh 无侵入方案

#### 2.1 原理

Istio 通过 **VirtualService + DestinationRule + Gateway** 实现流量控制，不需要改业务代码。

```yaml
# Istio VirtualService：按权重灰度
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service
spec:
  hosts:
  - my-service
  http:
  - route:
    - destination:
        host: my-service
        subset: v1
      weight: 95      # 95% 流量走 v1
    - destination:
        host: my-service
        subset: v2
      weight: 5       # 5% 流量走 v2（金丝雀）

---
# DestinationRule：定义 subset（对应 K8s Service Selector）
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-service
spec:
  host: my-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL  # mTLS
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

#### 2.2 Istio 灰度 Go SDK 集成（生产示例）

```go
// 使用 Istio Go SDK 判断当前流量版本
import (
    "istio.io/client-go/pkg/apis/networking/v1beta1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    restconfig "k8s.io/client-go/rest"
)

func GetTrafficWeight(serviceName string) (v1Weight, v2Weight int, err error) {
    config, err := restconfig.InClusterConfig()
    if err != nil {
        return 0, 0, err
    }
    clientset, err := kubernetes.NewForConfig(config)
    
    // 获取 VirtualService
    vsClient := clientset.NetworkingV1beta1().VirtualServices("istio-system")
    vs, err := vsClient.Get(context.Background(), serviceName, metav1.GetOptions{})
    if err != nil {
        return 0, 0, err
    }
    
    // 解析 weight（简化版，假设只有两个 subset）
    for _, httpRoute := range vs.Spec.Http {
        for _, dest := range httpRoute.Route {
            switch dest.Destination.Subset {
            case "v1":
                v1Weight = int(dest.Weight)
            case "v2":
                v2Weight = int(dest.Weight)
            }
        }
    }
    return v1Weight, v2Weight, nil
}
```

#### 2.3 不用 Istio 的轻量灰度方案：Nginx

```nginx
# Nginx 按权重分流（不用 Istio 时的替代方案）
upstream backend {
    server 10.0.0.1:8080 weight=95;  # v1
    server 10.0.0.2:8080 weight=5;   # v2 金丝雀
}
```

---

### 3. 数据库变更：灰度发布的最大陷阱

#### 3.1 问题本质

**灰度发布失败案例（血的教训）：**
- 某服务 v2 把 `users` 表的 `email` 字段从 `NOT NULL` 改成了可空
- 但 MySQL 在线改字段会锁表，全量流量切走后 MySQL CPU 100%，服务雪崩
- 最终回滚，但数据库已经受损，恢复花了 4 小时

#### 3.2 数据库灰度升级三阶段

**阶段一：兼容旧代码（代码 v1.1）**
```sql
-- 新增 nullable 字段（向后兼容）
ALTER TABLE users ADD COLUMN email_new VARCHAR(255) DEFAULT NULL;

-- 旧代码继续写入 email（旧字段）
-- 新代码同时写入 email + email_new
```

**阶段二：新代码全量上线，迁移数据（代码 v2.0）**
```go
// 离线数据迁移：双写期间，把 email 同步到 email_new
func MigrateEmails(batchSize int) error {
    for {
        rows, err := db.Query(`
            SELECT id, email FROM users 
            WHERE email_new IS NULL AND email IS NOT NULL 
            LIMIT ?
        `, batchSize)
        if err != nil { return err }
        if !rows.Next() { break }
        
        for rows.Next() {
            id, email := rows.Int("id"), rows.Str("email")
            _, err := db.Exec(`UPDATE users SET email_new=? WHERE id=?`, email, id)
        }
        rows.Close()
    }
    return nil
}
```

**阶段三：验证后删除旧字段（代码 v2.1）**
```sql
-- 确认所有流量在 v2.1 后，删除旧字段
ALTER TABLE users DROP COLUMN email;
ALTER TABLE users DROP COLUMN email_new; -- 如果有双写
```

#### 3.3 高危操作清单

| 操作 | 风险等级 | 安全做法 |
|------|---------|---------|
| 添加 NOT NULL 列 | 🔴 极高 | 先加 nullable，数据迁完再加 NOT NULL |
| 加索引 | 🟡 高 | 加索引用 `ALGORITHM=INPLACE, LOCK=NONE`（MySQL 5.6+）|
| 加字段 | 🟢 低 | 直接加，字段变更风险最低 |
| 加默认值 | 🟡 高 | 要看 MySQL 版本，5.6 以下加字段会锁表 |
| 删除字段 | 🔴 极高 | 必须确保所有旧版本代码都不引用该字段 |

---

### 4. 快速回滚机制

#### 4.1 回滚决策树

```
收到告警（错误率飙升 / P99 延迟突增）
    ↓
是否灰度期间？（是）→ 自动/手动回滚流量到旧版本
    ↓
确认是新版本问题（对比新旧版本指标）
    ↓
执行回滚：切换流量 + 标记版本为不可用
    ↓
通知相关团队（PA / 运维）
    ↓
根因分析（48小时内）
```

#### 4.2 Go 自动回滚实现

```go
type GrayReleaser struct {
    router       *TrafficRouter
    metricsPort  *MetricsCollector
    rollbackCh   chan struct{}
}

func (g *GrayReleaser) Start() {
    // 持续监控金丝雀指标
    for {
        metrics := g.metricsPort.Collect()
        
        // 判断是否需要回滚
        if g.shouldRollback(metrics) {
            log.Warn("canary metrics unhealthy, initiating rollback")
            if err := g.router.SetWeight("v2", 0); err != nil {
                log.Error("rollback failed, escalating: %v", err)
            }
            close(g.rollbackCh) // 通知告警系统
            break
        }
        time.Sleep(10 * time.Second) // 每 10 秒检查一次
    }
}

func (g *GrayReleaser) shouldRollback(m *CanaryMetrics) bool {
    // 错误率突增 > 1%
    if m.v2ErrorRate > 0.01 {
        return true
    }
    // P99 延迟超过 500ms
    if m.v2P99 > 500*time.Millisecond {
        return true
    }
    // 核心接口 5xx 率超阈值
    if m.v2ServerErrors > m.v2Total*0.005 {
        return true
    }
    return false
}
```

#### 4.3 回滚后数据库回滚的特殊处理

如果灰度期间做了数据迁移（如新加字段），回滚时**不需要回滚数据库结构**，因为：
- 新字段是 nullable / 有默认值的，不会影响旧版本运行
- 数据迁移（如 `email → email_new`）回滚后可继续双写，不丢数据

**但以下情况需要同步回滚数据库：**
- 灰度期间做了**不可逆操作**（如删字段、删表、改索引）
- 这种操作**严禁灰度期间做**，必须全量验证后才能操作

---

### 5. 灰度发布 SOP（生产操作手册）

#### 5.1 灰度前检查清单

```go
// 灰度发布前检查清单（伪代码）
type PreReleaseChecklist struct {
    checks []CheckItem
}

type CheckItem struct {
    Name    string
    CheckFn func() bool
    Critical bool
}

func (c *PreReleaseChecklist) Run() error {
    failed := []string{}
    for _, item := range c.checks {
        if !item.CheckFn() {
            if item.Critical {
                failed = append(failed, "CRITICAL: "+item.Name)
            }
        }
    }
    if len(failed) > 0 {
        return fmt.Errorf("pre-release check failed: %v", failed)
    }
    return nil
}

// 检查项示例
checks := PreReleaseChecklist{
    checks: []CheckItem{
        {"DB migration backward compatible", checkDBBackwardCompat, true},
        {"Feature flags configured", checkFeatureFlags, true},
        {"Monitoring dashboards ready", checkDashboards, false},
        {"Rollback procedure tested", checkRollbackTested, true},
        {"On-call team notified", checkOncallNotified, false},
    },
}
```

#### 5.2 灰度阶段与监控指标

| 阶段 | 流量比例 | 持续时间 | 关注指标 |
|------|---------|---------|---------|
| 内部测试 | 0%（自身流量）| 30 分钟 | 功能正确性 |
| 金丝雀 | 1-5% | 1-2 小时 | 错误率、P99、核心链路 |
| 放量 | 10% → 30% → 50% → 100% | 每步 30 分钟 | 同上 + 业务指标 |
| 稳定观察 | 全量后 2 小时 | - | 错误率回归正常 |

**每个阶段都要通过才能进入下一阶段，否则立即回滚。**

#### 5.3 灰度期间的监控告警配置

```yaml
# Prometheus 告警规则：灰度期间金丝雀异常检测
groups:
- name: canary-alerts
  rules:
  - alert: CanaryHighErrorRate
    expr: |
      (sum(rate(http_requests_total{version="v2",status=~"5.."}[5m])) 
      / sum(rate(http_requests_total{version="v2"}[5m])))
      > 0.01
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Canary v2 error rate > 1%"

  - alert: CanaryHighLatency
    expr: |
      histogram_quantile(0.99, 
        sum(rate(http_request_duration_seconds_bucket{version="v2"}[5m])) 
        by (le)) > 0.5
    for: 5m
    labels:
      severity: warning
```

---

## 高频追问

### Q1：金丝雀和 ABTest 的本质区别是什么？

**答案：**
- **金丝雀（Canary）：** 技术层面验证，风险控制目的。目标是确认新版本"能工作且性能不降"，用户不知道自己被灰度了。
- **ABTest：** 业务层面验证，效果对比目的。目标是确认新版本"用户更喜欢"，用户明确知道自己在体验新功能。

**实现差异：** ABTest 通常用 UUID/Cookie 标识用户，分桶后固定桶不变（用户第一次访问看到 A，之后一直看到 A）。金丝雀用权重分流，同一用户可能今天走 v1、明天走 v2（只要权重变了）。

### Q2：数据库表结构大改动怎么做灰度？

**答案：** 核心思路是**扩展-迁移-收缩三阶段**，具体参考上文"数据库灰度升级三阶段"。关键原则：
1. 扩展阶段：新加字段，**不能删旧字段**
2. 迁移阶段：双写 + 离线数据同步
3. 收缩阶段：确认所有流量在新版本后，才能删旧字段

### Q3：灰度期间服务发现如何处理版本感知？

**答案：**
- **无 Service Mesh：** 用 `version` label 区分服务实例，客户端通过 Header（`X-Version: v2`）指定要调用的版本
- **有 Service Mesh（推荐）：** Envoy/Istio 在 Sidecar 层自动处理版本路由，业务代码完全无感知

### Q4：灰度过程中如何防止缓存雪崩？

**答案：**
1. 灰度期间**新老版本共用同一套缓存**（不要为每个版本单独建缓存 Key）
2. 如果必须分开，`version` 作为 Key 前缀（如 `user:v1:123` vs `user:v2:123`），避免缓存未命中导致击穿 DB
3. 灰度完成后统一清理旧版本缓存 Key（用 SCAN + DEL，不要阻塞）

---

## 延伸阅读

- [Istio 官方文档：Canary Deployments](https://istio.io/latest/docs/tasks/traffic-management/canary/)
- [CNCF 灰度发布白皮书](https://www.cncf.io/blog/2020/11/12/progressive-delivery-with-service-meshes/)
- [Spinnaker 官方灰度策略](https://spinnaker.io/docs/guides/operator/canary/)
- [MySQL 在线 DDL 最佳实践](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html)
- 《Site Reliability Engineering》第 8 章：发布实践
