# 值接收者 vs 指针接收者：方法集规则与一致性原则

> 考察频率：★★★★☆  优先级：P1
> 关键词：value receiver、pointer receiver、method set、consistency rule、interface satisfaction

---

## 面试官考察意图

这道题表面在问"什么时候用值接收者，什么时候用指针接收者"，实际上考察候选人对 **Go 类型系统底层规则**的理解。初级只会凭直觉写 `func (s *Struct) Method()`；高级要能讲清楚 **method set 的精确定义、接口实现的条件、以及为什么 Go 社区强烈推荐"一致性原则"**。这是区分会照搬教程和真正理解语言设计的关键考题。

---

## 核心答案（30 秒版）

**基本规则：**

| 场景 | 选择 | 原因 |
|------|------|------|
| 需要修改receiver | 指针接收者 | 值接收者是副本，修改无效 |
| struct 很大 | 指针接收者 | 避免每次调用都拷贝整个结构体 |
| 方法不需要修改receiver | 值或指针都行 | 但需遵循一致性原则 |
| 方法需要修改receiver | 必须是指针接收者 | 值接收者无法修改原对象 |

**Method Set 规则（核心考点）：**

- `T` 的方法集 = T 的所有具有 **值接收者** 的方法
- `*T` 的方法集 = T 的所有方法（值和指针接收者都有）
- 因此 `*T` 可以 satisfy 要求 `T` 方法的接口，但 `T` 不能 satisfy 要求 `*T` 方法的接口

**一致性原则（Consistency Rule）：** 如果一个类型的任意一个方法需要指针接收者，那么所有方法都应该用指针接收者 —— 保持 method set 一致，避免混淆。

---

## 深度展开

### 1. 两种接收者的行为差异

```go
type Counter struct {
    n int
}

// 值接收者：操作的是副本
func (c Counter) Inc() {
    c.n++ // 修改的是副本，原始 Counter 不受影响
}

// 指针接收者：操作的是原始对象
func (c *Counter) Dec() {
    c.n-- // 修改的是原始对象
}
```

**验证代码：**

```go
func main() {
    c := Counter{n: 10}
    c.Inc() // n 仍然是 10（副本被修改后丢弃）
    fmt.Println(c.n) // → 10

    c.Dec() // n 变为 9
    fmt.Println(c.n) // → 9
}
```

### 2. Method Set 规则 —— 接口实现的本质

这是 Go 面试中最容易被忽视的核心概念：

```go
type Reader interface {
    Read(buf []byte) (n int, err error)
}

type Server struct {
    host string
}

// 情况1：值接收者方法
func (s Server) Read(buf []byte) (int, error) { /*...*/ }
// Server 的方法集包含 Read()
// *Server 的方法集也包含 Read()
var _ Reader = Server{}      // ✅ OK
var _ Reader = &Server{}     // ✅ OK

// 情况2：指针接收者方法
func (s *Server) Write(buf []byte) (int, error) { /*...*/ }
// Server 的方法集是空的（没有值接收者方法叫 Write）
// *Server 的方法集包含 Write()
var _ Reader = (*Server)(nil) // ❌ 编译错误：Server 不满足任何要求指针接收者的接口

type Writer interface {
    Write(buf []byte) (int, error)
}
var _ Writer = Server{}      // ❌ 编译错误
var _ Writer = &Server{}     // ✅ OK
```

**关键理解：**

```
T 的方法集 ⊆ *T 的方法集

* 的值接收者方法 → 同时出现在 T 和 *T 的方法集中
* 的指针接收者方法 → 只出现在 *T 的方法集中
```

这就是为什么 `json.Marshaler` 接口可以用值接收者实现，但如果某个方法用了指针接收者，你必须传指针才能满足接口。

### 3. 性能考虑：拷贝成本

```go
// 小结构体：值拷贝成本极低，值接收者更高效
type Point struct {
    X, Y float64
}
func (p Point) Distance(q Point) float64 { /*...*/ }
// Point 只有两个 float64，16 字节，寄存器传递，无需堆分配

// 大结构体：值拷贝代价高昂
type Server struct {
    config     Config       // 假设 2KB
    db         *sql.DB     // 8 字节指针
    cache      sync.Map    // 约 48 字节
    logger     *slog.Logger // 8 字节指针
    mu         sync.Mutex  // 24 字节
    shutdown   chan struct{} // 8 字节指针
    workers    []*Worker   // 切片头 24 字节
}
func (s *Server) Start(ctx context.Context) error { /*...*/ }
// 如果写成值接收者 func (s Server) Start()，每次调用拷贝 ~200+ 字节
```

