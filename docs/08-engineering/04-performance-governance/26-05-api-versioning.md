# API 版本管理与向后兼容：REST 版本策略与 Protobuf 字段演进

> 考察频率：★★★★☆  优先级：P1
> 关键词：API 版本、向后兼容、语义化版本、Protobuf 字段编号、Breaking Change、废弃流程

## 面试官考察意图

"你的接口怎么升级？"——这是高级后端工程师的必答题。它考的不是"你会不会加个 `/v2`"，而是你对**兼容性契约**的理解：

1. 什么改动是**向后兼容**的（安全），什么是 **Breaking Change**（危险）；
2. REST 和 gRPC/Protobuf 两种范式下，版本策略的差异；
3. 你做过**废弃（Deprecation）流程**吗？——直接下线老接口、用户全炸，是典型的初级做法。

**核心立场：API 是契约，一旦发布就要像数据库 schema 一样谨慎演进。向后兼容不是"可选优化"，是设计纪律。**

---

## 核心答案（30 秒版）

**兼容性判据（最重要的一张表）**：

| 改动 | 兼容性 | 说明 |
|------|--------|------|
| **新增可选字段** | ✅ 兼容 | 老客户端忽略未知字段 |
| **新增枚举值** | ⚠️ 半兼容 | 老客户端可能不认识新值，需有 default 分支 |
| **删除字段 / 改字段名 / 改类型** | ❌ Breaking | 老客户端解析失败 |
| **字段编号复用（含 reserved 忽略）** | ❌ Breaking | 解析出错误数据，Protobuf 大忌 |
| **把必填改可选** | ✅ 兼容 | |
| **把可选改必填** | ❌ Breaking | 老客户端不传就失败 |
| **新增接口 / 增大返回值范围** | ✅ 兼容 | |

**REST 版本策略三选一**：

1. **URL 版本**（`/api/v1/orders`）——最直观，缓存/日志友好，**推荐**；
2. **Header 版本**（`Accept: application/vnd.api+json;version=2`）——RESTful 纯粹，但调试不便；
3. **Query 参数**（`?version=2`）——简单但不优雅。

**gRPC 版本策略**：**不用在方法名里加 v2**，而是靠 **Protobuf 的字段演进规则** + **包名版本**（`order.v1`、`order.v2`）。

**一句话**：**加字段、加接口、加枚举值是安全的；删、改、复用编号是危险的。Breaking Change 必须走"新版本 + 并行运行 + 废弃流程"。**

---

## 深度展开

### 一、先建立"兼容性契约"思维

向后兼容的定义：**新版本服务端能正确处理老版本客户端的请求**（backward compatible）；**老版本服务端能处理新版本客户端的请求**（forward compatible）。

现实约束：**服务端无法强制所有客户端同时升级**——iOS 老版本可能几个月都不更新。所以服务端必须"向后兼容老客户端"。

```text
时间轴：
  v1 发布 ────────────── v2 发布 ──────── v1 下线
                          │                 │
  t0                      t1                t2
  老客户端仍在用 v1 ────── 必须继续支持 ──── 确认无流量后才可下线
```

**关键指标**：下线 v1 的前提是**监控到 v1 流量归零**（按客户端版本维度打点），而不是"公告说 3 个月后下线"。

### 二、REST 版本管理实战

**URL 版本（推荐）**

```go
// Go 1.22+ ServeMux 支持方法 + 路径变量
mux := http.NewServeMux()
mux.HandleFunc("GET /api/v1/orders/{id}", v1.GetOrder)
mux.HandleFunc("GET /api/v2/orders/{id}", v2.GetOrder)
```

**共享版本无关逻辑**：

```go
type OrderUsecase struct{ /* 业务逻辑 */ }

// v1 handler：输出扁平结构
func (h *v1Handler) GetOrder(w http.ResponseWriter, r *http.Request) {
    o := h.uc.Get(...)
    json.NewEncoder(w).Encode(v1Order{
        ID:   o.ID,
        Amount: o.AmountYuan,          // v1 用元
        Status: o.Status,
    })
}

// v2 handler：输出更丰富结构（加了 createdAt、分单位）
func (h *v2Handler) GetOrder(w http.ResponseWriter, r *http.Request) {
    o := h.uc.Get(...)
    json.NewEncoder(w).Encode(v2Order{
        ID:        o.ID,
        AmountFen: o.AmountFen,        // v2 改成分
        Status:    o.Status,
        CreatedAt: o.CreatedAt.UTC(),  // v2 新增
    })
}
```

