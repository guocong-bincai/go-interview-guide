# Kubernetes 控制面深度原理

> 考察频率：★★★★★  优先级：P0

## TODO（待填写）

## 1. 控制面整体架构
- [ ] API Server：所有操作唯一入口，认证 → 授权 → 准入控制链路
- [ ] etcd：K8s 的"数据库"，Watch 机制驱动控制面
- [ ] Controller Manager：控制循环集合（Deployment/ReplicaSet/Node 控制器）
- [ ] Scheduler：Pod 调度决策

## 2. API Server 认证授权链路（面试重点）
- [ ] 认证（Authentication）：x509 / ServiceAccount Token / OIDC
- [ ] 授权（Authorization）：RBAC（Role/ClusterRole/RoleBinding）
- [ ] 准入控制（Admission）：MutatingWebhook → ValidatingWebhook
- [ ] 完整请求链路：kubectl → API Server → etcd

## 3. Controller Manager：Reconcile 模式
- [ ] 核心思想：声明式（期望状态 vs 实际状态），控制器不断对齐
- [ ] Informer 机制：List + Watch → 本地缓存 → 事件队列
- [ ] Deployment 控制器如何管理 ReplicaSet 滚动升级
- [ ] 为什么 K8s 组件崩溃重启后能快速恢复（幂等 Reconcile）

## 4. Scheduler 扩展点（Framework Plugin）
- [ ] 调度流程：Filter（过滤不满足节点）→ Score（打分）→ Bind
- [ ] 常用 Filter 插件：NodeAffinity / PodAntiAffinity / ResourceFit
- [ ] 自定义调度器：实现 Scheduler Plugin 接口

## 5. Kubelet 工作机制
- [ ] Pod 生命周期：Pending → Running → Succeeded/Failed
- [ ] CRI（Container Runtime Interface）：与 containerd 交互
- [ ] CNI（Container Network Interface）：Pod 网络分配
- [ ] CSI（Container Storage Interface）：持久化存储

## 6. CRD + Operator 模式
- [ ] 什么时候需要 Operator（有状态应用的生命周期管理）
- [ ] CRD 自定义资源定义流程
- [ ] controller-runtime 框架实现 Operator
- [ ] 典型案例：TiDB-Operator / MongoDB-Operator

## 7. 生产常见问题
- [ ] Pod 一直 Pending：资源不足 / 节点选择器不匹配 / PVC 未绑定
- [ ] CrashLoopBackOff：内存 OOM / 探针配置错误
- [ ] 滚动升级期间 502：`terminationGracePeriodSeconds` + preStop hook
- [ ] etcd 满了怎么办（compact + defrag）

## 高频追问
- [ ] K8s 的 Deployment 滚动升级和金丝雀发布怎么实现？
- [ ] RBAC 中 Role 和 ClusterRole 的区别？
- [ ] Informer 的 List-Watch 机制为什么比轮询高效？
