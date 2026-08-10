# nil slice vs empty slice：面试必考区分题

> 考察频率：★★★★☆  优先级：P1  
> 关键词：nil slice、empty slice、JSON null vs []、make vs nil、append 行为、zero value

---

## 面试官考察意图

考察候选人对 Go 基础类型零值的理解深度。
初级工程师常说 "nil slice == empty slice"，高级工程师能清晰指出它们在某些场景等价、在另一些场景完全不同的本质区别。这道题看似简单，但能通过 JSON 序列化、append 行为、内存分配等多个角度持续追问，非常适合用来筛选候选人的实际编码经验。

---

## 核心答案（30秒版）

| 特性 | nil slice | empty slice (`[]T{}` 或 `make([]T, 0)`) |
|------|-----------|----------------------------------------|
| `len()` | `0` | `0` ✅ 相同 |
| `cap()` | `0` | `0` ✅ 相同 |
| `for range` | 不进入循环 | 不进入循环 ✅ 相同 |
| `len(s) == 0` | `true` ✅ 相同 | `true` ✅ 相同 |
| `s == nil` | `true` | `false` ❌ **不同** |
| `json.Marshal` | `"null"` | `"[]"` ❌ **不同** |
| append 后底层数组 | 从零分配 | 复用已有零长度数组头 |
| 零值 | `nil`（不需要 make） | 需要 `make` 或 `[]T{}` |

**关键结论：判断空用 `len(s) == 0`，判断 nil 用 `s == nil`。两者在大多数业务逻辑中等价，但在 JSON 序列化时语义完全不同。**

---

## 深度展开

### 1. 内部结构对比

```go
// nil slice：ptr、len、cap 都是 0
var nilSlice []int  // ptr=nil, len=0, cap=0
fmt.Println(len(nilSlice), cap(nilSlice)) // 0 0

// empty slice：ptr 指向 runtime.zerobase，len=0, cap=0
emptySlice := []int{}           // ptr=&runtime.zerobase, len=0, cap=0
emptySlice2 := make([]int, 0)   // 同上
fmt.Println(len(emptySlice), cap(emptySlice)) // 0 0
```

**内存层面：**

| Slice 类型 | ptr 指向 | len | cap | 内存占用 |
|-----------|---------|-----|-----|---------|
| nil slice | `nil` | 0 | 0 | 24 bytes（slice header），ptr 字段为 nil |
| empty slice | `&runtime.zerobase` | 0 | 0 | 24 bytes，ptr 指向零字节位置 |

### 2. 什么时候完全等价？

#### 2.1 长度检查 ✅

```go
func isEmpty[T any](s []T) bool { return len(s) == 0 }

var nilSlice []int
emptySlice := make([]int, 0)

isEmpty(nilSlice)    // true
isEmpty(emptySlice)  // true
```

#### 2.2 range 遍历 ✅

```go
for i, v := range nilSlice {
    fmt.Println(i, v) // 不执行
}
for i, v := range emptySlice {
    fmt.Println(i, v) // 不执行
}
```

#### 2.3 append ✅（这是最容易踩坑的地方！）

```go
var nilSlice []string
result := append(nilSlice, "hello")
fmt.Println(result)       // [hello]
fmt.Println(len(result))  // 1
fmt.Println(cap(result))  // 4
fmt.Println(result[0])    // hello —— 不会 panic！

// append 的语义：如果底层数组为空（nil 或零长度），会自动分配新内存
```

**为什么 append 能正确处理 nil slice？**

因为 `append` 的底层实现会先检查 `len < cap`，如果 len == cap（包括两者都为 0），就调用运行时分配器扩容，而不是依赖现有指针。所以 nil slice 的 append 完全安全。

#### 2.4 copy ✅

```go
src := []int{1, 2, 3}
dst := make([]int, 3)
copy(dst, src)        // dst = [1, 2, 3]

dst2 := make([]int, 0)
n := copy(dst2, src)  // n = 0（dst2 长度为 0），不会 panic

// copy 只拷贝 min(len(dst), len(src)) 个元素
```

### 3. 什么时候不一样？❌

#### 3.1 JSON 序列化（最核心的面试考点！）

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Response struct {
    Tags []string `json:"tags"`
}

func main() {
    r1 := Response{Tags: nil}      // nil slice
    r2 := Response{Tags: []string{}} // empty slice

    b1, _ := json.Marshal(r1)
    b2, _ := json.Marshal(r2)

    fmt.Println(string(b1)) // {"tags":null}     ← null!
    fmt.Println(string(b2)) // {"tags":[]}       ← 空数组!
}
```

**API 对接中的致命差异：**

```go
// 前端 JavaScript 接收到的结果完全不同：

