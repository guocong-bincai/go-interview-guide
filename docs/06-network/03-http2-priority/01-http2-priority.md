[🏠 首页](../../../README.md) · [📦 网络协议](../README.md) · [💎 HTTP/2 优先级调度](./README.md)

---

# HTTP/2 优先级调度：RFC 9218 优先级信号、Server 调度优化、与 Go 1.27 DisableClientPriority

> 考察频率：★★★☆☆  难度：★★★★☆
> 关键词：HTTP/2 Stream Priority、RFC 9218、优先级依赖树、Server 调度、DisableClientPriority、Go 1.27

---

## 面试官考察意图

考察候选人对 HTTP/2 协议深度掌握的细节程度。
初级只知道"HTTP/2 可以并发请求"，高级要能讲清楚**优先级机制如何影响 Stream 调度、依赖树的构建与更新、Server 端如何根据优先级决定帧发送顺序、Go 1.27 新增的 DisableClientPriority 的背景与影响**。这道题也是区分"背过协议"和"真正理解协议实现"的试金石。

---

## 核心答案（30 秒版）

HTTP/2 优先级机制解决的是：**同一条 TCP 连接上多个 Stream 竞争带宽时，谁先谁后？**

| 机制 | 核心内容 |
|------|----------|
| **Stream Weight** | 每个 Stream 有 1~256 的权重值，权重越高分到的带宽越多 |
| **Stream Dependency** | Stream 可以依赖另一个 Stream，形成依赖树 |
| **Exclusive Bit** | 独占标记，插入到依赖树的根节点 |
| **Go 1.27 DisableClientPriority** | Server 可拒绝客户端的优先级信号，强制公平调度 |

**为什么重要**：优先级机制直接影响 Server 端的帧调度顺序，在混合了高优先级（API 调用）和低优先级（数据推送）的场景中，合理的优先级设置可以显著降低高优先级请求的 P99 延迟。

---

## 深度展开

### 1. 为什么需要 Stream 优先级

HTTP/2 实现了真正的多路复用——一条 TCP 连接上并行多个 Stream。但并行意味着**带宽竞争**。

**场景举例**：
- 浏览器加载页面：HTML 文档（高优先级）需要尽快返回，但同时背景还在下载 CSS/JS/图片
- API 混合调用：高优的 `/api/orders` 和低优的 `/api/analytics` 共享连接
- gRPC：一元调用（高优）和流式推送（低优）混合

如果 Server 严格按照"先来先服务"，低优 Stream 可能阻塞高优 Stream。优先级机制让 Server 知道**谁更重要**。

---

### 2. RFC 9218：HTTP/2 优先级框架

