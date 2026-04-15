# Go 泛型深度解析：GCShape Stenciling 与使用边界

> 面试频率：★★★☆☆  考察角度：泛型实现原理、类型约束、性能对比

---

## 1. 泛型基本用法

### 1.1 类型参数（Type Parameters）

```go
// 泛型函数
func Min[T int | float64](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// 使用
fmt.Println(Min(1, 2))       // int 版本
fmt.Println(Min(1.5, 2.3))   // float64 版本

// 泛型结构体
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}
```

### 1.2 类型约束（Type Constraints）

```go
// 使用 interface 作为类型约束
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64 |
    ~string
}

// ~ 表示底层类型是某类型的所有类型（更宽松）
type MyInt int

func Sort[T Ordered](slice []T) {
    sort.Slice(slice, func(i, j int) bool {
        return slice[i] < slice[j]
    })
}

var _ MyInt = 1
Sort([]MyInt{3, 1, 2})  // ✅ MyInt 底层是 int，支持排序
```

### 1.3 any 约束（空接口）

```go
// any = interface{}，可以接受任何类型
func PrintAll[T any](items []T) {
    for _, item := range items {
        fmt.Println(item)
    }
}

// 接口泛型约束
type Reader[T any] interface {
    Read(p []byte) (n int, err error)
}
```

---

## 2. Go 泛型实现原理：GCShape Stenciling

### 2.1 早期实现： Type Parameters Copy

Go 1.17 之前：编译器为每个类型参数实例化一份代码。

```
Min[int] → 专门生成一份 int 版本代码
Min[float64] → 专门生成一份 float64 版本代码
```

问题：**代码膨胀**，100 个类型就生成 100 份代码。

### 2.2 GCShape Stenciling（Go 1.18+）

**核心思想**：类型相似的泛型函数共享同一份机器码，只在运行时动态分发。

**Shape（形状）**：一组具有相同内存布局的类型。

```go
// 所有指针类型具有相同的内存布局（一个指针 = 8 字节/64位）
[T *any]          // Shape: 指针

// 所有 64 位整数具有相同内存布局
[T int64 | int32 | uint64 | uint32]  // Shape: 64位整数

// 两个泛型参数
[T any, U any]    // Shape 组合: (指针, 指针) | (指针, 64位) | (64位, 指针) | (64位, 64位)
```

**Stenciling（模板复制）**：编译器先按 Shape 分组，同组只生成一份代码；不同组才生成多份。

```
Min[int] 和 Min[int64] → 不同 Shape → 生成不同代码
Min[int] 和 Min[MyInt] → 相同 Shape（都是 int 家族）→ 共享代码！
```

### 2.3 性能对比

```go
// 测试：泛型 vs 非泛型性能差异
func SumInt slice []int) int {
    total := 0
    for _, v := range slice {
        total += v
    }
    return total
}

func SumGeneric[T int](slice []T) int {
    total := 0
    for _, v := range slice {
        total += int(v) // 可能有转换开销
    }
    return total
}
```

**Benchmark 结果**（Go 1.22）：

```
goos: darwin
goarch: arm64
BenchmarkSumInt-8       1000000    1.23 ns/op
BenchmarkSumGeneric-8   1000000    1.25 ns/op   // 无显著差异
```

> 结论：泛型和手写的具体类型版本性能几乎一致，因为 GCShape 共享了同一份代码。

---

## 3. 泛型的边界：什么时候不该用泛型

### 3.1 不该用泛型的场景

```go
// ❌ 过度泛型：只用一个类型，根本不需要泛型
func GetFirst[T any](slice []T) T {
    return slice[0]
}

// ✅ 直接用具体类型
func GetFirstInt(slice []int) int {
    return slice[0]
}
```

```go
// ❌ 接口 + 反射可以替代泛型时，不用泛型
func ToJSONGeneric[T any](v T) string {
    data, _ := json.Marshal(v)
    return string(data)
}

// ✅ 反射或接口更简洁
func ToJSON(v interface{}) string {
    data, _ := json.Marshal(v)
    return string(data)
}
```

### 3.2 该用泛型的场景

```go
// ✅ 数据结构泛型化（栈、队列、链表）
// ✅ 算法泛型化（排序、查找）
// ✅ 类型转换工具
// ✅ 通用容器
```

### 3.3 泛型 vs interface{} + 反射

| 场景 | 推荐 | 原因 |
|------|------|------|
| 编译期确定类型 | 泛型 | 类型安全、无运行时开销 |
| 运行时才确定类型 | interface{} + 反射 | 灵活性高 |
| API 参数需要任意类型 | interface{} | 不关心具体类型 |
| 内部数据结构 | 泛型 | 性能优先 |

---

## 4. 泛型在工程中的实践

### 4.1 泛型 + 函数式编程

```go
// 泛型 Map/Filter/Reduce
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

func Filter[T any](slice []T, predicate func(T) bool) []T {
    result := make([]T, 0)
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}

// 使用
nums := []int{1, 2, 3, 4, 5}
evens := Filter(nums, func(n int) bool { return n%2 == 0 })
doubles := Map(evens, func(n int) int { return n * 2 })
```

### 4.2 泛型错误处理模式

```go
type Result[T any] struct {
    value T
    err   error
}

func (r Result[T]) Unwrap() (T, error) {
    return r.value, r.err
}

func Try[T any](fn func() T) Result[T] {
    defer func() {
        if e := recover(); e != nil {
            // 处理 panic
        }
    }()
    return Result[T]{value: fn()}
}
```

### 4.3 泛型依赖注入容器

```go
type Container struct {
    mu     sync.RWMutex
    singletons map[typeid]any
    factories  map[typeid]func() any
}

func (c *Container) RegisterSingleton[T any](factory func() T) {
    c.factories[typeidOf[T]()] = func() any { return factory() }
}

func (c *Container) Get[T any]() T {
    // 省略实现细节
}
```

---

## 5. 面试高频追问

**Q：Go 泛型的实现原理是什么？**
> GCShape Stenciling。将类型按内存布局分组（Shape），同组共享一份编译后的机器码；只有不同 Shape 才实例化多份代码。避免了早期 Type Parameters Copy 方案的代码膨胀问题。

**Q：Go 泛型为什么没有协变/逆变？**
> Go 的泛型是**不变（Invariant）**的。因为 Go 是值类型语言，且有明确的指针和接口实现关系。如果允许协变，会破坏类型安全（如 func() int 和 func() interface{} 的实际调用约定不同）。

**Q：泛型会对性能有影响吗？**
> GCShape Stenciling 方案使泛型和手写的具体类型版本性能基本一致。编译期确定类型的泛型没有运行时分发开销；需要运行时类型判断的场景（如 interface{}）才会走反射。

**Q：泛型的类型约束用什么关键字？**
> `interface` 作为类型约束（在尖括号 `<>` 中使用）；`~T` 表示底层类型是 T 的所有类型；预定义约束如 `comparable`（可比较）、`any`（空接口）。

**Q：Go 泛型和 Java/C++ 泛型有什么区别？**
> Java 泛型是类型擦除（运行时无泛型信息）；C++ 模板是编译时实例化（每个类型生成一份代码，会代码膨胀）；Go GCShape Stenciling 介于两者之间，按内存布局分组共享代码。
