[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 运行时原理](../README.md)

---

# runtime.Pinner：cgo 场景下安全地把 Go 指针交给 C

> 考察频率：★★★☆☆  优先级：P1（CGO / 云原生底层加分项）
> 关键词：runtime.Pinner、cgo 指针规则、Pin / Unpin、GC root、cgocheck、Go 1.21

---

## 面试官考察意图

考察候选人对 **CGO 指针传递规则** 与 **GC 生命周期** 的交叉理解。
初级只知道"cgo 慢"；高级要能讲清楚：**为什么 Go 指针不能长期存在 C 内存里、`runtime.Pinner` 解决了什么合法性问题、Pin 之后 GC 行为有什么变化、以及忘记 Unpin 会带来什么后果**。这是中间件、数据库驱动、音视频等重度 CGO 场景的实测考点。

---

## 核心答案（30 秒版）

Go 的 cgo 指针规则规定：**C 代码不得在调用返回后继续持有 Go 指针**（不能把 Go 指针存进 C 的全局变量、结构体等）。

`runtime.Pinner`（Go 1.21 引入）就是为"必须让 C 长期持有 Go 指针"的场景提供的**合法逃生口**：

| 维度 | 说明 |
|------|------|
| 作用 | 把 Go 对象"钉住"，使其不被 GC 回收，且地址在 Unpin 前始终有效 |
| 合法性 | 被 Pin 的对象可以安全地存入 C 内存（`cgocheck` 允许） |
| 生命周期 | `Pin` 之后必须显式 `Unpin`，否则对象无法回收 → 内存泄漏 |
| 代价 | Pin 的对象会被当作 GC root 扫描，Pin 太多会拖慢 GC |

一句话：**Pinner 把"对象还活着"这件事从 Go 侧显式告诉运行时，从而让 C 侧持有 Go 指针变得合法。**

---

## 深度展开

### 1. 为什么需要 Pinner：cgo 指针规则的边界

```go
// ❌ 违反 cgo 指针规则：C 侧保存了 Go 指针
/*
// C 代码
static void *g_saved;
void save(void *p) { g_saved = p; }
*/
func bad(p *int) {
    C.save(unsafe.Pointer(p)) // C 返回后仍然持有 p → 运行时可能 panic 或静默 UB
}
```

规则的本质是两条：

1. **Go 指针不能长期存活在 C 内存中**（C 无法在 GC 时被扫描为根）；
2. **GC 不知道 C 侧还引用着这个对象**，可能提前回收，也可能（未来引入移动式 GC 后）移动它。

`Pinner` 的作用就是把这两条都堵上：既让运行时知道"这个对象别回收"，也让 `cgocheck` 认可"存进 C 内存是安全的"。

### 2. 基本用法

```go
import "runtime"

type ctx struct {
    buf  []byte
    cb   C.callback_t
    pin  runtime.Pinner
}

func newCtx(n int) *ctx {
    c := &ctx{buf: make([]byte, n)}
    // 1) 先把对象 Pin 住
    c.pin.Pin(c)
    // 2) 安全地把 Go 指针交给 C 长期保存
    C.register(unsafe.Pointer(c))
    return c
}

func (c *ctx) Close() {
    // 3) C 侧不再使用后，解除 Pin
    C.unregister(unsafe.Pointer(c))
    c.pin.Unpin()
}
```

关键点：

- `Pin(pointer any)` 的参数必须是 **Go 指针**（指向堆对象），否则 panic；
- 一个 `Pinner` 可以 Pin 多个对象，`Unpin()` 一次性全部解除；
- `Pinner` 自身如果被 GC 回收，其 Pin 也会随之失效（所以通常挂在被 Pin 的结构体或长生命周期对象上）。

### 3. Pin 之后 GC 行为的变化

| 行为 | 普通对象 | 被 Pin 的对象 |
|------|---------|--------------|
| 可达性 | 由引用关系决定 | **始终可达**（隐式根），GC 不会回收 |
| 地址 | heap 上稳定（Go 目前不移动堆对象） | 同样稳定，且语义上有保证 |
| GC 扫描 | 按引用链扫描 | 作为根额外扫描，可能增加开销 |
| 泄漏风险 | 无引用即回收 | **忘记 Unpin 就永久泄漏** |

