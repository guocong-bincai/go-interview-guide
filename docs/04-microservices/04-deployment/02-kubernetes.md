[🏠 首页](../../../README.md) · [📦 微服务工程](../README.md) · [☸️ Kubernetes](../README.md)

---

# Kubernetes 核心：Pod 调度、HPA、滚动发布

## 面试官考察意图

考察候选人对 Kubernetes 生产使用的深度理解。
5~8 年工程师不仅要会用 `kubectl`，还要能讲清楚**调度决策如何生效、HPA 如何扩缩容、滚动更新如何保证零宕机**，以及线上踩过的坑。

---

## 核心答案（30 秒版）

Kubernetes 调度核心流程：**API Server 接收 Pod → Scheduler 预选/打分 → 绑定到 Node → Kubelet 实际创建**。

- **调度策略**：预选（Predicates）过滤不合规节点，打分（Priorities）选最优
- **HPA**：基于 CPU/内存/自定义指标，自动调整 ReplicaSet，目标 Pod 数量 = ceil(当前CPU% / target%)
- **滚动更新**：逐步替换旧版本 Pod，maxSurge（多扩）+ maxUnavailable（少停）控制节奏，默认为 25%

---

## 深度展开

### 1. Pod 调度机制

#### 1.1 调度流程

```
用户提交 Pod (kubectl apply)
         │
         ▼
    API Server 写入 etcd (status.phase=Pending)
         │
         ▼
    Scheduler Watch 到新 Pod，开始调度
         │
         ├─→ 预选阶段（Predicates）：过滤硬性约束
         │   ├── PodFitsResources：CPU/内存是否足够
         │   ├── HostPorts：端口是否冲突
         │   ├── MatchNodeSelector：nodeSelector/label
         │   └── NoDiskConflict：存储卷是否冲突
         │
         ├─→ 打分阶段（Priorities）：选择最优节点
         │   ├── LeastRequestedPriority：空闲资源越多分数越高
         │   ├── BalancedResourceAllocation：CPU/内存均衡
         │   ├── ImageLocality：镜像已存在的节点加分
         │   └── PodAffinity/AntiAffinity：亲和性/反亲和
         │
         ▼
    选中节点，API Server 更新 Pod spec.nodeName
         │
         ▼
    Kubelet 定期 syncPod，发现分配到本节点的 Pod
         │
         ▼
    创建 pause 容器（网络栈）→ 业务容器 → CNI 配置网络
```

#### 1.2 高级调度策略

```yaml
# nodeSelector：固定调度到特定标签节点
nodeSelector:
  zone: beijing

# 节点亲和性（软硬结合，比 nodeSelector 更灵活）
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values: [beijing-1a]
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        preference:
          matchFields:
            - key: metadata.name
              operator: In
              values: [node-high-memory-1]

# Pod 亲和性：与特定 Pod 调度到同一区域（减少延迟）
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: redis
      topologyKey: topology.kubernetes.io/zone

# Pod 反亲和：Pod 分布在不同节点（提高可用性）
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: gateway
    topologyKey: kubernetes.io/hostname
```

**生产场景：**
- 有状态服务（Redis、MySQL）用 `podAntiAffinity` 强制打散到不同节点
- 中间件用 `nodeAffinity` 调度到高配物理机
- 敏感服务用 `taints + tolerations` 独占节点池

#### 1.3 污点与容忍（Taints & Tolerations）

```bash
# 给专用节点打污点：NoSchedule/PreferNoSchedule/NoExecute
kubectl taint nodes high-mem-node memory-intensive=true:NoSchedule

# Pod 必须有对应 toleration 才能调度上去
tolerations:
  - key: "memory-intensive"
    operator: "Exists"
    effect: "NoSchedule"
```

**典型场景：**
- 专用 GPU 节点：`kubectl taint nodes gpu-1 nvidia.com/gpu=true:NoSchedule`
- 运维节点允许所有 Pod：`kubectl taint nodes infra-node dedicated=true:NoExecute`
- 节点故障时 `NoExecute` 立即驱逐未容忍 Pod，`NoSchedule` 则不调度新 Pod

#### 1.4 调度问题排查

