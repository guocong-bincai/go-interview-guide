# Kubernetes 控制面深度原理

> 考察频率：★★★★★  优先级：P0
> 关键词：API Server、Controller Manager、Scheduler、Informer、CRD/Operator、Watch、List-Watch

[🏠 首页](../../../README.md) · [📚 微服务模块](../../04-microservices/04-deployment/README.md)

---

## 面试官考察意图

这道题考察候选人对 **Kubernetes 核心设计** 的理解深度，是区分"会用 K8s"和"理解 K8s"的分水岭。
- 初级工程师只会 `kubectl apply` / `kubectl get pods`
- 高级工程师要能讲清楚 **API Server 的请求链路（认证→授权→准入控制）**、**Controller 的控制循环机制**、**Scheduler 的调度策略**、**Informer 的 List-Watch 高效同步机制**，以及 **CRD/Operator 的原理**
- 加分项：能讲清楚 K8s 事故中的定位思路（如某个 Controller 为什么不工作）

---

## 核心答案（30 秒版）

K8s 是一个**控制循环驱动的声明式 API 系统**：

| 组件 | 职责 | 关键机制 |
|------|------|---------|
| **API Server** | 唯一入口，提供 CRUD 和 Watch | 认证→授权→准入→存储 |
| **etcd** | 状态存储，所有资源以 versioned 对象存放 | Watch 驱动通知 |
| **Controller Manager** | 运行所有控制器，执行协调循环 | Reconcile 模式 |
| **Scheduler** | 为 Pod 选择最佳节点 |  predicates + priorities |
| **Kubelet** | 节点 agent，负责容器生命周期 | cAdvisor + CRI |

**核心设计思想：** 任何资源的期望状态写入 etcd，Controller 不断将"实际状态"推向"期望状态"。

---

## 深度展开

### 1. API Server 请求链路（面试高频）

```
Client（kubectl / SDK）
    ↓ HTTP 请求
  认证（Authentication）
    ↓ 验证身份（x509 Cert / ServiceAccount Token / OIDC）
  授权（Authorization）
    ↓ RBAC：Role → Binding → 判定权限
  准入控制（Admission Control）
    ↓ Mutating（修改对象，如设置默认值）→ Validating（校验，如名称规范）
  存储到 etcd（通过 etcd WAL + MVCC）
    ↓
 返回 response + 触发 Watch 事件给所有 Watcher
```

#### 1.1 认证（Authentication）

| 方式 | 场景 | 原理 |
|------|------|------|
| x509 证书 | kubeconfig 文件 | 证书内置 CN（用户名）和 O（组织）|
| ServiceAccount Token | Pod 内访问 API Server | Token 存 Secret，卷挂载进 Pod |
| OIDC | 第三方 IdP（Keycloak/dex） | Token 内含用户身份，与 K8s UserAccount 映射 |

**生产案例：** Pod 里用 Go SDK 访问 K8s API 报错 `Unauthorized`，排查发现 ServiceAccount Token 过期（Token 有 TTL）。解法：K8s 1.20+ Token 不再自动续期，需要在 Pod spec 里显式配置 `automountServiceAccountToken: true`。

#### 1.2 授权（Authorization）—— RBAC

```yaml
# Role：命名空间级别权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: my-ns
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# RoleBinding：将 Role 绑定到 User/SA
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-rolebinding
  namespace: my-ns
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: my-ns
roleRef:
  kind: Role
  name: my-role
  apiGroup: rbac.authorization.k8s.io
```

**高频追问：** 为什么有了 RBAC 还要 Admission Control？
- RBAC 控制"谁能做什么"（请求级别的权限）
- Admission Control 控制"对象是否符合规范"（对象级别的校验和改写），如限制资源配额、禁止privileged容器、注入 sidecar

#### 1.3 准入控制器（Admission Controller）

K8s 1.26+ 内置Admission Controller 约 50 个，按执行顺序分两类：

**Mutating（变更型）** — 在对象写入前修改：
```go
// 伪代码：理解 Pod 安全策略的 Mutating 过程
func mutatePod(pod *core.Pod) *core.Pod {
    // 1. 设置默认 security context
    if pod.Spec.SecurityContext == nil {
        pod.Spec.SecurityContext = &core.PodSecurityContext{}
    }
    // 2. 注入 Istio sidecar（如果启用自动注入）
    if istioEnabled && !podHasSidecar(pod) {
        pod = injectSidecar(pod)
    }
    return pod
}
```