**要点**：**版本差异只体现在 DTO 层，业务逻辑复用**。不要为每个版本复制一份 usecase。

**为什么 URL 版本更受欢迎**：

- 日志、监控、网关路由能按路径维度统计（`/api/v1/*` 的 QPS 一眼可见）；
- CDN / 缓存策略可以按路径配置；
- 调试时 `curl` 一眼看出用的是哪版。

### 三、Protobuf / gRPC 的兼容性规则（面试重点）

Protobuf 的兼容性**围绕字段编号（field number）而非字段名**：

```proto
message Order {
  int64  id         = 1;
  string status     = 2;
  // 3 已删除，必须 reserved，绝不能复用！
  reserved 3;
  reserved "old_amount";
  int64  amount_fen = 4;   // 新增字段用新编号
  optional string remark = 5; // proto3 显式 optional 才能判"是否设置"
}
```

**铁律清单**：

1. **字段编号永不复用**：删字段只 `reserved`，编号让给下一个新字段会**静默解析出错误数据**（最危险的 bug）；
2. **不要改字段类型**（`int32` → `string` 直接崩；`int32` → `int64` 在 Go 里因为 `int` 平台相关也有坑，**用 `int64` 起步**）；
3. **enum 必须留 `_UNSPECIFIED = 0`**，新值只加在末尾，老客户端遇到未知值走 default 分支（**不能用 `switch` 直接 panic**）；
4. **不要删 required、不要给已上线消息加 required**（proto3 已无 required，但 proto2 混用要注意）；
5. **用 `optional` 区分"零值"和"未设置"**：proto3 里 `int32 x = 1` 收到 0 无法判断是"传了 0"还是"没传"，加 `optional` 才有 `has_x`。

**版本升级方式**：

```proto
// 方案 A（推荐）：包名带版本，v1/v2 并存
package order.v1;
package order.v2;  // 重大不兼容时新建包

// 方案 B：只演进字段，服务端同时支持新老字段
//    老客户端只填部分字段 → 服务端需处理"字段缺失"的语义
```

### 四、废弃（Deprecation）流程：这才是高级答案

直接删接口 = 生产事故。正确流程：

```
第 1 步：标记废弃
  - REST: 响应头 Deprecation: true、Sunset: <RFC1123 日期>（RFC 8594）
  - gRPC: option deprecated = true;
  - 文档标注 "将在 2026-12-31 下线"

第 2 步：埋点监控调用方
  - 按 API key / 客户端版本 / User-Agent 统计 v1 的调用量与调用方
  - 找出 TOP 调用方，主动通知

第 3 步：宽限期 + 逐渐限制
  - 响应头加 Warning 提示
  - 对新调用方（新 key）只开放 v2

第 4 步：下线
  - 确认 v1 流量 = 0（连续 N 天）
  - 灰度：先对 1% 返回 410 Gone（HTTP）观察
  - 全量下线，返回 410 + 迁移文档链接
```

**HTTP 状态码语义**：

- `410 Gone`：资源永久移除（比 404 更明确，告诉调用方"别再试了"）；
- `404`：可能是路径写错，语义模糊，**下线接口用 410**。

**Nginx 层拦截下线接口**：

```nginx
location ~ ^/api/v1/legacy {
    return 410;  # 快速失败，不打到后端
    add_header X-Deprecated-Since "2026-06-01";
    add_header Link '<https://docs.example.com/migrate-v2>; rel="deprecation"';
}
```

### 五、契约测试：防止兼容性被悄悄破坏

光有规范不够，要用**契约测试**兜住：

1. **向后兼容性检查工具**：
   - Protobuf：`buf breaking --against '.git#branch=main'`——**CI 里跑，任何 breaking change 直接红**；
   - REST：`oasdiff breaking`——对比 OpenAPI yaml，检测破坏性改动；