```bash
# 查看 Pod 调度失败原因
kubectl describe pod <pod-name> | grep -A10 Events

# 常见原因
# 1. 资源不足：NoNodesAvailable → 调整 resource.limits 或扩节点
# 2. 亲和性冲突：node(s) didn't match pod affinity/selector
# 3. 污点未容忍：node(s) had taints that the pod didn't tolerate

# 强制调度到指定节点（不推荐生产使用）
kubectl label node <node> dedicated=game-server --overwrite
kubectl run nginx --image=nginx --dry-run=client -o yaml | \
  kubectl apply -f -
# 在 yaml 中加 nodeName 字段强制绑定
```

---

### 2. HPA（Horizontal Pod Autoscaler）

#### 2.1 核心原理

```
指标采集（Metrics Server）
      │
      ▼
HPA Controller 计算：
  desiredReplicas = ceil(currentReplicas * currentMetricValue / targetMetricValue)
      │
      ▼
更新 Deployment/ReplicaSet 的 replicas
      │
      ▼
Deployment Controller 创建/删除 Pod
```

**默认指标：**
- CPU：Pod 当前 CPU 使用 / requests CPU（80% 阈值 = 5个Pod需扩到 7个）
- 内存：超过内存 requests 的百分比（注意：内存无法缩容到低于 requests）

#### 2.2 HPA 实战配置

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
    # CPU 指标
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    # 内存指标（注意：内存通常只建议设扩容上限，不建议设缩容下限）
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 500Mi
    # 自定义指标（需要 Prometheus Adapter）
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1k"  # 每 Pod 1000 QPS
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # 扩容窗口：30秒内不重复扩容
      policies:
        - type: Percent
          value: 100                    # 最多一次扩 100% Pod
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容冷却 5 分钟
      policies:
        - type: Pods
          value: 2                      # 最多一次缩 2 个 Pod（保护容量）
          periodSeconds: 60
```

#### 2.3 生产常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 扩容滞后（流量突增时雪崩） | stabilizationWindowSeconds 太长 | 设短扩容窗口 + Prometheus 实时指标 |
| 缩容过快（流量恢复时 Pod 不够） | 缩容窗口太短 | 增加 minReplicas / 延长缩容窗口 |
| 内存 HPA 缩容失败 | requests.memory 设置过低 | 合理设置 requests，让内存使用率有参考价值 |
| HPA 不生效 | Metrics Server 未安装 | `kubectl top pods` 确认指标可采集 |

#### 2.4 Go 1.25 Container-aware GOMAXPROCS（重点）

> Go 1.25 新增 · Kubernetes 生产高频踩坑点 · 面试区分度极高

**面试考察意图：**
考察候选人对 Go 运行时在容器环境中行为的深度理解。
Go 1.25 之前，GOMAXPROCS 默认等于宿主机逻辑 CPU 核数，在 Kubernetes 中设置 CPU limit 后，Go 程序会创建过多线程导致调度开销增大、绑核失败。这是 5~8 年工程师在容器化部署时**几乎必踩的坑**，高级工程师要能讲清楚问题根因和解决路径。

**核心答案（30 秒版）：**

Go 1.25 之前，GOMAXPROCS 默认等于宿主机逻辑核数，与容器 CPU limit 无关。在 K8s 中设置 `resources.limits.cpu=500m` 后，Go 依然会用满宿主机的所有核，造成**过度调度和性能下降**。Go 1.25 实现了 cgroup 感知：默认 GOMAXPROCS 会自动匹配容器 CPU bandwidth limit，且支持动态更新。

```bash
# 问题现场：Go 1.23 进程，K8s 设置 2核 limit，但 GOMAXPROCS = 32（宿主机核数）
$ kubectl exec -it order-service-xxxx -- cat /proc/1/cmdline | tr '\0' ' '
# GOMAXPROCS=32，但容器 CPU limit 只有 2 核
# 后果：线程在 2 核上争抢，上下文切换剧烈，延迟飙升