**Validating（校验型）** — 在对象写入前校验：
```go
// 伪代码：理解 LimitRanger 的 Validating 过程
func validatePod(pod *core.Pod) error {
    // 检查资源请求是否在 namespace limits 范围内
    for _, container := range pod.Spec.Containers {
        if container.Resources.Requests.Cpu().Cmp(limitRange.MaxCPU) > 0 {
            return fmt.Errorf("CPU request %s exceeds limit %s", 
                container.Resources.Requests.Cpu(), limitRange.MaxCPU)
        }
    }
    return nil
}
```

---

### 2. Controller Manager：控制循环机制

#### 2.1 控制循环（Control Loop / Reconcile Pattern）

```go
// 理解 K8s Controller 的 Reconcile 模式
type Controller interface {
    // Reconcile 是控制循环的核心：
    // 传入期望状态 key，Controller 确保实际状态匹配期望状态
    Reconcile(key string) error
}

// 简化版 Deployment Controller 逻辑
func (d *DeploymentController) worker() {
    for {
        item := <-d.queue  // workqueue
        key, ok := item.(string)
        if !ok { continue }
        
        // 1. 从 indexer 获取真实对象
        obj, exists, err := d.indexer.GetByKey(key)
        if err != nil || !exists {
            continue
        }
        
        // 2. 调用 Reconcile（核心协调逻辑）
        if err := d.reconcile(obj); err != nil {
            d.queue.AddRateLimited(key) // 失败，重新入队
        } else {
            d.queue.Forget(key) // 成功，遗忘
        }
    }
}

func (d *DeploymentController) reconcile(deployment *apps.Deployment) error {
    // 期望状态：replicas = 3
    desired := int32(3)
    
    // 3. 获取当前状态（实际运行的 ReplicaSet）
    rsList, err := d.getReplicaSets(deployment)
    
    // 4. 找到可用的 RS，计算差值
    available := filterAvailable(rsList)
    diff := desired - available
    
    // 5. 采取行动（扩缩容）
    if diff > 0 {
        return d.scaleUp(deployment, int(diff))
    } else if diff < 0 {
        return d.scaleDown(deployment, int(-diff))
    }
    return nil // 对齐
}
```

**控制循环的核心特点：** 永远不会"完成"，只有"下次再检查"。这保证系统在任何异常后都能自我恢复。

#### 2.2 常用 Controller 及职责

| Controller | 监控资源 | 协调动作 |
|------------|---------|---------|
| Deployment | Deployment | 创建/更新 ReplicaSet |
| ReplicaSet | ReplicaSet | 创建/删除 Pod |
| Node Controller | Node | 标记 NotReady 状态，触发 pod eviction |
| Endpoints Controller | Service + Pod | 更新 Endpoints（服务发现）|
| ServiceAccount Controller | SA | 为新 namespace 创建 default SA |

#### 2.3 生产案例：Deployment 卡在 Progressing 不动

**症状：** `kubectl get deployment` 显示 `STATUS: Progressing`，超过 10 分钟不完成

**排查思路（Controller 视角）：**
```
① 看 ReplicaSet：kubectl get rs -n my-ns
    → ReplicaSet 是否创建？有没有 Pending Pod？
② 看 Pod 事件：kubectl describe pod <pod-name>
    → ImagePullBackOff / Evicted / OOMKilled？
③ 看 Controller 日志：
    kubectl logs -n kube-system deployment-controller-xxx
    → Controller 是否有 Bug？workqueue 是否积压？
④ 看 Scheduler 日志：
    kubectl logs -n kube-system scheduler-xxx
    → Pod 是不是调度不上去？（资源不足/亲和性冲突）
⑤ 看 Kubelet 日志：
    journalctl -u kubelet -n 100
    → 节点本身健康吗？（磁盘满/网络插件问题）
```

---

### 3. Scheduler：调度决策

#### 3.1 调度流程（两阶段）

