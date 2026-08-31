# Go 金丝雀发布（Canary Deployment）与无损上线

> 考察频率：★★★★☆  优先级：P1
> 关键词：灰度发布、服务权重、readiness probe、SIGTERM 优雅关闭、连接摘流、熔断回滚

## 面试官考察意图

**"新版本上线出问题怎么办？"** 是高级岗位必问的工程化问题。这道题考的不是"我按个按钮就部署了"，而是候选人有没有设计过**可灰度、可观测、可快速回滚**的发布流程。

高级工程师要能说清楚从流量切分 → 指标监控 → 自动/手动决策 → 快速回滚的完整闭环。

---

## 核心答案（30 秒版）

金丝雀发布本质上是**把新版本逐步放量、持续观察**。在 K8s 中通过两个 Deployment（old + new）+ Service 权重控制来实现。**第一步部署新实例但只加到 Service（不摘流）**，等 readiness probe 通过后手动/自动将流量切 10% 观察指标（错误率、延迟），正常后逐步放大到 50%/100%，异常则立即停止放量或回滚到旧版本。配合 **preStop sleep + SIGTERM 处理**实现请求处理的无损摘流。

---

## K8s 金丝雀发布架构

### 双 Deployment + Service 模式

```yaml
# old-deployment.yaml — 当前生产版本
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service-old
spec:
  replicas: 9
  template:
    spec:
      containers:
      - name: my-service
        image: registry.example.com/my-service:v1.2.0
        ports:
        - containerPort: 8080

---
# new-deployment.yaml — 新版本（初始副本数为 0）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service-new
spec:
  replicas: 0   # 初始为 0，随发布流程逐步增加
  template:
    spec:
      containers:
      - name: my-service
        image: registry.example.com/my-service:v1.3.0-rc1
        ports:
        - containerPort: 8080
        readinessProbe:      # 关键：健康检查
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        lifecycle:           # 关键：优雅关闭
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 30"]  # 等待请求处理完
```

### Service 权重控制（两种方式）

#### 方式 A：Service Selector（适合 2 个版本）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-service
    version: canary  # 切换这里实现权重分配
  ports:
  - port: 80
    targetPort: 8080
---
# 全量切换到新版本
kubectl label deployment my-service-old app=my-service-version-selector=false
kubectl label deployment my-service-new app=my-service,version=canary --overwrite=true

# 50% 灰度（用 Weighted Round Robin）
kubectl scale deployment my-service-old --replicas=5
kubectl scale deployment my-service-new --replicas=5
```

#### 方式 B：Istio VirtualService（精确流量切分）

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service
spec:
  hosts:
  - my-service.default.svc.cluster.local
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"    # 带 header 的请求走新版本
    route:
    - destination:
        host: my-service-new
        weight: 10       # 10% 固定比例
  - route:
    - destination:
        host: my-service-old
        weight: 90
---
# 升级流量比例（通过修改 YAML 重新应用即可）
# kubectl apply -vsv-v1.yaml   # 改为 weight: 50/50
# kubectl apply -vsv-v2.yaml   # 改为 weight: 100/0 = 全量
```

---

## Go 侧配合：SIGTERM 优雅关闭

### 正确实现 graceful shutdown

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    srv := &http.Server{
        Addr:    ":8080",
        Handler: router(),
    }

    // 启动 HTTP server goroutine
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            slog.Error("server error", "error", err)
            os.Exit(1)
        }
    }()

    // 监听终止信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit

    slog.Info("shutdown signal received, draining connections...")

    // 优雅关闭：最多给 30 秒
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        slog.Error("forced shutdown", "error", err)
    }

    slog.Info("server exited cleanly")
}

// handler 中正确处理超时取消
func OrderHandler(w http.ResponseWriter, r *http.Request) {
    // r.Context() 会在 server.Shutdown() 时被 cancel
    select {
    case <-r.Context().Done():
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    default:
        // 正常处理逻辑
    }
    
    result, err := processOrder(r.Context())
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    json.NewEncoder(w).Encode(result)
}
```

### 为什么需要 preStop sleep？

```
┌─────────────────────────────────────────────┐
│ Kubernetes 发送 SIGTERM                       │
│                     ↓                        │
│ preStop sleep 30s                            │
│ ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐            │
│ │  ◈  ◈  ◈  ◈  ◈  ◈  ◈  ◈  ◈  ◈ │  ← 继续处理已有请求
│ └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘            │
│                     ↓                        │
│ SIGKILL（强制杀死 pod）                       │
└─────────────────────────────────────────────┘

没有 preStop: pod 被瞬间杀死 = 用户收到 502
有 preStop:  给 HTTP server 时间完成正在处理的请求
```

---

## 金丝雀发布流程（Checklist）

```bash
# Step 1: 部署新版本到集群（replicas=0，不出现在 Service 中）
kubectl apply -f new-deployment.yaml

# Step 2: 确认 readiness probe 通过
kubectl get pods -l app=my-service-new
# NAME                          READY   STATUS    RESTARTS
# my-service-new-abc123-def456  1/1     Running   0

# Step 3: 切 10% 流量
kubectl apply -f virtualservice-10pct.yaml
# 或者直接修改 Deployment replica 数
kubectl scale deployment my-service-old --replicas=9
kubectl scale deployment my-service-new --replicas=1

# Step 4: 观察 5~10 分钟
#   - 错误率 < 0.1%
#   - TP99 < SLA 阈值
#   - CPU/Memory 无异常增长
promtail dashboard | grep "my-service-new.*error_rate"

# Step 5: 放大到 50%
kubectl scale deployment my-service-old --replicas=5
kubectl scale deployment my-service-new --replicas=5

# Step 6: 再观察，确认稳定
# 然后放大到 100%

# Step 7: 清理旧版本
kubectl delete deployment my-service-old
```

---

## 故障回滚策略

```go
// 自动化回滚判断逻辑
func shouldRollback(metric string, threshold float64) bool {
    current := fetchMetricValue(metric)
    return current > threshold
}

// K8s 回滚命令
kubectl rollout undo deployment/my-service-new

// Istio 流量回退
kubectl edit virtualservice my-service
# 把所有 weight 改回 old 版本
```

**回滚触发条件清单：**
| 指标 | 阈值 | 动作 |
|------|------|------|
| HTTP 5xx 错误率 | > 1% | 立即暂停放量 |
| HTTP 5xx 错误率 | > 5% | 自动回滚 |
| TP99 延迟 | > SLA × 2 | 暂停放量 |
| Pod OOM Kill | 任意次数 | 自动回滚 |
| P99 DB 查询耗时 | > baseline × 3 | 暂停放量 |

---

## 面试话术

> "我们的发布流程是：先以 0 副本部署新版本确认 readiness probe 通过，然后通过 Istio VirtualService 把 10% 流量切到新实例，观察 10 分钟确认错误率和延迟正常后，逐步放大到 50%、100%。如果任何阶段发现 5xx 超过 5% 直接一键回滚。Go 服务侧配合实现 SIGTERM 优雅关闭——收到信号后停止接收新请求，但处理完已有的请求才退出。整个发布过程对用户完全无感知。"

---

[🏠 首页](../../../README.md) · [📦 工程素养](../README.md) · [🛡️ 技术领导力](./README.md)