# Go 1.25+：运行时自动读取 cgroup CPU bandwidth limit
$ cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us   # 200000 → 2核
$ cat /sys/fs/cgroup/cpu/cpu.cfs_period_us  # 100000
# GOMAXPROCS 自动设为 2，无需手动设置
```

**深度展开：**

##### 1. 问题根因（Go 1.25 之前）

Go 的 GMP 调度器中，P（Processor）的默认数量由 `gomaxprocs` 控制：

```go
// Go 1.24 及之前：gomaxprocs 默认 = runtime.NumCPU()
// NumCPU() 返回的是宿主机逻辑核数，与容器 cgroup 限制无关

// 例如：32 核宿主机，K8s 容器 limit=2 核
// GOMAXPROCS = 32，但实际只有 2 核可用
// → 创建 32 个 M（线程），在 2 核上争抢
// → 线程上下文切换开销剧增，P99 延迟可差 3~5 倍
```

**生产中的典型症状：**

| 场景 | 表现 |
|------|------|
| 高并发 API 服务 | CPU limit 内的请求延迟正常，但超出 limit 后雪崩 |
| gRPC 流处理 | 线程数过多，绑核失败，服务不稳定 |
| 定时任务 | CPU throttle 严重，任务执行时间不可预测 |

**解决方案（Go 1.25 之前）：**

```yaml
# 方案1：手动设置 GOMAXPROCS（需人工计算，易出错）
env:
  - name: GOMAXPROCS
    valueFrom:
      downwardAPI:
        fieldRef:
          fieldPath: metadata.annotations['cpu-limit']
# 但 Downward API 不直接支持 cpu.limit，需要自己计算或用 sidecar

# 方案2：使用入口脚本自动感知（社区常见做法）
# entrypoint.sh:
# GOMAXPROCS=$(grep cgroup /proc/self/cgroup | cut -d: -f3 | xargs -I{} cat /sys/fs/cgroup/cpu/{}/cpu.cfs_quota_us 2>/dev/null | awk '{if($1>0)printf "%d", $1/100000; else print 0}')
# [ -n "$GOMAXPROCS" ] && [ "$GOMAXPROCS" -gt 0 ] && export GOMAXPROCS
```

##### 2. Go 1.25 的解决方案

Go 1.25 运行时在启动时自动读取 cgroup CPU bandwidth 信息：

```
# cgroup v1（多数 K8s 集群）
/sys/fs/cgroup/cpu/cpu.cfs_quota_us ÷ cpu.cfs_period_us = CPU 核数
# 例如：quota=200000, period=100000 → 2 核

# cgroup v2（新版 K8s，如 K8s 1.25+）
/sys/fs/cgroup/cpu.max:
# 格式：max 或 "quota period"
# 例如："200000 100000" → 2 核
```

**两大行为变化：**

1. **启动时自动匹配**：GOMAXPROCS 默认 = cgroup CPU bandwidth limit
2. **运行时动态更新**：如果 cgroup limit 变化（如 K8s HPA 动态调整），GOMAXPROCS 会自动更新

```go
// Go 1.25 新增 GODEBUG 控制
// containermaxprocs=0：禁用容器感知（回退到宿主机核数）
// updatemaxprocs=0：禁用运行时动态更新

// 典型使用场景：
// - 长期稳定的 CPU limit → 默认行为即可
// - 需要动态调整的场景 → 确保 updatemaxprocs 未被禁用（默认开启）
```

##### 3. 面试高频追问

**Q：手动设置 GOMAXPROCS 会覆盖 Go 1.25 的自动感知吗？**

会。如果设置了 `GOMAXPROCS` 环境变量，运行时不会自动读取 cgroup 信息。需要用 `GODEBUG=containermaxprocs=0` 并手动计算才能覆盖。

**Q：cgroup v1 和 cgroup v2 有区别吗？**

Go 1.25 两个版本都支持。读取路径不同（cgroup v1 用 `cpu.cfs_quota_us`，cgroup v2 用 `cpu.max`），运行时自动适配。

**Q：Go 1.25 的 container-aware 对已运行的 Pod 有效吗？**

如果 `updatemaxprocs=1`（默认），运行时每 10 秒检查一次 cgroup limit 变化并更新 GOMAXPROCS，不需要重启 Pod。

**Q：这和 Java/JVM 的 cgroup 感知有什么异同？**

JVM 长期有 `-XX:+UseContainerSupport`，但 JVM 默认也是宿主机核数。Go 1.25 是首个将此作为默认行为的主流语言运行时，且支持动态更新，优于 JVM 的静态感知。

---

### 3. 滚动更新（RollingUpdate）

#### 3.1 策略参数

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%       # 最多超出期望 Pod 数（可超开）
      maxUnavailable: 25%  # 最多不可用（保证最低服务能力）
```

