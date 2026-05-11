# make vs new：底层原理与适用场景

> 考察频率：★★★☆☆  优先级：P1  
> 关键词：make、new、零值初始化、指针、引用类型、内存分配

---

## 面试官考察意图

考察候选人对 Go 内存分配机制的深层理解。
初级只知道"make 用于 slice/map/channel"，高级要能讲清楚 **new 只做零值初始化、make 额外初始化底层结构、两者的底层汇编差异、以及生产中如何根据类型选择**，并能通过源码解释为什么 new 返回指针而 make 返回值本身。

---

## 核心答案（30 秒版）

| 函数 | 分配类型 | 返回值 | 初始化 |
|------|----------|--------|--------|
| `new(T)` | 任意类型 T | `*T`（指针）| 仅零值 |
| `make(T, ...)` | slice / map / channel | T（值，不是指针）| **完整初始化** |

```go
// new：返回指向零值内存的指针
p := new(int)    // *int = 0

// make：返回已初始化的引用类型
s := make([]int, 0)   // len=0, cap=0, 空切片但不是 nil
m := make(map[string]int)  // 空 map，不是 nil
ch := make(chan int, 1)    // 有缓冲 channel
```

---

## 深度展开

### 1. new：纯粹的零值分配

**底层实现（伪代码）：**

```go
// runtime/malloc.go
func newobject(typ *_type) unsafe.Pointer {
    return mallocgc(typ.size, typ, true)  // 清零分配
}

// 等价的 Go 函数
func new(T any) *T {
    var t T
    return &t  // 编译器优化：用 mallocgc 分配并清零
}
```

**关键特性：**
- `new(T)` 分配大小为 `sizeof(T)` 的零值内存
- 返回 `*T` 类型的指针
- **不调用类型的构造函数/初始化方法**
- 只做一件事：**分配 + 清零**

**使用场景：**

```go
// 场景1：创建指向结构体的指针
type Config struct {
    Name string
    Port int
}

cfg := new(Config)
// cfg 类型是 *Config
// cfg.Name = ""（零值），cfg.Port = 0（零值）

// 场景2：创建指向基础类型的指针
count := new(int)  // count 类型是 *int，值为 0

// 场景3：几乎等价于取地址
var i int
p1 := &i            // 栈上分配
p2 := new(int)      // 堆上分配（可能被逃逸分析优化到栈）
```

### 2. make：引用类型的完整初始化

**底层实现（伪代码）：**

```go
// runtime/makeslice.go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
    mem := mallocgc(et.size*uintptr(cap), et, true)
    return mem
}

// 初始化 slice 的内部结构
type slice struct {
    array unsafe.Pointer  // 指向底层数组
    len   int             // 当前长度
    cap   int             // 容量
}

// make 额外做的事情：
// 1. 分配底层数组内存
// 2. 初始化 slice struct（array/len/cap）
// 3. 返回 slice 值（不是指针）
```

**make 对不同类型的处理：**

| 类型 | make 做了什么 | 返回值 |
|------|---------------|--------|
| `[]T` | 分配底层数组，初始化 len/cap | `[]T`（值） |
| `map[K]V` | 分配 map 头，初始化哈希表 | `map[K]V`（值） |
| `chan T` | 分配 channel 结构，初始化队列 | `chan T`（值） |

```go
// slice：分配底层数组
s1 := make([]int, 0)      // len=0, cap=0
s2 := make([]int, 3)       // len=3, cap=3, 元素为 0
s3 := make([]int, 0, 10)   // len=0, cap=10

// map：初始化哈希表头
m := make(map[string]int)  // 空 map，nil map 也能读写但会 panic
m["a"] = 1

// channel：初始化队列
ch1 := make(chan int)       // 无缓冲
ch2 := make(chan int, 10)   // 有缓冲，容量 10
```

### 3. 底层汇编对比

```go
// 源代码
var p *int
p = new(int)

var s []int
s = make([]int, 0)
```

**汇编（go tool compile -S）：**

