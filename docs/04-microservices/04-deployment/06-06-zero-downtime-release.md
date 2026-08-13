# 无损发布：优雅上下线与流量摘除

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：无损发布、优雅上线、优雅下线、readiness 探针、preStop、SIGTERM、连接排空、流量摘除、滚动发布

---

## 面试官考察意图

"你们发布时请求会报错吗？"——这是大厂面试的高频场景题。初级候选人会说"发布很快，影响不大"；高级候选人要能完整讲出**无损发布的闭环**：

1. 发布一个实例时，如何做到**零失败请求**？
2. readiness 探针、preStop hook、SIGTERM、优雅关闭四者如何配合？
3. 长连接（WebSocket/gRPC）和短连接（HTTP）的摘流量方式有何不同？
4. 注册中心模式下（非 K8s），如何摘流？

这道题把 K8s、Go 运行时、网络、运维串在一起，是高级工程师的试金石。

---

## 核心答案（30 秒版）

无损发布 = **先摘流量，再停服务，处理完存量请求后退出**。K8s 下的标准时序：

```text
发布开始（滚动更新，逐个替换实例）
  │
  ▼
① readiness 探针返回 Fail → K8s 从 Service Endpoints 摘除该 Pod（新流量不再进来）
  │
  ▼
② preStop hook 执行（优雅下线脚本：sleep 或通知注册中心注销）
  │
  ▼
③ K8s 发送 SIGTERM → 应用开始优雅关闭（停止接收新请求）
  │
  ▼
④ 应用等待存量请求处理完成（drain，带超时上限）
  │
  ▼
⑤ 进程退出 → SIGKILL 兜底（超时未退出会被强杀）
```

**核心口诀：先摘流量（readiness/preStop），再排空存量（graceful drain），最后退出。**

---

## 深度展开

### 1. 为什么发布会有失败请求？（问题本质）

发布一个 Pod 的瞬间会发生两件事：

| 事件 | 问题 |
|------|------|
| Pod 进程被 kill | 正在处理中的请求**连接被断开** → 客户端 5xx |
| 新 Pod 未就绪但已加入 Endpoints | 流量打到**未就绪实例** → 连接拒绝 |

无损发布要解决的就是这两个时间窗口：**存量请求排空窗口** + **新实例就绪窗口**。

### 2. K8s 无损发布四件套

#### 2.1 readiness 探针：控制"新流量"（上线摘流/放量）

readiness 失败 → Pod 从 Service Endpoints 摘除，但容器**不重启**（区别于 liveness）。

```yaml
readinessProbe:
  httpGet:
    path: /healthz/ready   # 就绪探针：依赖就绪（DB/Redis 连接池预热完成）才返回 200
    port: 8080
  initialDelaySeconds: 5    # 启动后 5 秒才开始探测
  periodSeconds: 5          # 每 5 秒探测一次
  timeoutSeconds: 2
  failureThreshold: 3       # 连续 3 次失败 → 摘除
```

**就绪探针的两个关键用途**：
- **上线**：进程起来了但依赖（DB 连接池、本地缓存）还没就绪时返回 503，避免流量打到"半成品"
- **下线**：应用收到 SIGTERM 前，先让 /healthz/ready 返回 503，K8s 会在 ~5s 内把 Pod 摘出 Endpoints

> 注意：就绪探针摘流有**延迟**（探测周期 5s + 失败阈值 3 = 最长 15s 才摘除），所以还需要 preStop 兜底。

#### 2.2 preStop hook：给摘流留时间

preStop 是 K8s 在发 SIGTERM **之前**执行的钩子，常用于"sleep 等待 Endpoints 摘除生效"或"主动通知注册中心注销"。

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

**为什么要 sleep？** SIGTERM 发出后，LB/客户端可能还在用旧 Endpoints 转发流量（特别是 Nginx 长连接、注册中心缓存）。sleep 10s 给流量调度层一个"冷却"窗口，避免新请求打到正在关闭的实例。

> 面试加分点：sleep 时长 = 探针失败周期上限（periodSeconds × failureThreshold）+ 流量调度层缓存刷新时间（如 Nginx upstream 10s、注册中心 15s）。

#### 2.3 SIGTERM 处理：Go 优雅关闭

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: router}

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // 监听 SIGTERM / SIGINT
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit
    log.Println("shutting down...")

    // 优雅关闭：停止接收新连接，等待存量请求完成（带超时上限）
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("forced shutdown: %v", err)
    }
    log.Println("server exited")
}
```

**关键点**：
- 只监听 SIGTERM/SIGINT，**不监听 SIGKILL**（SIGKILL 无法捕获）
- `srv.Shutdown(ctx)`：先关闭监听（拒绝新连接），再等待存量请求完成，超时强退
- 超时上限必须有：否则存量请求永远不完（如挂死的 goroutine），Pod 永远不退出

#### 2.4 terminationGracePeriodSeconds：兜底强杀

```yaml
terminationGracePeriodSeconds: 60
```

SIGTERM 发出后，超过该时间进程未退出 → K8s 发 SIGKILL 强杀。**该值必须 > 应用最大排空时间**（Shutdown 超时 + preStop sleep）。

### 3. 完整无损发布时序（K8s + Go）

```text
t=0s    滚动更新开始，创建新 Pod
t=0~5s  新 Pod 启动，readiness 未就绪（initialDelay），不在 Endpoints 中
t=5~10s readiness 通过 → 加入 Endpoints，开始接收流量
        （此时旧 Pod 开始终止流程）