```
Pod 创建 → API Server 写入 etcd
    ↓ etcd Watch 事件 → Scheduler 收到
阶段一：过滤（Filtering）— 淘汰不满足条件的节点
    → predicates：NodeSelector/Taint/资源是否足够/PodFitsResources
阶段二：评分（Scoring）— 给剩余节点打分
    → priorities：LeastRequestedCPU / BalanceResourceAllocation / NodeAffinity
    → 最高分节点 → bind Pod 到节点 → kubelet 接收并创建
```

#### 3.2 调度策略示例（Go 伪代码）

```go
// predicates 过滤：检查节点资源是否足够
func NodeResourcesFit(pod *core.Pod, node *core.Node) bool {
    allocatable := node.Status.Allocatable
    
    // CPU
    if pod.Requests.Cpu().Cmp(allocatable.Cpu()) > 0 {
        return false
    }
    // Memory
    if pod.Requests.Memory().Cmp(allocatable.Memory()) > 0 {
        return false
    }
    return true
}

// priorities 评分：选择负载最低的节点
func LeastRequestedCPU(pod *core.Pod, node *core.Node) int {
    allocatable := node.Status.Allocatable.Cpu().Value()
    requested := sumPodRequestsCPU(node)
    score := int((allocatable - requested) * 100 / allocatable)
    return score
}
```

#### 3.3 生产问题：Pod 调度不上去（Pending）

**常见原因：**
1. **资源不足** — 所有节点 CPU/Memory 不足，`kubectl describe node` 看 Allocatable
2. **亲和性冲突** — PodAffinity/PodAntiAffinity 条件太严格
3. **Taint/Toleration** — 节点有 Taint（污点），Pod 没有匹配 Toleration
4. **PVC 未绑定** — Pod 用 PVC，但 PVC 没有对应的 PV（StorageClass 匹配问题）

**排查命令：**
```bash
# 1. 看 Pod 为什么 Pending
kubectl describe pod my-pod -n my-ns
# 事件里有 "0/3 nodes are available: 1 Insufficient cpu, 2 node(s) had taints"

# 2. 看节点污点
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# 3. 看调度决策（SchedulingGates，K8s 1.26+）
kubectl get pod my-pod -o jsonpath='{.spec.schedulingGates}'

# 4. 临时绕过调度限制（用于排查）
kubectl patch pod my-pod -n my-ns -p '{"spec":{"tolerations":[{"operator":"Exists"}]}}'
```

---

### 4. Informer 机制：高效的状态同步

#### 4.1 为什么需要 Informer

**问题：** 如果所有 Controller 都直接调用 API Server List/Watch，会把 API Server 打爆（成千上万的 Controller 同时 List）

**解法：** Informer = Local Cache + ListWatch

```
K8s 组件（如 Controller）
    ↓ 共享 Informer（每个 Group/Resource 一个 Informer 实例）
    ↓
Informer 内部：
   ① List：从 API Server 拉取全量数据，填充本地 Cache
  ② Watch：持续监听 API Server，事件驱动更新 Cache
  ③ Indexer：提供基于 Key 的快速查询（Key = namespace/name）
  ④ WorkQueue：事件入队，Controller worker 消费
```

#### 4.2 Informer 工作流程

```go
// 理解 SharedIndexInformer 的核心流程
func (s *SharedIndexInformer) Run(stopCh <-chan struct{}) {
    // 1. List：从 API Server 拉全量数据，填充 store
    listObj, err := s.listerWatcher.List()
    
    // 2. Watch：持续监听 API Server 变更事件
    for {
        watchObj, err := s.listerWatcher.Watch()
        for event := range watchObj.ResultChan() {
            switch event.Type {
            case core.Added, core.Modified, core.Deleted:
                // 3. 事件入队，不阻塞 API Server
                s.queue.Add(event.Object)
            }
        }
    }
}

// Controller 使用 Informer：从不直接访问 API Server
func (c *DeploymentController) addDeploymentHandler(obj interface{}) {
    key, err := cache.MetaNamespaceKeyFunc(obj)
    if err != nil { return }
    
    // 这个 key 对应本地 store 的对象，不会调用 API Server
    c.queue.Add(key)
}
```

**性能关键：** 事件处理函数里**不能阻塞**，否则 Watch 事件堆积。所有慢操作（DB调用、外部API）必须入队后异步处理。

#### 4.3 生产问题：Informer 缓存不同步导致 Bug

**症状：** Controller 创建了资源，但 Watch 事件迟迟没收到，导致重复创建