**实际效果（replicas=4）：**
- `maxUnavailable=0, maxSurge=1`：一批只替换 1 个，始终有 4 个可用
- `maxUnavailable=1, maxSurge=0`：一批替换 1 个，但总数会降到 3
- `maxUnavailable=25%`：一批最多替换 1 个（ceil(4*0.25)=1）

#### 3.2 滚动更新流程

```
旧 Pod: 4/4 Ready
       │
       ▼ (开始滚动)
新 Pod-1 创建 → Ready → 旧 Pod-1 终止
       │
       ▼ (继续滚动)
新 Pod-2 创建 → Ready → 旧 Pod-2 终止
       │
       ▼ (直到完成)
新 Pod: 4/4 Ready
```

**如何保证零宕机：**
1. **就绪探针（ReadinessProbe）**：新 Pod Ready 才加入 Service 端点
2. **就绪 Gate**：所有 Pod Ready 才允许流量切换
3. **优雅终止**：Pod 删除前先从 Endpoints 摘除，再等 `terminationGracePeriodSeconds`（默认 30s）

#### 3.3 灰度发布（Canary）

```yaml
# v1 版本
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-v1
spec:
  replicas: 4
  selector:
    matchLabels:
      app: order-service
      version: v1

---
# v2 canary 版本（只放 10% 流量）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-v2
spec:
  replicas: 1   # 1/5 的 Pod 数量 ≈ 20% 流量
  selector:
    matchLabels:
      app: order-service
      version: v2
```

```yaml
# Istio VirtualService 精确控制
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service-v1
            subset: stable
          weight: 90
        - destination:
            host: order-service-v2
            subset: canary
          weight: 10
```

#### 3.4 滚动更新失败处理

```bash
# 查看滚动状态
kubectl rollout status deployment/order-service

# 查看历史版本
kubectl rollout history deployment/order-service

# 回滚到上一版本
kubectl rollout undo deployment/order-service

# 回滚到指定版本
kubectl rollout undo deployment/order-service --to-revision=3

# 手动暂停滚动（适合分批验证）
kubectl rollout pause deployment/order-service
kubectl rollout resume deployment/order-service
```

---

## 高频追问

**Q：为什么 HPA 建议用 KEDA 而不是原生 HPA？**

KEDA 支持更多指标源（Kafka lag、Redis 队列长度、Prometheus 自定义指标），可以实现**事件驱动缩容**，而原生 HPA 主要基于 CPU/内存，粒度不够精准。典型场景：Kafka Consumer 滞后时扩 Pod，比 CPU 飙升更早响应。

**Q：Pod 调度到特定节点后，节点宕机会发生什么？**

Pod 进入 `Terminating` 状态，Kubernetes 在其他节点创建替代 Pod（如果 Deployment replicas > 1）。通过 `PodDisruptionBudget` 可以限制同时不可用的 Pod 数量，保证业务连续性。

**Q：maxUnavailable=0 和 maxSurge=0 有什么区别？**

两者不能同时为 0。`maxUnavailable=0` 意味着所有旧 Pod 必须等新 Pod Ready 才能删除，保证服务不中断；`maxSurge=0` 意味着总数不能超过期望值，无法快速扩容应对突发流量。

**Q：滚动更新期间如何做 A/B 测试？**

用两个 Deployment（v1/v2）共用一个 Service selector，通过 Istio VirtualService 或 nginx-ingress 的 `canary weight` 控制流量比例。监控两套版本的错误率和延迟，达标后逐步切换。

---

## 延伸阅读

- [Kubernetes 调度器原理](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/)
- [HPA v2 官方文档](https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale/)
- [滚动更新策略](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/#strategy)

---

**[← 上一篇：告警规则设计](./03-alerting.md)**