t=T     旧 Pod 收到终止信号：
        ├─ readiness 探针开始返回 503 → K8s 准备摘除 Endpoints（最长 15s 生效）
        ├─ preStop sleep 10s → 等待 Endpoints 摘除完成、LB 缓存刷新
        ├─ SIGTERM → Go 收到信号 → srv.Shutdown(ctx, 30s)
        │    ├─ 停止监听新连接
        │    ├─ 等待存量请求完成（drain）
        │    └─ 30s 到 → 强制关闭剩余连接
        └─ 进程退出（terminationGracePeriodSeconds=60s 兜底）
```

### 4. 长连接场景的摘流（WebSocket / gRPC / 自研 TCP）

短连接（HTTP 请求-响应）用 readiness + drain 就够了，但**长连接不会主动断开**，必须主动踢：

```go
// 优雅关闭时主动关闭长连接
func (s *WSServer) Shutdown() {
    s.mu.Lock()
    defer s.mu.Unlock()
    for _, conn := range s.conns {
        conn.WriteControl(websocket.CloseMessage,
            websocket.FormatCloseMessage(websocket.CloseGoingAway, "server shutting down"),
            time.Now().Add(5*time.Second))
        conn.Close()
    }
}
```

**长连接摘流策略**：
1. 主动发关闭帧（WebSocket Close / gRPC GOAWAY）
2. gRPC 场景：发 GOAWAY 告诉客户端"别再发新请求了"，存量请求处理完再断
3. 客户端需支持**连接重建 + 重试**（这是双端配合的工程问题）

### 5. 注册中心模式（非 K8s）的优雅上下线

不用 K8s 时（如自建机房 + Consul/Nacos），摘流靠注册中心：

```text
下线流程：
① 主动向注册中心注销（De-register）→ 调用方不再发现该实例
② 等待注册中心同步 + 调用方本地缓存过期（TTL，一般 10~30s）
③ 停止接收新请求（健康检查接口返回 503）
④ 排空存量请求（graceful drain）
⑤ 进程退出

上线流程：
① 进程启动，初始化依赖（连接池预热）
② 本地就绪后向注册中心注册
③ 注册成功后开始接收流量（避免注册即被打死）
```

---

## 高频追问

### Q1：readiness 探针返回 503 到 Pod 真正被摘除，需要多久？
> 最长 ≈ periodSeconds × failureThreshold（如 5s × 3 = 15s）。这就是为什么需要 preStop sleep 兜底，避免 SIGTERM 发出时 Endpoints 还没摘干净。

### Q2：liveness 和 readiness 探针的区别？
> liveness 失败 → 重启容器（治"进程假死"）；readiness 失败 → 摘除流量不重启（治"未就绪/下线"）。**无损发布只用 readiness**，liveness 误判会导致发布期间反复重启。

### Q3：Shutdown 超时到了还有请求没处理完怎么办？
> 只能强杀（SIGKILL 由 K8s 在 terminationGracePeriodSeconds 后发出），残留请求由客户端重试兜底。所以**幂等 + 重试**是服务端的底线能力。另外可以监控"排空超时"指标，反推排空时间设置是否合理。

### Q4：优雅上线怎么防止"一注册就被打死"（流量洪峰）？
> 连接池/缓存预热 + 渐进式放量：注册前先预热（warmup），注册后通过注册中心权重从 0 逐步上调，配合 readiness 就绪检查。

### Q5：无损发布和灰度发布的关系？
> 无损发布解决"发布不停机、不报错"；灰度发布解决"新版本风险可控"（先放 10% 流量验证）。生产实践通常是**滚动/灰度框架 + 无损上下线能力**组合使用，详见本模块灰度发布篇。

---

## 延伸阅读

- 本模块：[K8s 核心概念、滚动发布](16-02-kubernetes.md)（RollingUpdate 参数与流程）
- 本模块：[灰度发布实战](03-04-gray-release.md)（灰度策略、回滚、数据库变更）
- 本模块：[CI/CD 流水线、蓝绿部署、金丝雀](03-cicd/13-03-cicd.md)
- Go 语言：[Graceful Shutdown 深度](../../01-golang/05-stdlib/79-graceful-shutdown.md)
- 注册中心视角：[服务注册与发现](../../03-distributed/04-service-mesh/14-02-service-discovery.md)