2. **消费者驱动契约**：`pact-go`，让"调用方"定义期望，服务端 CI 验证；
3. **快照测试**：把 API 响应结构序列化成快照，改动时 diff 出差异。

```yaml
# .github/workflows/api-compat.yml
- name: Check Protobuf backward compatibility
  run: buf breaking --against 'https://github.com/org/repo.git#branch=main'
```

**这是面试的高分点**：不是"我们规定不许改"，而是"**CI 强制不许改，改了过不了**"。

### 六、REST vs gRPC 版本策略对比

| 维度 | REST | gRPC / Protobuf |
|------|------|-----------------|
| 版本载体 | URL / Header | package 版本 or 字段演进 |
| 未知字段处理 | JSON 默认忽略（宽松） | 保留在 message 中（`unknown fields`） |
| 兼容性检查 | 手动 / oasdiff | `buf breaking` 自动化 |
| 最佳实践 | URL `/v1/` | 尽量不加版本，靠字段演进 |
| 破坏性改动 | 新建 `/v2` | 新建 `package x.v2` |

**一句话差异**：**REST 靠"路径版本"做粗粒度隔离；gRPC 靠"字段编号规则"做细粒度演进，只有语义大变才开 `v2` 包。**

---

## 高频追问

**Q1：为什么不直接上 `v2` 而优先做字段演进？**
维护两套 API 的成本很高（双倍测试、双倍文档、路由分裂）。**能用加字段解决的，就不要开新版本**。只有"语义/结构发生不可调和变化"（如列表接口改成游标分页且参数不兼容）才值得开 `v2`。

**Q2：JSON 里新增字段，老客户端会崩吗？**
标准 JSON 解析器会忽略未知字段，**默认兼容**。但要注意：① 有些客户端开了"严格模式"（Jackson `FAIL_ON_UNKNOWN_PROPERTIES`）会崩；② 如果客户端做了**全量字段校验**（如 JSON Schema `additionalProperties: false`）也会拒。所以"加字段"虽然理论上兼容，**仍要评估客户端解析策略**。

**Q3：怎么判断某个字段还有没有人用？**
在网关/中间件里**按字段埋点**（反序列化后记录哪些字段被访问），或依赖 gRPC 的字段级监控。**没有数据就别删字段。**

**Q4：Protobuf 删了字段，老数据怎么办？**
老数据里该编号仍有值，若复用编号，反序列化时会**把旧数据的新字段位置读出脏值**。这就是 `reserved` 必须写的原因。**永远 `reserved`，永不复用。**

**Q5：`v1` 和 `v2` 能共用同一份 DB 吗？**
通常能，**版本差异只在 API/DTO 层**。若数据模型也变了（如金额从元改分），需要**双写过渡期**：写入时同时写两个字段，读时按版本返回，确认迁移完成后再下线旧字段。

---

## 面试话术

> "我们把 API 当**契约**管理：版本用 **URL 前缀**（`/api/v1`），差异只体现在 DTO 层，业务逻辑复用。兼容性上坚持**只加不改**——加可选字段、加接口是安全的；删字段、改类型、复用 Protobuf 字段编号都是 Breaking Change。gRPC 侧靠**字段编号演进 + `reserved` 禁复用**，CI 里跑 **`buf breaking`** 强制拦截破坏性改动。真要大改就开 `v2`，走**废弃流程**：响应头加 `Deprecation/Sunset` → 埋点找调用方 → 宽限期 → 确认流量归零后返回 `410 Gone`。**下线的前提是监控数据，不是公告日期。**"

---

## 延伸阅读

- [RFC 8594: The Sunset HTTP Header](https://www.rfc-editor.org/rfc/rfc8594)
- [buf breaking change detection](https://buf.build/docs/breaking/overview)
- [Protobuf: Updating A Message Type](https://protobuf.dev/programming-guides/proto3/#updating)
- [Stripe API Versioning 实践](https://stripe.com/blog/api-versioning)
- 本仓库：[IDL 设计](https://github.com/guocong-bincai/go-interview-guide/blob/main/docs/04-microservices/01-rpc/12-04-idl-design.md)