```asm
// new(int) → mallocgc + 清零
0x000d  00013 (main.go:5)  CALL    runtime.newobject
0x0012  00018 (main.go:5)  MOVQ    (SP), AX

// make([]int, 0) → makeslice（分配 + 初始化）
0x000d  00013 (main.go:8)  CALL    runtime.makeslice
```

### 4. nil vs 空切片：经典陷阱

```go
// nil slice：未分配内存
var s1 []int
fmt.Println(s1, len(s1), cap(s1))  // [] 0 0

// empty slice：分配了底层数组（但长度为 0）
s2 := []int{}
s3 := make([]int, 0)
fmt.Println(s2, len(s2), cap(s2))  // [] 0 0

// 关键区别：
// s1 == nil → true   （可以 append，但第一次 append 会触发扩容）
// s2 == nil → false  （已分配底层数组）
// s3 == nil → false

// 生产建议：用 make 代替空切片字面量，避免混淆
s := make([]int, 0)  // ✅ 语义清晰
s := []int{}         // ⚠️ 容易和 nil 混淆
```

### 5. new 与 make 的选择决策树

```
需要分配什么类型？
    │
    ├── 指针类型 / 基础类型（int/float/bool/string）
    │       └── new(T) ✅
    │
    ├── slice / map / channel
    │       └── make(T) ✅
    │
    └── 结构体 / 接口等复杂类型
            ├── 只想要零值对象 → new(T)
            ├── 想要完整初始化 → 逐字段初始化或构造函数
```

**生产中的惯用模式：**

```go
// 推荐：结构体用字面量初始化，不用 new
type Server struct {
    Name string
    Port int
}

// 方式1：new（返回指针，但字段都是零值）
s1 := new(Server)

// 方式2：字面量（推荐，语义清晰）
s2 := &Server{Name: "localhost", Port: 8080}

// 方式3：构造函数
func NewServer(name string, port int) *Server {
    return &Server{Name: name, Port: port}
}
```

---

## 高频追问

**Q：new 和 make 哪个更快？**

两者差距很小，new 分配后清零，make 分配后额外初始化底层结构。对于 slice/map/channel，如果不需要完整初始化再用 new 获取零值，再自己手动初始化，性能影响可忽略。**生产中不应为此优化而选型**，应根据语义选 make 或 new。

**Q：为什么 slice/map/channel 返回值而不是指针？**

因为这三个类型本身就是引用语义（底层是隐式指针）。slice 的 array/len/cap 是编译器管理的元数据，如果返回指针需要手动管理生命周期会很麻烦。

```go
// slice 的底层结构
type slice struct {
    array unsafe.Pointer  // 隐藏的底层指针
    len   int
    cap   int
}
// make 返回 slice 值，这个值拷贝时底层指针共享，非常高效
```

**Q：new 分配的内存一定在堆上吗？**

不一定。编译器做逃逸分析，如果变量的生命周期可以证明只在一个函数栈帧内，就会分配在栈上。

```bash
go build -gcflags="-m -m" main.go
# 观察 "does not escape" 或 "escapes to heap"
```

**Q：可以用 make 创建指针吗？**

不行。make 只适用于 slice/map/channel，强行用于其他类型会编译错误。

```go
make(*int)  // 编译错误：make of untyped pointer
new(*int)   // ✅ 正确
```

**Q：sync.Pool 的 New 字段返回的值为什么不能是指针？**

`sync.Pool.New` 返回 `interface{}`（即 `any`），可以是指针，但通常返回指针是为了避免 GC 开销。

```go
pool := sync.Pool{
    New: func() interface{} {
        return &bytes.Buffer{}  // ✅ 返回指针，减少 GC 压力
    },
}
```

---

## 延伸阅读

- [Go make 函数源码](https://github.com/golang/go/blob/master/src/runtime/makeslice.go)
- [Go new 函数源码](https://github.com/golang/go/blob/master/src/runtime/malloc.go)
- [Allocation and garbage collection in Go](https://go.dev/blog/malloc)（官方博客）
- [Understanding Go's memory allocation](https://www.ardanlabs.com/blog/2017/06/understanding-go-allocation.html)

---

**[← 上一篇：闭包原理与陷阱](./19-closure.md)** · **[下一篇：string 与 []byte 深度解析 →](./18-string-byte.md)**