// null → JSON.parse 返回 null，前端必须写 tags && tags.length > 0
// [] → JSON.parse 返回空数组，前端直接写 tags.length > 0

// 错误示例：前端收到 null 然后直接 .forEach() → TypeError
tags.forEach(t => console.log(t)) // panic! null doesn't have forEach

// 解决方案：统一序列化
b, _ := json.Marshal(r1)
if string(b) == `{"tags":null}` {
    // 把 null 替换成空数组
    b = []byte(`{"tags":[]}`)
}
```

**最佳实践：前端兼容方案**

```go
// 方案 1：自定义 MarshalJSON
func (r Response) MarshalJSON() ([]byte, error) {
    type Alias Response
    if r.Tags == nil {
        r.Tags = []string{}
    }
    return json.Marshal(&struct {
        *Alias
    }{Alias: (*Alias)(&r)})
}

// 方案 2：使用omitempty标签 + 确保赋值
type Response struct {
    Tags []string `json:"tags,omitempty"`
}

// 在业务逻辑中始终初始化为空切片
func NewResponse() *Response {
    return &Response{Tags: []string{}}
}

// 方案 3：Go 1.24+ omitzero 标签（最优雅）
type Response struct {
    Tags []string `json:"tags,omitzero"` // Go 1.24+ 零值自动省略
}
```

#### 3.2 s == nil 判断 ❌

```go
var nilSlice []int
emptySlice := make([]int, 0)

fmt.Println(nilSlice == nil)       // true ✅
fmt.Println(emptySlice == nil)     // false ❌
fmt.Println(len(nilSlice) == 0)    // true ✅
fmt.Println(len(emptySlice) == 0)  // true ✅
```

**推荐写法：始终用 `len(s) == 0`，不要用 `s == nil`**

```go
// ❌ 容易漏掉 empty slice 的情况
if slice == nil {
    // empty slice 会走到这里外面
}

// ✅ 正确写法：覆盖所有情况
if len(slice) == 0 {
    // nil 和 empty 都进来
}
```

#### 3.3 反射层面的差异 ❌

```go
import "reflect"

v1 := reflect.ValueOf(nilSlice)
v2 := reflect.ValueOf(emptySlice)

fmt.Println(v1.IsNil()) // true  ✅
fmt.Println(v2.IsNil()) // false ❌

// IsZero() 返回相反结果
fmt.Println(v1.IsZero())  // true
fmt.Println(v2.IsZero())  // false
```

#### 3.4 append 前是否需要初始化？✅❌（有细微差别）

```go
var nilSlice []string

// append 可以直接用在 nil slice 上
nilSlice = append(nilSlice, "a", "b")
fmt.Println(nilSlice) // [a b]

// 但如果需要先检查再追加
if someCondition {
    nilSlice = append(nilSlice, "x")  // ✅ 安全
}

// 结论：append 可以无缝处理 nil slice，不需要预先 make
```

### 4. 生产环境实战建议

#### 4.1 函数返回值规范

```go
// ✅ 推荐：如果需要返回切片给调用方，永远返回 non-nil empty slice
func GetTags(id int) []string {
    tags, err := db.QueryTags(id)
    if err != nil {
        return []string{} // 不要返回 nil
    }
    return tags // 如果查询结果为空，也应该是 []string{} 而非 nil
}

// 调用方无需二次判 nil
tags := GetTags(1)
for _, tag := range tags { ... } // 安全！即使没数据也不会 panic
```

#### 4.2 数据库映射

```go
type User struct {
    ID    int
    Tags  sql.NullString // 数据库允许 NULL
}

// 问题：sql.NullString 可能为 NULL，映射到 slice 时需要转换
func (u User) GetTags() []string {
    if u.Tags.Valid && u.Tags.String != "" {
        strings.Split(u.Tags.String, ",")
    }
    return []string{} // 统一返回 non-nil empty slice
}
```

#### 4.3 接口设计准则

```go
// Rule 1: 输入参数可以用 nil（append 等函数天然支持）
// Rule 2: 输出参数永远保证 non-nil（让调用方少一层判断）
// Rule 3: JSON 输出根据 API 协议约定（通常要求空数组非 null）

func ProcessItems(items []Item) []Result {
    var results []Result  // 初始化即 nil
    
    for _, item := range items {
        results = append(results, transform(item))
    }
    
    return results // ⚠️ 如果 items 为空，这里返回 nil
}

