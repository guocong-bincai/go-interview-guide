[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [💎 语言机制](../README.md)

---

# Go 1.22/1.23 循环与迭代器：循环变量语义、range-over-int、range-over-func

## 面试官考察意图

考察候选人对 Go 语言近两个大版本新特性的掌握程度。
Go 1.22 修复了**困扰 Go 开发者多年的循环变量闭包 bug**，这是面试中的高频坑点。Go 1.23 的 **range-over-function 迭代器**则是函数式编程与 Go 的结合，代表了 Go 在泛型和抽象能力上的新方向。高级工程师不仅要会用，还要能讲清楚**为什么这样设计、历史背景是什么**。

---

## 核心答案（30 秒版）

| 特性 | 版本 | 核心变化 |
|------|------|----------|
| **循环变量语义变更** | Go 1.22 | 每次迭代创建新变量，修复闭包 bug |
| **range-over-int** | Go 1.22 | `for i := range n` 等价于 `for i := 0; i < n; i++` |
| **range-over-function** | Go 1.23 | 用函数替代数组/slice 作为 range 的数据源 |
| **泛型类型别名** | Go 1.23（预览） | `type IntList = list[int]` 简化泛型引用 |

**历史背景（必问）：** Go 1.21 及之前，for 循环的变量是**函数级作用域**，所有迭代共享同一个变量地址。这导致经典的闭包 bug：

```go
// Go 1.21 及之前：所有 goroutine 共享变量 i，打印 3, 3, 3
// Go 1.22 及之后：每次迭代创建新变量，打印 0, 1, 2
for i := range []int{0, 1, 2} {
    go func() { fmt.Print(i) }()
}
```

---

## 一、Go 1.22 之前的循环变量语义（历史 + 面试坑点）

### 1.1 经典 Bug：闭包捕获循环变量

```go
// 面试官问：这段代码打印什么？
func main() {
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Print(i, " ") // 打印：2 2 2
        }()
    }
    wg.Wait()
}
```

**原因分析：**

Go 1.21 及之前，for 循环的循环变量 `i` 是**函数级作用域**（准确说是**循环体可见的外层变量**），所有 goroutine 共享同一个 `i`。当主 goroutine 完成循环时，`i` 的值已经是 3，所以三个 goroutine 都打印 3。

```
Go 1.21 之前的变量作用域：
┌──────────────────────────────────────┐
│  i (共享变量，地址固定)              │
│   ┌────────┐  ┌────────┐  ┌────────┐ │
│   │goroutin1│  │goroutin2│  │goroutin3│ │
│   │读取i=3 │  │读取i=3 │  │读取i=3 │ │
│   └────────┘  └────────┘  └────────┘ │
└──────────────────────────────────────┘
```

### 1.2 错误的修复方式

```go
// ❌ 错误修复1：传参也不行，因为参数是 i 的副本
for i := 0; i < 3; i++ {
    go func(val int) { fmt.Print(val) }(i) // 仍然可能有问题（取决于时机）
}

// ❌ 错误修复2：只在极端情况下能用
```

### 1.3 正确的修复方式（Go 1.21 及之前）

```go
// ✅ 方式1：显式创建新变量（Go 1.21 及之前的正确写法）
for i := 0; i < 3; i++ {
    i := i // 创建循环体内的副本
    go func() { fmt.Print(i) }()
}

// ✅ 方式2：通过函数参数捕获（Go 1.21 及之前的正确写法）
for i := 0; i < 3; i++ {
    go func(val int) { fmt.Print(val) }(i)
}
```

### 1.4 Go 1.21 过渡支持：loopvar 实验

Go 1.21 引入了 `GOEXPERIMENT=loopvar` 编译选项，允许用户提前体验新语义。Go 1.22 默认启用此行为。

---

## 二、Go 1.22 循环变量语义变更

### 2.1 核心变化：每次迭代创建新变量

Go 1.22 起，**每次 for 循环迭代都会创建新的循环变量**，彻底解决了闭包 bug。

```go
func main() {
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Print(i, " ") // Go 1.22+：打印 0 1 2
        }()
    }
    wg.Wait()
}
```

```
Go 1.22+ 的变量作用域：
┌─────────────────────────────────────────┐
│  迭代1: i₁ 创建，goroutine 捕获 i₁      │
│  迭代2: i₂ 创建（i₁ 不再可见）          │
│  迭代3: i₃ 创建（i₂ 不再可见）          │
└─────────────────────────────────────────┘
```

### 2.2 range 循环的同样变化

```go
// Go 1.22+：range 循环也是每次迭代创建新变量
for i, v := range []int{10, 20, 30} {
    go func() {
        fmt.Print(i, v) // 打印 0 10, 1 20, 2 30（正确）
    }()
}
```

### 2.3 range-over-int：遍历整数范围

Go 1.22 新增了对整数的 `range` 支持，不再需要 `for i := 0; i < n; i++` 这种传统写法。

```go
// Go 1.22+：等价于 for i := 0; i < 5; i++
for i := range 5 {
    fmt.Println(i) // 0, 1, 2, 3, 4
}

// 传统写法（仍然有效，但 range-int 更简洁）
for i := 0; i < 5; i++ {
    fmt.Println(i)
}
```

**使用场景：**

```go
// 场景1：简洁的倒序循环
for i := range 10 {
    fmt.Print(10 - i - 1, " ") // 9 8 7 6 5 4 3 2 1 0
}

// 场景2：替代 for i := 0; i < n; i++（更符合 Go 风格）
func reverse(s string) string {
    runes := []rune(s)
    for i := range len(runes) {
        runes[i], runes[len(runes)-1-i] = runes[len(runes)-1-i], runes[i]
    }
    return string(runes)
}
```

### 2.4 面试追问：Go 为什么这样改？

**1. 解决历史兼容性问题**
Go 1.22 之前的循环变量语义是社区长期抱怨的问题。官方调查发现，约 10% 的 goroutine 使用代码存在类似 bug。这个修复虽然破坏了极少量代码（官方预计 < 0.5%），但大幅提升了语言安全性。

**2. 为什么 Go 不默认用值拷贝而是用新变量？**
Go 团队认为"每次迭代创建新变量"比"变量每次迭代重新赋值"更符合直觉。如果每次迭代重新赋值，仍然会导致闭包捕获到意外的值。

**3. 这个改变会影响性能吗？**
不会。对于整数循环变量，编译器会进行优化，实际生成的代码与之前相同（只是语义变了）。

---

## 三、Go 1.23 range-over-function（迭代器）

### 3.1 背景：为什么需要迭代器函数？

Go 的 `range` 关键字原本只能遍历数组、slice、map、channel。对于复杂的数据结构（如树、图、流），如果要支持 `for range` 语法，需要先将整个数据结构加载到内存中，浪费资源。

Go 1.23 引入的 **range-over-function** 允许用**函数**作为 `range` 的数据源，从而实现惰性迭代（lazy iteration）。

### 3.2 迭代器函数签名

`range` 子句现在支持三种函数签名：

```go
// 无值迭代器
func(func() bool)

// 单值迭代器
func(func(K) bool)

// 双值迭代器
func(func(K, V) bool)
```

iterator 函数每调用一次，返回一对值（或者用 `break` 返回 `false` 终止迭代）。

### 3.3 标准库 iter 包

Go 1.23 引入了 `iter` 包，提供常用迭代器工具函数：

```go
package iter

// Seq 是单值迭代器的类型别名
type Seq[V any] func(yield func(V) bool)

// Seq2 是双值迭代器的类型别名
type Seq2[K, V any] func(yield func(K, V) bool)
```

### 3.4 基本使用示例

```go
package main

import "fmt"

func main() {
    // 遍历 0 到 4
    for i := range 5 {
        fmt.Print(i, " ") // 0 1 2 3 4
    }

    // 自定义迭代器：遍历二叉树
    type Tree struct {
        Val   int
        Left  *Tree
        Right *Tree
    }

    // 前序遍历迭代器
    func inOrder(root *Tree) func(func(int) bool) {
        return func(yield func(int) bool) {
            var walk func(*Tree)
            walk = func(n *Tree) {
                if n == nil {
                    return
                }
                if !walk(n.Left) {
                    return
                }
                if !yield(n.Val) {
                    return
                }
                walk(n.Right)
            }
            walk(root)
        }
    }

    tree := &Tree{Val: 2,
        Left:  &Tree{Val: 1},
        Right: &Tree{Val: 3},
    }

    for v := range inOrder(tree) {
        fmt.Print(v, " ") // 1 2 3
    }
}
```

### 3.5 迭代器的组合

```go
// 使用 iter.Seq2 遍历 map
func iterateMap[K comparable, V any](m map[K]V) func(func(K, V) bool) {
    return func(yield func(K, V) bool) {
        for k, v := range m {
            if !yield(k, v) {
                return
            }
        }
    }
}

for k, v := range iterateMap(map[string]int{"a": 1, "b": 2}) {
    fmt.Println(k, v)
}
```

### 3.6 常见面试追问

**Q1：range-over-func 和泛型有什么关系？**

A：迭代器函数使用了泛型类型参数（`func(func(V) bool)` 中的 `V`）。Go 1.23 的 `iter.Seq[V any]` 和 `iter.Seq2[K, V any]` 就是泛型类型别名。这个特性依赖于 Go 1.18 引入的泛型基础设施。

**Q2：迭代器函数和 Channel 相比，有什么优劣？**

| 维度 | 迭代器函数 | Channel |
|------|------------|---------|
| 惰性求值 | ✅ 原生支持 | ✅ 天然支持 |
| 背压控制 | ✅ 通过 yield 返回值控制 | ✅ 发送阻塞自然背压 |
| 并发安全 | 取决于实现 | ✅ 原生并发安全 |
| 错误处理 | 复杂（需要额外机制） | 简单（close + 读channel） |
| 适用场景 | 树、图、文件流、数据库游标 | 生产者-消费者、事件流 |

**Q3：Go 的迭代器和 Python/Java 的迭代器有什么不同？**

Python/Java 的迭代器是**对象**，需要实现特定接口（`__iter__` / `Iterator`）。Go 的迭代器是**函数**，不需要额外的类型约束，更轻量，也更符合 Go 的习惯。

**Q4：如何在生产代码中使用 range-over-func？**

Go 1.23 的 range-over-func 仍相对较新，生产使用建议：
- 用于自定义数据结构的遍历抽象（如 ORM 的游标、树的遍历）
- 用于处理大数据流，避免一次性加载到内存
- 避免在性能关键路径中使用（函数调用开销比直接循环大）

---

## 四、Go 1.23 Timer 变化（容易忽略的破坏性变更）

### 4.1 Timer/Ticker 变化

Go 1.23 对 Timer 和 Ticker 做了两个重要变更：

**变更1：未停止的 Timer/Ticker 可被 GC 回收**

```go
// Go 1.22 及之前：即使没有 Stop()，Timer 也不会被 GC 回收
// Go 1.23：未 Stop() 且不再被引用的 Timer 会被 GC 回收

var timer *time.Timer

func init() {
    timer = time.NewTimer(10 * time.Second)
    // Go 1.22：timer 永远不会被 GC（因为 runtime 持有引用）
    // Go 1.23：如果没有其他地方引用 timer，它可能会在后台被 GC
}

// ✅ Go 1.23 正确做法：如果需要持久的 Timer，必须确保有外部引用
// 或者使用 time.NewTicker + Stop() + defer
```

**变更2：Timer/Ticker 的 channel 变为无缓冲**

```go
// Go 1.22 及之前：Timer 的 C 字段是 cap=1 的 channel
// Go 1.23：Timer/Ticker 的 channel 是无缓冲的（cap=0）

timer := time.NewTimer(1 * time.Second)
fmt.Println(cap(timer.C)) // Go 1.22: 1; Go 1.23: 0

// 影响：Reset 和 Stop 的语义更简单，不会有"旧值残留"问题
```

**为什么这是破坏性变更？**

某些代码依赖 Timer channel 的 cap=1 特性来"预设置" Reset：
```go
// Go 1.22 特殊用法（依赖缓冲 channel）
timer := time.NewTimer(0)
timer.Reset(1 * time.Second) // 在 Go 1.22 可以，但语义不清晰
```

Go 1.23 修复了这种"Reset 时可能收到旧值"的问题，使 Timer 的行为更加可预测。

### 4.2 面试追问

**Q：Go 1.23 Timer 可被 GC 回收意味着什么？**

A：这意味着如果你的代码**依赖 Timer 的副作用（channel 发送）来触发某个操作**，而没有在 Timer 上保持引用，Go 1.23+ 可能会在 Timer 触发前就将其 GC 掉，导致行为不一致。

**正确做法：** 如果需要 Timer 触发某个操作，始终通过 `time.Ticker` 或明确持有 `*time.Timer` 的引用，并使用 `defer timer.Stop()`。

---

## 五、综合面试题

### 题目 1：循环变量闭包（必问）

```go
// 问：Go 1.22 前后，这段代码分别打印什么？
func main() {
    vals := []int{1, 2, 3}
    for _, v := range vals {
        go func() {
            fmt.Print(v) // ?
        }()
    }
    time.Sleep(time.Second)
}
```

**答案：**
- Go 1.21：`3 3 3`（共享变量 v）
- Go 1.22+：`1 2 3`（每次迭代新变量）

### 题目 2：range-over-int 的作用域

```go
// 问：这段代码会打印什么？
for i := range 3 {
    fmt.Print(i)
    i := 0 // 重新声明同名变量，合法吗？
    // 答：合法，这是循环体内的变量 shadowing
}
```

### 题目 3：迭代器与泛型结合

```go
// 问：这个函数的返回类型是什么？
func squares(n int) func(func(int) bool) {
    return func(yield func(int) bool) {
        for i := 0; i < n; i++ {
            if !yield(i * i) {
                return
            }
        }
    }
}

// 用 for range 遍历
for sq := range squares(5) {
    fmt.Print(sq, " ") // 0 1 4 9 16
}
```

---

## 六、面试 checklist

- [ ] 能解释 Go 1.22 之前循环变量共享导致闭包 bug 的原因
- [ ] 能画出 Go 1.22 前后循环变量作用域的变化
- [ ] 能写出 Go 1.22 range-over-int 的至少两个使用场景
- [ ] 能解释 range-over-function 的函数签名含义
- [ ] 能对比 Channel 和迭代器函数的适用场景
- [ ] 能说明 Go 1.23 Timer GC 回收变更的影响
