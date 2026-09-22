[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [💎 语言机制](../README.md)

---

# cgo.Handle：把 Go 对象安全地传给 C 回调（整数句柄方案）

> 考察频率：★★★☆☆  优先级：P1（CGO 实战高频）
> 关键词：cgo.Handle、NewHandle、Value、Delete、user data、指针传递规则

---

## 面试官考察意图

考察候选人**在 CGO 场景下处理回调与 user data 的能力**。
初级用全局 map + Mutex 自己造轮子（还容易泄漏）；高级会直接说 `cgo.Handle`，
并讲清楚**为什么"整数句柄"比"裸指针"更安全、Handle 表为什么会泄漏、以及它和 `runtime.Pinner` 的分工**。

---

## 核心答案（30 秒版）

C 代码经常需要"user data"指针来做回调上下文（如 libuv、sqlite3、音视频 SDK）。
但 **cgo 规则禁止把 Go 指针存进 C 内存**，于是 Go 1.17 提供了 `runtime/cgo.Handle`：

```go
h := cgo.NewHandle(myGoObject)  // 返回 uintptr 句柄
C.register(C.uintptr_t(h))      // C 侧只保存这个整数，完全合法
// ...回调里...
obj := h.Value().(*MyType)      // 用句柄换回 Go 对象
h.Delete()                      // C 侧不再使用后必须删除
```

| 维度 | 裸指针 | `cgo.Handle` |
|------|--------|--------------|
| 合法性 | ❌ 存入 C 内存违反规则 | ✅ 整数，无指针语义 |
| 地址稳定性 | 依赖 Pinner | 不涉及（不暴露地址）|
| 生命周期 | 需手动 Pinner 保活 | Handle 表内部持有引用 |
| 泄漏风险 | 忘记 Unpin | 忘记 Delete |
| 并发安全 | 需自己保证 | 内部加锁，安全 |

**结论：需要"给 C 一个上下文标识"时，优先用 `cgo.Handle`，而不是 `Pinner` + 裸指针。**

---

## 深度展开

### 1. 为什么不能用裸指针：回顾 cgo 指针规则

```go
// ❌ 走不通：C 侧保存 Go 指针（长期持有）
/*
static void *g_userdata;
void set_userdata(void *p) { g_userdata = p; }
*/
C.set_userdata(unsafe.Pointer(obj)) // 违规：C 不能在返回后持有 Go 指针
```

违规的直接后果：`cgocheck` 报错（`panic: runtime error: cgo argument has Go pointer to unpinned Go pointer`），
即便绕过检查，GC 也可能回收对象导致 C 侧悬空指针。

`cgo.Handle` 的思路是**把指针换成整数**：

```
Go 对象  ──注册──→  Handle 表（运行时管理，uintptr → interface{}）
                          ↑
C 侧只保存 uintptr ────────┘
```

C 侧持有的只是 `uintptr_t`，不违反任何指针规则；真正的对象引用安全地留在 Go 运行时里。

### 2. 完整示例：C 库回调带上下文

```go
/*
// C 代码
#include <stdint.h>
typedef void (*cb_t)(uintptr_t ctx, int code);
void start_async(uintptr_t ctx, cb_t cb);
*/
import "C"

import "runtime/cgo"

type Session struct {
    conn *Conn
    log  *slog.Logger
}

//export goCallback
func goCallback(ctx C.uintptr_t, code C.int) {
    // 用句柄换回 Go 对象
    s, ok := cgo.Handle(ctx).Value().(*Session)
    if !ok {
        return // 已被删除或类型不符
    }
    s.onEvent(int(code))
}

func (s *Session) Start() {
    h := cgo.NewHandle(s) // 1) 注册，得到句柄
    s.handle = h
    C.start_async(C.uintptr_t(h), (C.cb_t)(C.goCallback)) // 2) 只传整数
}

func (s *Session) Close() {
    s.handle.Delete()    // 3) C 侧确实不再回调后删除
    s.handle = 0
}
```

要点：

- `NewHandle` 返回值是 `cgo.Handle`（`uintptr` 类型），可安全转成 `C.uintptr_t`；
- `Value()` 返回 `any`，需要类型断言；
- **`Delete()` 必须调用**：Handle 表在运行时内部持有对对象的强引用，
  不 Delete 就永远不回收（表现为内存缓慢增长但 pprof 里对象都是 root）。

### 3. Handle 的内部机制与并发

```go
// 概念模型
var handleMap struct {
    mu   sync.Mutex
    m    map[uintptr]any
    next uintptr
}

func NewHandle(v any) Handle {
    // 加锁、分配 id、存入 map
}
func (h Handle) Value() any { /* 加锁读取 */ }
func (h Handle) Delete()    { /* 加锁删除 + 置空 */ }
```

- **并发安全**：NewHandle/Value/Delete 内部都有锁，多线程回调可放心用；
- **id 复用**：Delete 后 id 可能被后续 NewHandle 复用，**已删除句柄再 Value() 会 panic 或返回错误对象**，
  所以"删除后立刻再回调"的竞态必须由业务层用状态机/锁挡住；
- **性能**：每次 Value() 有一次加锁查表，热点回调（每秒百万次）要评估，必要时改批量/减少查表。

### 4. 与 Pinner 的分工

| 需求 | 选择 |
|------|------|
| C 需要一个"上下文标识"（回调 user data）| ✅ `cgo.Handle` |
| C 需要**直接读写** Go 内存（如把数据写进 Go 的 buffer）| ✅ `runtime.Pinner` + 指针 |
| C 只是调用期间读取 Go 内存 | 直接传指针即可（无需 Pin）|
| 想避免指针规则的一切麻烦 | ✅ `cgo.Handle`（传下标/句柄 + 在 Go 侧读写）|

> 工程原则：**能用句柄就不传指针**。句柄模式下所有内存访问都发生在 Go 侧，
> 既避开了 cgo 指针规则，也避开了 `Pinner` 忘记 Unpin 的泄漏风险。

### 5. 生产坑点清单

```go
// ❌ 坑 1：Delete 之后再回调 → 句柄可能已被复用，拿到错误对象
// ❌ 坑 2：把 Handle 当长期 ID 用（存活期不可控，句柄会被复用）
// ❌ 坑 3：忘记 Delete → 强引用泄漏
// ✅ 坑 4 的正确解法：用状态字段 + Mutex 保护"是否已关闭"，回调先检查
```

- 关闭流程必须与回调线程做**双向同步**（先置 closed 标记，再 Delete）；
- 用 `runtime.NumCgoCall`、`cgo.Handle` 表大小趋势结合 metrics 观察泄漏；
- 大量短生命周期回调不要每个都 NewHandle/Delete，可考虑**一个句柄对应一个长生命周期对象**。

---

## 高频追问

**Q：`cgo.Handle` 里存的对象会被 GC 扫描吗？**

会被视为存活（强引用），因为它存在运行时的 handle map 里，属于根可达；
这正是忘记 Delete 会泄漏的根本原因。

**Q：为什么不让 C 侧直接存索引（数组下标）？**

手动下标方案需要自己实现"数组 + 空闲链表 + 并发保护"，
`cgo.Handle` 就是标准库把这个轮子造好了，稳定且经过优化。

**Q：`cgo.Handle` 与 `syscall.Handle`（Windows）有关系吗？**

没有。`cgo.Handle` 是 `runtime/cgo` 里的 Go 对象句柄；`syscall.Handle` 是 Windows 的系统句柄类型，二者不相干。

---

## 面试话术

> "C 回调要上下文时，不能把 Go 指针塞进 C 内存，Go 1.17 的 `cgo.Handle` 就是标准答案：
> `NewHandle` 拿到一个 uintptr 存到 C 侧，回调里 `Value()` 换回对象，用完必须 `Delete`，
> 否则运行时强引用让你内存泄漏。它比 `Pinner` 更安全，因为整个过程中 C 从没拿到过真实指针。"

---

## 延伸阅读

- [runtime/cgo.Handle 文档](https://pkg.go.dev/runtime/cgo#Handle)
- [Go 1.17 Release Notes — cgo.Handle](https://go.dev/doc/go1.17#cgo)
- [cgo 指针传递规则](https://pkg.go.dev/cmd/cgo#hdr-Passing_pointers)

---

**[上一篇：unique 驻留](./86-unique-interning.md)** · **[下一篇：unsafe.Slice / unsafe.String →](./88-unsafe-slice-string.md)**
