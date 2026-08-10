# Interface 内存布局

> 考察频率：★★★★★  难度：★★★★☆

## iface vs eface

Go 的 interface 有两种底层表示：

| 类型 | 名称 | 何时使用 |
|------|------|---------|
| `iface` | 带方法的接口 | 包含至少一个方法的接口 |
| `eface` | 空接口 | `interface{}` / `any` |

### 内存布局

```go
// eface（空接口）= 2 个指针
type eface struct {
    _type *struct{}  // 类型指针
    data  unsafe.Pointer // 值指针
}

// iface（非空接口）= 2 个指针 + 方法表指针
type iface struct {
    tab  *itab  // 接口表指针
    data unsafe.Pointer // 值指针
}

// itab 结构（itab table）
type itab struct {
    inter *interfaceType  // 接口类型
    _type *_type          // 实际类型
    hash  uint32          // _type.hash，缓存
    _     [4]byte
    fun   [1]unsafe.Pointer // 方法指针（变长数组）
}
```

### 图示

```
iface（非空接口）:
┌──────────────┬──────────────┐
│    tab  ─────┼──→ itab     │
│    data ─────┼──→ 具体值   │
└──────────────┴──────────────┘

itab:
┌──────────────┬──────────────┬───────┬──────────────┐
│ interfaceType│  _type       │ hash  │ fun[0]       │
│ (接口定义)    │  (实际类型)  │       │ (方法指针)    │
└──────────────┴──────────────┴───────┴──────────────┘
```

---

## 动态分发（Dynamic Dispatch）

当调用接口方法时：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

func process(r Reader) {
    r.Read(buf) // 运行时通过 itab.fun[0] 找到实际方法
}
```

**开销**：
1. 通过 itab 找到方法指针
2. 间接调用（函数指针调用）
3. 无法内联（因为是虚函数调用）

### 性能对比

```go
// 直接调用：可内联，快
func directCall(b []byte) {
    strings.NewReader(string(b)).Read(b)
}

// 接口调用：间接调用，无法内联，慢
func interfaceCall(r io.Reader, b []byte) {
    r.Read(b) // 动态分发
}

func BenchmarkDirect(b *testing.B)   { /* strings.Reader 直接调用 */ }
func BenchmarkInterface(b *testing.B) { /* io.Reader 接口调用 */ }
```

Benchmark 结果：接口调用通常慢 10-30%。

---

## 类型断言

### 两种语法

```go
var x interface{} = "hello"

// 语法一：comma-ok 语法（安全）
if v, ok := x.(string); ok {
    fmt.Println(v)
}

// 语法二：switch 类型分支
switch v := x.(type) {
case string:
    fmt.Println("string:", v)
case int:
    fmt.Println("int:", v)
default:
    fmt.Println("unknown")
}
```

### 源码解析

```go
// 编译器生成代码（伪代码）
// 假设 iface 结构：tab, data
// itab.fun[0] 存的是类型信息

// 实际 type assertion 编译为：
itab := iface.tab
if itab._type != stringType {
    // panic: interface conversion
}
v := *(*string)(iface.data)
```

### 性能

- 类型断言本身是 O(1)，只是指针比较
- 但在热路径中仍然是开销

---

## nil 判断陷阱

### 接口包装 nil 值

```go
func returnsError() error {
    var p *MyError = nil
    return p // 返回的是 error 接口，eface = {type: *MyError, data: nil}
    // 此时 error != nil！因为 eface._type != nil
}

func main() {
    err := returnsError()
    if err != nil { // 这里为 true！
        fmt.Println("有错误")
    }
}
```

**原因**：`*MyError` 是非 nil 的类型指针，`nil` 值被包进了 eface，eface 本身不为 nil。

### 修复

```go
func returnsError() error {
    var p *MyError = nil
    if condition {
        return p
    }
    return nil
}

// 判断空接口的正确方式
if err != nil && err.Error() != "" {
    // 真的有错误
}
```

---

## 空接口（any / interface{}）

```go
var i any = 1
var i interface{} = "hello"
var i interface{} = []int{1, 2, 3} // slice 可以存
```

### 性能

空接口比具体类型接口更重：
- 存储 slice/map/chan 时，data 指针指向堆上对象
- 没有方法表，直接存储类型指针

### 存储不同类型

```go
any slice:
┌─────────────────────────────────────┐
│  _type: *sliceType                   │
│  data: ──────────────────────→ []int{1,2,3} │
└─────────────────────────────────────┘
```

---

## 接口内嵌

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// 接口组合（编译时展开）
type ReadCloser interface {
    Reader
    Closer
}
```

编译后自动生成 ReadCloser 的方法集。

---

## 常见面试题

### Q：空接口 `interface{}` 和 `any` 有区别吗？

> 没有区别，`any` 是 `interface{}` 的别名（Go 1.18+）。`any` 更语义化，推荐使用。

### Q：接口是否可以比较？

> 同一接口比较规则：
> 1. 都是 nil → 相等
> 2. 动态类型相同、值相同 → 相等
> 3. 动态类型相同、值不同 → 不等
> 4. 动态类型不同 → panic（不可比较类型）
>
> **注意**：包含 slice、map、func 的接口不可直接比较，会 panic。

### Q：如何判断接口是否实现？

```go
// 编译检查（常用）
var _ io.Reader = (*MyReader)(nil)

// 或运行时检查
func implements[T any](x any) bool {
    _, ok := x.(T)
    return ok
}
```

### Q：接口的内存占用是多少？

> 在 64 位平台：`eface` = 16 bytes（2 × 8），`iface` = 16 bytes（itab指针 + data指针）。
> 但 itab 本身有额外开销，多个接口共享 itab（每个接口类型+具体类型组合对应一个 itab）。

### Q：为什么接口类型不能直接比较 dynamic type pointer？

```go
var e1 error = errors.New("err")
var e2 error = errors.New("err")
// e1 != e2！因为 data 指针不同（两个不同字符串对象）
```
