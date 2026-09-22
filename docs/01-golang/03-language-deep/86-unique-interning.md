[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [💎 语言机制](../README.md)

---

# unique 包：字符串驻留（Interning）与内存去重利器

> 考察频率：★★★★☆  优先级：P1（Go 1.23+ 高频新特性）
> 关键词：unique.Make、Handle、interning、字符串驻留、weak pointer、内存优化

---

## 面试官考察意图

考察候选人对 **Go 1.23 新标准库 `unique`** 的理解，以及**内存优化的工程判断力**。
初级不知道有这个包；高级要能讲清楚 **`unique.Make` 的语义与实现（全局并发 map + 弱引用）、Handle 的可比较性、适用与不适用的场景、以及和 `sync.Map`/`string.Intern` 式方案的区别**。

---

## 核心答案（30 秒版）

`unique` 包提供**值驻留（interning）**：相同值只保留一份内存，返回一个轻量的 `Handle[T]`。

```go
h1 := unique.Make("application/json")
h2 := unique.Make("application/json")
// h1 == h2         （Handle 可比较）
// h1.Value() == h2.Value()，且底层字符串只有一份
```

| 特性 | 说明 |
|------|------|
| 约束 | `T` 必须 `comparable`（string、int、结构体、数组等）|
| 句柄 | `Handle[T]` 可比较、可作 map key、零 GC 压力（本质是一个指针）|
| 回收 | 没有任何 Handle 存活时，驻留值由 GC 回收（内部用弱引用）|
| 代价 | 每次 `Make` 要哈希 + 加锁，**不适合高频唯一值** |

**一句话：`unique` 用"少量哈希开销"换"大量重复值的内存节省"，适合高重复、低基数的值（MIME、状态枚举、label、URL 模板）。**

---

## 深度展开

### 1. 典型用法：把重复字符串压成一份

```go
import "unique"

type RequestLog struct {
    Method  unique.Handle[string]
    MIME    unique.Handle[string]
    Status  unique.Handle[string]
    Latency int64
}

func logRequest(r *http.Request, status string) RequestLog {
    return RequestLog{
        Method: unique.Make(r.Method),                       // 常见重复值
        MIME:   unique.Make(r.Header.Get("Content-Type")),   // 常见重复值
        Status: unique.Make(status),
    }
}

// 统计：Handle 可直接当 map key
func count(logs []RequestLog) map[unique.Handle[string]]int {
    m := make(map[unique.Handle[string]]int)
    for _, l := range logs {
        m[l.Status]++
    }
    return m
}
```

收益：100 万条日志里 `"GET"`、`"application/json"` 只存一份，
`Handle` 本身只是一个指针大小，比每条日志各存一个字符串 header（16 字节 + 数据）省得多。

### 2. 实现原理

```go
// 概念模型（简化）
type Handle[T comparable] struct {
    value *T   // 指向驻留表中的唯一副本
}

func Make[T comparable](v T) Handle[T] {
    // 1) 在全局驻留表（分片并发 map）中查找
    // 2) 命中 → 直接返回已有 Handle
    // 3) 未命中 → 插入 v，返回新 Handle
    // 表的 value 是弱引用：没有 Handle 指向时，条目被 GC 回收
}
```

要点：

- **内部是"分片哈希表 + 弱引用"**：分片降低锁竞争，弱引用保证表不会无限增长（`weak` 包在 Go 1.24 正式落地，`unique` 是其早期使用者）；
- **Handle 是指针包装**，比较 Handle 就是比较指针（O(1)），比比较长字符串快；
- **`Make` 不是免费的**：每次调用要算哈希、可能抢锁，热点路径要评估。

### 3. 适用 / 不适用场景

| 场景 | 是否适合 | 原因 |
|------|---------|------|
| HTTP 方法、Content-Type、状态码 | ✅ | 基数极低、重复率极高 |
| 枚举字符串、日志级别、错误码 | ✅ | 同上 |
| 配置项、服务名、机房名 | ✅ | 数量有限、长期重复 |
| 每个请求 TraceID / UUID | ❌ | 值唯一 → 每次都是新条目，纯增加哈希与锁开销 |
| 用户昵称、URL 全路径 | ❌ | 基数巨大，表会膨胀（虽然有弱引用，但会持续抖动）|
| 大对象（大 struct） | ⚠️ | 驻留成功才省内存，否则白白拷贝+哈希 |

### 4. 与其它去重方案的对比

| 方案 | 内存 | 比较成本 | 回收 | 适用 |
|------|------|---------|------|------|
| 直接存 string | 高（每份都存）| 按内容比较 | 随引用回收 | 通用 |
| `unique.Make` | 低（一份）| 指针比较 O(1) | 弱引用表自动回收 | 低基数高重复 |
| `sync.Map` 自建缓存 | 中 | O(1) 但需管理 | 需手动清理，易泄漏 | 自定义逻辑 |
| 分片 map + 手动 TTL | 中 | O(1) | 手动 | 需要过期策略 |

> `unique` 相当于标准库内置的"带 GC 的 intern 表"，省掉了自己维护缓存泄漏的麻烦。

### 5. 生产注意点

```go
// ❌ 坑 1：在超高基数场景滥用，表持续抖动
h := unique.Make(string(uuid))  // 每次都新插入，纯亏

// ❌ 坑 2：误以为 Make 会"保留"值
//     没有 Handle 存活时，值会被回收；不要把 Handle 丢掉后再 Value()

// ✅ 正确：Handle 本身就是长期引用的载体，存 Handle 而不是原值
var statusOK = unique.Make("ok")
func getStatus() unique.Handle[string] { return statusOK }
```

- **不要把 `unique` 当缓存用**：它不做 TTL，也不解决"我想缓存大对象"的需求（用 `weak` + 自定义表）；
- **性能需实测**：`Make` 有哈希与原子/锁开销，基准测试对比后再决定；
- **跨包共享**：全局变量持有 Handle 是最常见的正确用法。

---

## 高频追问

**Q：`unique.Handle[T]` 为什么可比较，而 `T` 只是 comparable？**

`Handle` 内部是 `*T`，比较的是指针，天然可比较（且是 O(1)）。
这也意味着 **Handle 的相等蕴含值的相等**（因为相同值只驻留一份），可以安全作为 map key。

**Q：`unique` 和 Java 的 `String.intern()` 有什么区别？**

Java 的字符串常量池由 JVM 管理、默认不回收（容易 OOM）；
Go 的 `unique` 用弱引用，**没有 Handle 引用时条目可被 GC 回收**，更安全。

**Q：如果没有用 `unique`，如何手动做驻留？**

分片 `map[T]*T` + 读写锁是常见做法，但必须自己处理清理与并发；
Go 1.24 之后也可以用 `weak.Pointer` 让条目随引用消失，实际上就是 `unique` 的内部思路。

---

## 面试话术

> "`unique` 是 Go 1.23 的值驻留方案：`unique.Make` 返回可比较的 `Handle`，
> 相同值全局只存一份，句柄比较是 O(1) 指针比较，内部用分片表 + 弱引用所以不会泄漏。
> 它适合 HTTP 方法、Content-Type、枚举这种低基数高重复的值，
> 但 TraceID 这种唯一值用它反而亏，因为每次都要哈希和加锁。"

---

## 延伸阅读

- [unique 包文档](https://pkg.go.dev/unique)
- [Go 1.23 Release Notes — unique](https://go.dev/doc/go1.23#unique)
- [Proposal: unique package](https://go.dev/issue/62483)

---

**[返回模块首页 →](../README.md)**
