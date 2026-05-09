[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [💎 语言机制](../README.md)

---

# Go 内置函数 `new`：从基础到 Go 1.26 重大增强

## 面试官考察意图

考察候选人对 Go 内置函数的理解深度。多数候选人只知道 `new(T)` 创建一个指向类型 T 的零值的指针，但不了解其本质是绕过逃逸分析直接申请内存、以及 Go 1.26 对 `new()` 的重要增强（支持表达式作为参数）。高级工程师要能讲清楚 **new 的底层实现、与 make 的区别、以及 Go 1.26 的设计动机**。

---

## 核心答案（30 秒版）

| 版本 | new() 用法 | 核心变化 |
|------|-----------|---------|
| Go 1.0+ | `new(T)` — T 必须是类型名 | 创建 T 的零值指针 |
| **Go 1.26** | `new(expr)` — expr 可以是任意表达式 | 表达式求值后创建变量 |

**本质：** `new()` 是唯一能绕过编译器逃逸分析、将变量分配到堆上的显式手段（直接调用 runtime.mallocgc）。`make()` 仅用于 slice、map、channel 的初始化。

---

## 一、`new()` 基础：创建零值指针

### 1.1 基本用法

```go
p := new(int)    // *int，值为 0，指针指向堆（也可能栈，看编译器决策）
q := new(string) // *string，值为 ""（空字符串）
r := new([]int)  // *[]int，值为 nil
```

**常见误解：** `new(int)` 等价于 `var i int; &i`，只是前者返回一个指针。但编译器会进行**逃逸分析**，决定分配到堆还是栈。

### 1.2 `new()` vs `make()` 对比

| | `new(T)` | `make(T, ...)` |
|---|---|---|
| **返回类型** | `*T` | `T`（不是指针） |
| **适用类型** | 任意类型 | slice、map、channel、interface |
| **分配位置** | 堆或栈（逃逸分析决定） | 总是堆 |
| **初始化** | 零值 | 非零值（slice len=0 cap；map/ch 有缓冲） |
| **用途** | 绕过逃逸分析 | 只能用于引用类型 |

```go
// 常见错误：make 用于 channel
ch := make(chan int) // ✅ 正确
ch := new(chan int)  // ❌ 编译错误：new 返回 *chan int，不是 chan int
```

### 1.3 `new()` 的逃逸分析

```go
// 示例 1：new 在当前函数逃逸
func foo() *int {
    p := new(int) // 逃逸到堆（返回到外部）
    return p
}

// 示例 2：new 在函数内使用，不逃逸
func bar() {
    p := new(int)
    *p = 42
    // p 不逃逸，分配在栈上，函数返回后自动释放
}
```

**面试追问：** 什么时候用 `new()` 而不是直接变量 + 取地址？

```go
// 方式1：直接变量
i := 42
p := &i // 栈变量，不能返回（编译错误或逃逸）

// 方式2：new（显式堆分配，返回安全）
func factory() *int {
    p := new(int)
    *p = 42
    return p
}
```

---

## 二、Go 1.26 增强：`new(expr)` 表达式参数

### 2.1 动机与使用场景

Go 1.26 解除了 `new()` 操作数必须是**类型字面量**的限制，现在可以是**任意表达式**。这解决了序列化场景中的常见痛点。

**典型场景：序列化中的可选字段**

```go
import "encoding/json"

type Person struct {
    Name string   `json:"name"`
    Age  *int     `json:"age"` // 指针表示可选：nil = 未知
}

func personJSON(name string, born time.Time) ([]byte, error) {
    return json.Marshal(Person{
        Name: name,
        // Go 1.26 之前：必须分两步
        Age:  new(yearsSince(born)),  // new() 接收表达式！
    })
}

func yearsSince(t time.Time) int {
    return int(time.Since(t).Hours() / 365.25)
}
```

**Go 1.26 之前的等价写法（繁琐）：**

```go
func personJSON(name string, born time.Time) ([]byte, error) {
    age := yearsSince(born)
    return json.Marshal(Person{
        Name: name,
        Age:  &age, // 需要预先声明变量，再取地址
    })
}
```

### 2.2 语法变化对比

```go
// Go 1.25 及之前：操作数必须是类型
ptr := new(int)
ptr := new(string)
ptr := new(MyStruct)

// Go 1.26+：操作数可以是任意表达式
ptr := new(int(42))         // 等价于 new(int)，但更明确
ptr := new(yearsSince(t))   // ✅ 新：表达式
ptr := new(someFunc())
ptr := new(*int)            // ✅ 新：创建 int 指针的指针
```

### 2.3 底层实现

`new()` 无论参数是类型还是表达式，底层都调用 `runtime.newobject()` → `mallocgc()`：

```go
// new(T) 的编译时翻译
p := new(T)
// ↓ 编译器翻译
var buf [sizeof(T)]byte
p := &buf                  // 栈上临时空间（可能被优化）
// ↓（逃逸分析决定）
p := runtime.newobject(T)   // 分配到堆
```

对于 `new(expr)`：
```go
p := new(yearsSince(t))
// ↓ 编译时翻译
tmp := yearsSince(t)        // 表达式先求值
p := new(typeof(tmp))       // 用表达式结果的类型分配
*p = tmp                   // 赋值
```

### 2.4 面试高频追问

**Q：`new()` 和 `&T{}`（结构体字面量）有什么区别？**

| | `new(T)` | `&T{}` |
|---|---|---|
| 初始化 | 零值 | 字段按字面量赋值 |
| 返回 | `*T` | `*T` |
| 逃逸 | 可能栈分配 | 可能栈分配 |
| 可读性 | 适合字段全零值 | 适合部分字段显式赋值 |

```go
p1 := new(MyStruct)         // 所有字段零值
p2 := &MyStruct{Name: "a"} // Name="a"，其他字段零值
```

**Q：`new()` 分配的内存何时释放？**

由 Go GC 管理。`new()` 分配的内存通过 GC 回收，没有固定生命周期 —— GC 标记为垃圾时回收（通常在下一次 GC cycle）。

---

## 三、`new()` 的工程实践

### 3.1 绕过栈逃逸（显式堆分配）

某些场景需要强制堆分配以避免栈空间不足：

```go
// 大数组：栈空间通常只有 2KB，直接声明会栈溢出
func createLargeArray() *[1000000]int {
    return new([1000000]int) // 强制堆分配
}
```

### 3.2 泛型中的 `new()`

```go
// Go 1.26+：泛型工厂函数
func New[T any]() *T {
    return new(T) // 创建任意类型的零值指针
}

// 使用
p := New[int]()      // *int，值为 0
s := New[string]()   // *string，值为 ""
```

### 3.3 sync.Pool 的类似场景

`new()` 适合那些不需要池化的简单分配：

```go
// 不适合 sync.Pool 的场景（状态无关）
var global = new(someConfig) // 全局单例，GC 不回收
```

---

## 四、高频追问汇总

| 问题 | 核心答案 |
|------|---------|
| `new()` 分配在堆还是栈？ | 由编译器逃逸分析决定，不确定 |
| `new([]int)` 和 `make([]int, 0)` 哪个好？ | `new([]int)` 返回 `*[]int`（指向 nil slice 的指针），`make` 返回 `[]int`，用途不同 |
| `new()` 有性能开销吗？ | 相比栈分配有开销（mallocgc），但有 GC 优化 |
| `new(expr)` 和 `&expr` 哪个更好？ | `new(expr)` 更明确表示"新建并初始化"，`&expr` 可能引用已存在变量 |

---

## 延伸阅读

- [Go 1.26 Release Notes - Language Changes](https://go.dev/doc/go1.26#language)
- [Go Spec: new built-in function](https://go.dev/ref/spec#Allocation)
- [Compiler implementation of new(T) expressions](https://github.com/golang/go/blob/master/src/cmd/compile/README.md)

[🏠 返回首页](../../../README.md)