// 调用方：
results := ProcessItems(items)
_ = results // 如果是 nil，后续 .Len() 方法会 panic
```

#### 4.4 性能考虑

```go
// 频繁创建空 slice 的性能测试
func BenchmarkAppendToNil(b *testing.B) {
    var s []int
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        s = append(s, 1)
    }
}

func BenchmarkAppendToEmpty(b *testing.B) {
    s := make([]int, 0)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        s = append(s, 1)
    }
}

// 实测：两者性能基本一致
// BenchmarkAppendToNil   ~10ns/op
// BenchmarkAppendToEmpty ~10ns/op
// 差异在编译器优化后可忽略
```

### 5. 常见面试陷阱题

#### 陷阱 1：切片比较

```go
func main() {
    var nilSlice []int
    emptySlice := []int{}
    
    // ❌ 编译错误！slice 不能直接用 == 比较
    // fmt.Println(nilSlice == emptySlice) 
    
    // ✅ 正确比较方式
    fmt.Println(slices.Equal(nilSlice, emptySlice)) // true
}
```

#### 陷阱 2：map 中的 slice 作为值

```go
m := map[string][]int{}

// nil slice 写入 map
m["key1"] = nil
// empty slice 写入 map
m["key2"] = []int{}

// 读取时的区别
val1 := m["key1"] // nil
val2 := m["key2"] // []int{}

// append 的行为
val1 = append(val1, 1) // val1 = [1]（重新分配了底层数组）
val2 = append(val2, 1) // val2 = [1]（原地 append，不影响 cap 但改变了内容）

// ⚠️ 写回 map 时需要重新赋值
m["key1"] = val1
m["key2"] = val2
```

#### 陷阱 3：闭包捕获

```go
func main() {
    var slices [][]int
    
    for i := 0; i < 3; i++ {
        slices = append(slices, make([]int, i))
    }
    
    for _, s := range slices {
        fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
    }
    // len=0 cap=0
    // len=1 cap=1
    // len=2 cap=2
}
```

### 6. 高频追问

**Q：nil map / nil channel 和 nil slice 有什么共同点？**

三者都是「未初始化的零值」，都有对应的 safe 操作方式：
- nil slice：append/copy/range 是安全的
- nil map：读是安全的（返回零值），写会 panic
- nil channel：读阻塞，写阻塞，close 会 panic

**Q：如何判断一个函数返回的是不是 nil slice？**

```go
func isNilSlice[T comparable](s []T) bool {
    return len(s) == 0 && reflect.ValueOf(s).IsNil()
}

// 或者用 json 技巧
func isNils[T comparable](s []T) bool {
    b, _ := json.Marshal(s)
    return string(b) == "null"
}
```

**Q：nil interface{} 和 nil slice 有什么区别？**

```go
var nilInterface interface{} = nil          // kind: None
var nilSlice []int                           // kind: Slice, 且内部 ptr=nil

fmt.Printf("%T", nilInterface)  // <nil>
fmt.Printf("%T", nilSlice)      // []int

// 关键：interface{} 本身包含 type + value 两个字段
// nil slice 是一个有效的 []int 类型，只是值为 nil
// nil interface{} 连类型都是 nil
```

**Q：为什么 Go 不统一 nil slice 和 empty slice？**

历史原因 + 语义清晰：
1. nil slice 表示「从未被初始化」
2. empty slice 表示「被初始化过，但目前为空」
3. 在 C 的遗产下，NULL 和空缓冲区是不同的概念
4. JSON 序列化恰好利用了这一区分来区分 null vs []

---

## 总结

| 维度 | nil slice | empty slice |
|------|-----------|-------------|
| 适用场景 | 临时变量、append 源、函数参数 | 函数返回值、JSON 序列化、接口契约 |
| 判断空 | `len(s) == 0` | `len(s) == 0` |
| 判断 nil | `s == nil` | `s == nil` |
| JSON 输出 | `null` | `[]` |
| 默认选择 | 声明即可，无需 make | 需要 make 或字面量 |

**黄金法则：读时用 `len(s)==0`，写时优先非 nil，JSON 场景按协议约定。**

---

## 延伸阅读

- [Effective Go - Slices](https://go.dev/doc/effective_go#slices)
- [Go JSON 序列化指南 - Matt Holt](https://dev.to/mholt/how-to-use-json-in-go-b8l)
- [Go FAQ - Comparing Values](https://go.dev/doc/faq#compare)
- [Go Spec - Comparison operators](https://go.dev/ref/spec#Comparison_operators)

---

**[← 上一篇：slice 与 map 底层结构](./05-slice-map.md)** · **[下一篇：逃逸分析 →](./04-escape.md)**