**内存拷贝的成本：**

```
小结构体（< 128 bytes）→ 通常寄存器传递，零开销
中等结构体（128~1024 bytes）→ 栈上 memcpy，纳秒级
大结构体（> 1KB）→ 栈拷贝显著，应使用指针
```

### 4. 一致性原则（The Consistency Rule）

Go 社区公认的最佳实践：

```go
type BadExample struct {
    name string
}

// 混用 ❌ —— 违反一致性原则
func (b BadExample) GetName() string {
    return b.name
}

func (b *BadExample) SetName(name string) {
    b.name = name
}

// 问题：BadExample 本身只能调用 GetName()
// *BadExample 可以调用全部两个方法
// 用户不知道该传什么，API 不一致

type GoodExample struct {
    name string
}

// 全是指针接收者 ✅ —— 一致性
func (g *GoodExample) GetName() string {
    return g.name
}

func (g *GoodExample) SetName(name string) {
    g.name = name
}
// 无论用户传 GoodExample{} 还是 &GoodExample{}，都需要统一方式
```

**为什么需要一致性？**

```go
typeDescriber interface {
    Describe() string
}

type Mixed struct { name string }
func (m Mixed) Describe() string { return m.name } // 值接收者
type Pure struct { name string }
func (p *Pure) Describe() string { return p.name } // 指针接收者

var d1 typeDescriber = Mixed{}   // ✅ 值可以做
var d2 typeDescriber = &Pure{}   // ✅ 指针可以做

// 但是在集合中会有问题：
items := []typeDescriber{Mixed{}, &Pure{}}
for _, item := range items {
    fmt.Println(item.Describe())
}
// 看起来没问题...但如果接口增加更多方法就不一样了
```

### 5. 特殊情况：需要共享状态时必须用指针

```go
type Cache struct {
    store map[string][]byte
    size  int
}

func NewCache() *Cache {
    return &Cache{
        store: make(map[string][]byte),
    }
}

// ❌ 值接收者会导致每个副本拥有独立的 map
func (c Cache) Get(key string) ([]byte, bool) {
    val, ok := c.store[key] // 读取自己的副本
    return val, ok
}

// ✅ 指针接收者所有实例共享同一个 map
func (c *Cache) Get(key string) ([]byte, bool) {
    val, ok := c.store[key] // 读取共享的数据
    return val, ok
}
```

### 6. Go 源码中的实际模式

观察标准库的做法有助于建立判断直觉：

| 类型 | 方法数量 | 接收者 | 原因 |
|------|---------|--------|------|
| `bytes.Buffer` | ~20 | 混合 | G 有值也有指针，维护兼容性的历史包袱 |
| `strings.Builder` | ~5 | 混合 | 同上，字符串包发展较早 |
| `sync.Mutex` | 4 | 指针 | 需要获取锁地址 |
| `http.Server` | ~15 | 大部分指针 | 大型服务配置，需要修改状态 |
| `io.ReadWriter` 实现 | 各种 | 看需求 | 取决于是否需要写状态 |

**值得注意的案例——bytes.Buffer：**

```go
type Buffer struct {
    buf     []byte
    off     int
    lastRead readResult
}

func (b *Buffer) Len() int            // 指针接收者
func (b *Buffer) Read(p []byte) (n int, err error)  // 指针接收者
func (b *Buffer) Bytes() []byte       // 值接收者 ❗
```

为什么 `Bytes()` 可以是值接收者？因为它只需要返回内部的 slice，不需要修改 Buffer 的状态。虽然它破坏了纯粹性，但在 Go 1.x 兼容性约束下，标准库选择了折中方案。

---

## 🗣️ 面试话术

**一句话记住**：改状态用指针接收者、不改状态看大小和一致性——小结构体且纯读取可用值接收者，否则全部统一用指针。Go 面试爱考 method set 规则，核心就是 `T 的方法集 ⊆ *T 的方法集`。

---

## 🔗 延伸阅读

- [Effective Go: Pointers vs Values](https://go.dev/doc/effective_go#pointers_vs_values)
- [Go Blog: Package docs: godoc comments](https://go.dev/blog/package-docs)
- [Go Spec: Method sets](https://go.dev/ref/spec#Method_sets)
