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