**排查：** 
```bash
# 查看 API Server 的 Watch 连接数（正常每个 Controller 一个）
curl -s localhost:8001/metrics | grep watcher

# 查看 Informer 重连事件
kubectl logs -n kube-system -l component=kube-controller-manager \
  | grep -i "informer" | tail -20
```

---

### 5. CRD 与 Operator：扩展 K8s

#### 5.1 CRD vs Operator

| 概念 | 定义 | 示例 |
|------|------|------|
| **CRD** | 定义新的资源类型（Custom Resource Definition）| 声明一个 `Foo` 资源 |
| **Operator** | 实现 CRD 的 Controller，自动管理该资源 | 监听 `Foo`，自动创建 Pod |

**类比：**
- CRD = 定义一种新的"数据类型"（如自定义结构体）
- Operator = 给这个数据类型写"方法"（如构造函数、方法实现）

#### 5.2 CRD 示例

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: foos.example.com
spec:
  group: example.com
  names:
    kind: Foo
    listKind: FooList
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
---
# 使用 CRD 创建 Foo 资源
apiVersion: example.com/v1
kind: Foo
metadata:
  name: my-foo
spec:
  replicas: 3
  image: my-app:v2
```

#### 5.3 Operator 实战：使用 controller-runtime

```go
// 用 kubebuilder 生成的 Operator 骨架代码
type FooReconciler struct {
    client.Client   // 读取 K8s 资源
    Scheme *runtime.Scheme
}

// Reconcile 是 Operator 的核心逻辑
func (r *FooReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)
    
    // 1. 获取 Foo 对象
    foo := &examplev1.Foo{}
    if err := r.Get(ctx, req.NamespacedName, foo); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // 2. 调谐逻辑：确保实际状态匹配期望状态
    desiredReplicas := foo.Spec.Replicas
    
    // 获取关联的 Deployment（通过 OwnerReference 或 Labels）
    dep := &apps.Deployment{}
    if err := r.Get(ctx, req.NamespacedName, dep); err != nil {
        // 不存在 → 创建
        dep = r.buildDeployment(foo)
        return ctrl.Result{}, r.Create(ctx, dep)
    }
    
    // 已存在 → 扩缩容到期望值
    if *dep.Spec.Replicas != desiredReplicas {
        *dep.Spec.Replicas = desiredReplicas
        return ctrl.Result{}, r.Update(ctx, dep)
    }
    
    return ctrl.Result{}, nil
}
```

---

## 高频追问

### Q1：etcd 是怎么保证一致性和高可用的？

**答案：** K8s 用 etcd 作为存储后端，etcd 基于 Raft 协议实现强一致性：
- **写操作：** 必须获得集群多数节点（N/2+1）确认才落盘
- **Watch 机制：** 依赖 etcd 的 MVCC + Watch，API Server 收到变更事件后才响应客户端
- **高可用：** etcd 通常 3 节点部署，挂 1 节点可继续服务，挂 2 节点集群不可用

### Q2：Pod 创建的整体流程是什么？

**答案：**
```
kubectl apply → API Server 认证/授权/准入 → 写入 etcd
    ↓ etcd 事件
Scheduler 收到 Pod 事件 → 执行调度（过滤+评分）→ 绑定 Pod 到节点
    ↓ Kubelet 收到 Pod 事件（通过 CRI 接口）
Kubelet → 调用 CRI（containerd / dockershim）→ 创建容器（pause 容器 + 业务容器）
    ↓
Kubelet → 更新 Pod Status 到 API Server
```

### Q3：K8s 1.26+ 有哪些重大变化？

**答案（已发布稳定版）：**
- **Kubelet 的 Pod 资源申请变化：** `PodResources` API 替代 `cadvisor` 指标
- **非确定性对象的排序：** API Server 使用 CRD schema 进行排序，不再依赖字典序
- **调度器新策略：** `NodeResourcesFit` 默认行为变化，更精准的资源感知

---

## 延伸阅读

- [K8s 官方文档：Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [K8s Scheduler 调度策略](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Kubebuilder 官方文档](https://book.kubebuilder.io/)
- [ETCD RAFT 协议动画演示](http://thesecretlivesofdata.com/raft/)
- 《Kubernetes in Action》 第 10 章：调度器深度解析