> 注意：Go 当前是**非移动式 GC**，所以"地址不会变"在物理上本来就成立；
> `Pinner` 真正提供的价值是 **生命周期保证 + 指针规则的合法性**，而不是"防止复制"。
> 这也解释了为什么 Pin/Unpin 是**契约式**的：运行时无法替你判断 C 什么时候不再用这个指针。

### 4. 常见坑

```go
// ❌ 坑 1：Pin 了栈变量
func f() {
    var p runtime.Pinner
    var x int = 1
    p.Pin(&x) // 栈对象生命周期由编译器决定，不能这样交给 C 长期持有
}

// ❌ 坑 2：只 Pin 不 Unpin —— 对象永不回收
// ❌ 坑 3：把 Pinner 当成锁
//    Pinner 不提供任何互斥语义，并发访问仍需自己用 Mutex/atomic 保护
```

- **必须配合 `runtime.KeepAlive`**：如果在 C 调用之后代码里已不再引用对象，编译器可能认为它"已经死了"，需要用 `runtime.KeepAlive(c)` 把生命周期延长到调用点之后。
- **Unpin 之后 C 侧不能再访问**：否则就是 use-after-free。
- **不要滥用**：纯 Go 逻辑里用 Pin 等于主动制造"永久可达"对象，是典型的性能反模式。

---

## 生产实践：什么时候真的需要 Pinner

| 场景 | 是否用 Pinner |
|------|--------------|
| C 库回调需要一个 user data 指针 | ✅ 典型场景（也可用 `cgo.Handle`，见语言机制篇）|
| C 侧只读取、调用返回即用完 | ❌ 直接传即可，无需 Pin |
| 需要一个安全的整数句柄而非指针 | ❌ 用 `cgo.Handle` 更安全 |
| 想把 Go slice 的地址传给 C 做异步 IO | ✅ 必须 Pin + 保证 C 不再写 |

> 经验法则：**能不用指针就不用指针，能用 `cgo.Handle`（整数句柄）就不要用 `Pinner`（真实指针）**。
> Handle 不会被 GC 移动、不涉及指针规则，泄漏可控；Pinner 是"确实要把真实指针交给 C"时的最后手段。

---

## 高频追问

**Q：Go 堆对象本来就不移动，那 `Pinner` 是不是多此一举？**

不是。地址稳定只是"物理事实"，而 cgo 规则要求的是**语义上的安全保证**：
GC 必须知道 C 侧还持有这个对象，否则可能提前回收（C 侧拿到悬空指针）。
`Pinner` 就是把这个事实变成运行时可见的根集合，同时解除 `cgocheck` 的限制。

**Q：`Pinner` 和 `sync.Pool` / `weak.Pointer` 有什么区别？**

| 机制 | 目的 |
|------|------|
| `Pinner` | 让对象**不能被回收**，并允许 C 持有指针 |
| `weak.Pointer` | 让对象**可被回收**，只保留弱引用（缓存场景）|
| `sync.Pool` | 复用对象，降低分配压力，随时可被 GC 清空 |

**Q：忘记 Unpin 的后果怎么排查？**

用 `runtime/metrics` 观察 `/memory/classes/heap/objects:bytes` 是否单调上涨；
配合 `pprof heap` 看对象是否始终是 root 可达。CGO 场景下也可以用 `GODEBUG=cgocheck=2` 辅助定位非法指针传递。

---

## 面试话术

> "cgo 的指针规则不允许 C 长期持有 Go 指针。Go 1.21 的 `runtime.Pinner` 提供了合法通道：
> Pin 住对象让 GC 视其为根、地址有效，用完必须 Unpin，否则对象永久泄漏；
> 它的本质是"生命周期契约"，不是锁也不防复制。能用 `cgo.Handle` 整数句柄时优先用它。"

---

## 延伸阅读

- [Go 1.21 Release Notes — runtime.Pinner](https://go.dev/doc/go1.21#runtime)
- [cgo 指针传递规则（官方文档）](https://pkg.go.dev/cmd/cgo#hdr-Passing_pointers)
- [runtime.Pinner 文档](https://pkg.go.dev/runtime#Pinner)

---

**[← 上一篇：runtime.free](./12-12-runtime-free.md)** · **[返回模块首页 →](../README.md)**