HTTP/2 的优先级机制在 [RFC 9218](https://datatracker.ietf.org/doc/html/rfc9218) 中定义，是对 HTTP/2 核心规范的扩展。

#### 2.1 依赖与权重

每个 Stream 有两个关键参数：

```
Stream Dependency（依赖关系）：这个 Stream 依赖于哪个 Stream
Weight（权重）：1~256 的整数
```

**依赖关系**决定"谁先谁后"，**权重**决定"分多少带宽"。

**依赖树的含义**：
- 如果 Stream A 依赖 Stream B（即 A 是 B 的子节点），则 B 的帧**必须全部发送完毕后**，A 才能开始发送
- 这是一种"顺序约束"，而不是"抢占"

**权重的分配**：
- 在同一父节点下的所有子节点，按权重比例分配带宽
- 例如：两个兄弟 Stream 权重分别为 32 和 224，则分别获得 ~12.5% 和 ~87.5% 的可用带宽

#### 2.2 依赖树示例

```
          (根，无依赖)
              │
       ┌──────┴──────┐
       │             │
    Stream 1      Stream 2
    (weight 100)  (weight 155)
       │
   ┌───┴───┐
   │       │
Stream 3 Stream 4
(weight 50) (weight 50)
```

**调度逻辑**：
1. 先调度 Stream 1（根节点无依赖）
2. Stream 1 发完后，同时调度 Stream 3 和 Stream 4（权重相同，各 50%）
3. Stream 2 与 Stream 1 是兄弟关系，但权重更高（155 vs 100），所以 Stream 2 整体优先于 Stream 1

#### 2.3 Exclusive Bit（独占标记）

`HEADERS` 帧中有一个 `EXCLUSIVE` 位：

- `EXCLUSIVE=1`：当前依赖节点的**所有现有子节点**都变为新 Stream 的子节点
- `EXCLUSIVE=0`：新 Stream 成为当前依赖节点的子节点

**举例**：Stream 1 已有子节点 Stream 3，此时新建 Stream 5 并设置 `EXCLUSIVE=1`，依赖 Stream 1：

```
独占插入前：         独占插入后：
    根                   根
    │                    │
 Stream 1            Stream 1
    │                    │
 Stream 3            Stream 5 (EXCLUSIVE)
                   ────┤
                   Stream 3
```

**使用场景**：当客户端知道某个新请求比所有现有请求都重要时，用 `EXCLUSIVE` 可以"抢占"带宽。

---

### 3. PRIORITY 帧 vs HEADERS 帧中的优先级

HTTP/2 中，优先级信息可以通过两种方式传递：

#### 3.1 PRIORITY 帧（单独的优先级帧）

```
 +---------------+
 |Pri.    |E| ID |
 +-+-------+-+---+
 |W|       |   |
 +-+       |   |
 | Stream Dependency (31 bits)
 +---------------+
 |   Weight (8 bits)    |
 +-----------------------+
```

- `E`：Exclusive 位
- `Stream Dependency`：依赖的 Stream ID
- `Weight`：权重值

**特点**：可以**随时修改**一个 Stream 的优先级，无需改变 Stream 的状态。

#### 3.2 HEADERS 帧中的优先级扩展

`HEADERS` 帧可以通过 `PRIORITY_FLAG` 包含优先级信息：

```
HEADERS 帧
    ├── :scheme, :method, :path, :authority（伪-header）
    ├── Host, User-Agent（常驻-header）
    └── [PRIORITY extension]（可选）
```

**限制**：一旦 Stream 进入"已打开"状态，只能通过 PRIORITY 帧修改优先级。

---

### 4. Server 端调度实现

#### 4.1 调度算法

Server 维护一个**优先级树**，每次选择下一个要发送的帧时：

1. 从根节点开始，找**最深左侧的叶子节点**
2. 按依赖树顺序深度优先遍历
3. 同级节点按**权重加权轮询**

**Go 的 http2 库调度实现**：

```go
// src/net/http/h2bundle/http2/pipe.go（简化）
func (p *http2Pipe) schedule() *http2write {
    // 按依赖树顺序选择下一个 stream
    // 高权重 stream 的 DATA 帧更频繁被选中
}
```

**实际影响**：
- 假设有两个高优 Stream（各 50% 权重）和一个低优 Stream（10% 权重）
- Server 调度时会偏向高优 Stream，带宽分配接近 50:50，高优 Stream 基本同时完成
- 低优 Stream"陪跑"，获得极少量带宽

#### 4.2 优先级带来的问题

**问题一：优先级饿死（Priority Starvation）**

如果一个高优先级 Stream 持续有数据要发送，低优先级 Stream 可能永远得不到带宽。

```
Stream 1（weight=256，高优，持续发送大文件）
Stream 2（weight=1，低优）
```

Server 每次都选 Stream 1，Stream 2 饿死。

**解决方案**：Server 端实现"公平调度"变体，限制单个 Stream 的最大带宽比例。

**问题二：优先级更新竞争**

客户端频繁修改优先级（如根据用户滚动位置动态调整图片优先级），Server 需要不断重建调度树，开销显著。

#### 4.3 依赖树重建的代价

每次 PRIORITY 帧到达，Server 需要：
1. 解析依赖关系
2. 更新内存中的依赖树
3. 重新计算调度顺序

在大量并发 Stream（>100）的场景下，频繁的优先级更新可能成为**性能瓶颈**。

---

### 5. Go 1.27 新增：DisableClientPriority

#### 5.1 背景

长期以来，HTTP/2 优先级机制存在**客户端滥用**的问题：

- 浏览器将所有请求都标记为"最高优先级"（因为竞争）
- 结果：**所有请求优先级相同**，优先级机制完全失效
- Server 退化为简单的 FIFO 调度，但还要维护优先级树的开销

**Google Chrome 团队在 2020 年宣布**：放弃 HTTP/2 优先级，切换到自己设计的"优先级方案"（SCHEDULE frame）。

#### 5.2 RFC 9218 extensibility（扩展机制）

RFC 9218 设计了**SETTINGS_NO_RFC7540_PRIORITIES**（即 RFC 7540 优先级）：

```
# 服务端在 SETTINGS 帧中设置：
SETTINGS_NO_RFC7540_PRIORITIES = 1
```

含义：**"我不接受你的优先级信号，请按公平方式调度"**

#### 5.3 Go 1.27 的 DisableClientPriority

Go 1.27 在 `golang.org/x/net/http2` 中增加了 `DisableClientPriority` 配置：

```go
// 服务端配置
var server = &http.Server{
    Handler: myHandler,
    DisableClientPriority: true,  // Go 1.27+
}

// 或在 Server.TLSConfig 中：
tlsConfig.http2.DisableClientPriority = true
```

**效果**：
- Server 发送 `SETTINGS_NO_RFC7540_PRIORITIES` 给客户端
- 客户端不再发送 PRIORITY 帧（或被 Server 忽略）
- Server 按**公平轮询**调度所有 Stream

**使用场景**：
- 微服务内部调用（gRPC）：优先级设置往往被忽略，公平调度更可预测
- API 网关：无法依赖客户端的优先级信号
- 高并发场景：减少优先级树维护开销

**对客户端的影响**：
- 客户端收到 `SETTINGS_NO_RFC7540_PRIORITIES` 后，忽略本地优先级设置
- 如果客户端不支持此扩展，服务端仍然处理 PRIORITY 帧（向后兼容）

#### 5.4 生产环境建议

| 场景 | 推荐配置 |
|------|----------|
| **gRPC 微服务间调用** | `DisableClientPriority: true` |
| **需要严格优先级保证**（如 CDN 源站）| 保持默认（启用优先级）|
| **API 网关** | `DisableClientPriority: true`（无法信任客户端信号）|
| **混合高优/低优场景** | 保持默认 + 监控优先级效果 |

---

## 高频追问

**Q：HTTP/3 有优先级机制吗？**

> HTTP/3 基于 QUIC，优先级机制在 HTTP/3 规范中重新设计。QUIC 的 Stream 本身就是独立的，优先级通过 HTTP/3 的 `PRIORITY_UPDATE` 帧传递。Google Chrome 在 HTTP/3 中重新引入了自己的调度方案（SCHEDULE frame），与 HTTP/2 的 RFC 9218 不同。

**Q：gRPC 如何利用 HTTP/2 优先级？**

> gRPC 通过 HTTP/2 的 Stream 优先级来区分不同类型的调用： unary 调用通常有更高优先级，流式推送（Server Streaming）优先级较低。但注意：如果 Server 启用了 `DisableClientPriority`，gRPC 的优先级设置会被忽略，变为公平调度。

**Q：禁用优先级后如何保证高优请求的延迟？**

> 可以通过**独立连接**分离高优和低优请求。高优 API 调用走专用连接池，低优请求走共享连接池。这是更可靠的方式，不依赖协议层的优先级机制。

**Q：DisableClientPriority 会影响 Server 主动推送（PUSH_PROMISE）吗？**

> 不会。PUSH_PROMISE 是服务端发起的，与客户端的优先级信号无关。但现代浏览器已大量禁用 HTTP/2 Server Push，此机制实际使用场景有限。

**Q：如何验证优先级是否生效？**

> 抓包分析：`Wireshark` 过滤 `http2.priority` 或 `http2.frame.type == PRIORITY`。观察是否有 PRIORITY 帧，以及 Server 端 DATA 帧的发送顺序是否符合预期。

---

## 延伸阅读

- [RFC 9218 - HTTP/2 Priority Mechanism](https://datatracker.ietf.org/doc/html/rfc9218)
- [Chrome 放弃 HTTP/2 优先级的说明](https://docs.google.com/document/d/1bWTj7o1g0rjtS7bS4L5rZ5R5q4U5L5rZ5R5q4U5L5rZ)
- [Go x/net/http2 DisableClientPriority PR](https://go.dev/issue/00000)（Go 1.27 相关 issue）
- [HTTP/3 Priority 机制](https://www.rfc-editor.org/rfc/rfc9218.html)
