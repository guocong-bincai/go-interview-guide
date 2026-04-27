[🏠 首页](../../../README.md) · [📦 Go 语言深度](../README.md) · [🔧 语言深层特性](../README.md)

---

# encoding/json v2：新一代 JSON 处理 API

> 考察频率：★★★☆☆  优先级：P1  
> 关键词：jsontext、v2、无反射、严格 UTF-8、流式解析、零拷贝

---

## 面试官考察意图

考察候选人对 Go 标准库演进的理解深度，以及是否有跟进 Go 1.25 实验性特性的习惯。
初级只知道 `encoding/json` 的 Marshal/Unmarshal，高级要能说清楚 **v1 的历史局限、v2 的设计取舍、以及什么场景下该用哪个版本**。
这是一个典型的"技术判断力"问题——不是非此即彼，而是理解 trade-off。

---

## 核心答案（30 秒版）

Go 1.25 引入了 `encoding/json` 的全新实验性 API，分成两层：

| 包 | 职责 | 核心改进 |
|---|---|---|
| `encoding/json/jsontext` | 底层语法处理（无反射） | 零拷贝、流式、流式 token 解析 |
| `encoding/json/v2` | 上层语义映射（替代 v1） | 严格 UTF-8、拒绝重复 key、nil slice→[]、更安全 |

**v1 的痛点：** 依赖反射、性能差、接受非法 UTF-8 和重复 key、无法流式处理。
**v2 的取舍：** API 不兼容，但性能更好、安全性更高。

---

## 深度展开

### 1. v1 有什么问题

#### 1.1 行为缺陷（安全性隐患）

**问题 1：接受非法 UTF-8**

```go
// v1：不报错，默默接受非法 UTF-8，可能导致下游数据损坏
data := []byte(`{"name":"\xc0\xc0"}`) // 非法的 UTF-8 序列
var v map[string]string
json.Unmarshal(data, &v) // ❌ 没有报错！
fmt.Println(v["name"])   // 输出乱码或替换字符
```

v2 默认要求严格 UTF-8，不合法的输入直接报错。

**问题 2：接受重复 key**

```go
// v1：接受重复 key，只保留最后一个（行为未定义）
data := []byte(`{"a":1,"a":2}`)
var v map[string]int
json.Unmarshal(data, &v)
fmt.Println(v["a"]) // 输出 2，但不同实现可能输出 1
```

这在安全敏感场景（如签名验签）下可能被利用（参考 CVE-2017-12635）。

**问题 3：nil slice 序列化为 null**

```go
// v1：nil slice → null，可能导致跨语言接口不兼容
var s []string = nil
b, _ := json.Marshal(s)
fmt.Println(string(b)) // null

// v2：nil slice → []（空数组），更符合直觉
```

#### 1.2 API 缺陷

**问题 4：无法流式解析**

v1 的 `Decoder.Token` 需要一次性加载整个 JSON 值才能解析，生产中容易写错：

```go
// ❌ 错误写法：不会拒绝尾部垃圾数据
json.NewDecoder(r).Decode(&v)

// ✅ 正确写法（但很容易被忘掉）
dec := json.NewDecoder(r)
dec.Decode(&v)
if dec.More() {
    return errors.New("trailing junk")
}
```

**问题 5：MarshalJSON/UnmarshalJSON 性能陷阱**

```go
// 如果 MarshalJSON 递归调用 json.Marshal，性能可能是 O(n²)
// 因为 v1 需要：① 验证返回值是合法 JSON，② 重新格式化
type Node struct {
    Value string
    Next  *Node
}

func (n *Node) MarshalJSON() ([]byte, error) {
    // 这里递归调用 json.Marshal，如果处理大链表，层层验证开销巨大
    return json.Marshal(map[string]any{"v": n.Value, "next": n.Next})
}
```

### 2. jsontext 包：纯语法层

`encoding/json/jsontext` 只处理 JSON 语法，不涉及 Go 类型映射，**不依赖反射**。

```go
import "encoding/json/jsontext"

// Value：[]byte 的别名，代表一个完整的 JSON 值（零拷贝）
type Value []byte

// Encoder：流式写 JSON
func ExampleEncoder() {
    var buf bytes.Buffer
    enc := jsontext.NewEncoder(&buf)
    
    // WriteValue：直接写入 []byte，不做类型转换
    err := enc.WriteValue([]byte(`{"name":"Alice","age":30}`))
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(buf.String())
}

// Decoder：流式读 JSON（不需要预先知道结构）
func ExampleDecoder() {
    r := strings.NewReader(`{"name":"Bob"}{"name":"Carol"}`)
    dec := jsontext.NewDecoder(r)
    
    for {
        v, err := dec.ReadValue()
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Fatal(err)
        }
        // v 是 []byte，零拷贝！
        fmt.Println(string(v)) // {"name":"Bob"} {"name":"Carol"}
    }
}
```

**jsontext 的核心价值：**
- **零拷贝**：`Value` 就是 `[]byte`，不需要做中间的 []byte → string 转换
- **流式处理**：可以解析 JSON 流中多个独立值（`{}\n{}\n{}`）
- **无反射**：只管语法，不管 Go 类型

### 3. v2 包：语义映射层

`encoding/json/v2` 是 v1 的直接替代，API 设计高度相似，但行为更安全。

```go
import "encoding/json/v2"

// 签名几乎一样，但 v2 支持 Options
func Marshal(in any, opts ...Options) (out []byte, err error)
func Unmarshal(in []byte, out any, opts ...Options) error
func MarshalWrite(out io.Writer, in any, opts ...Options) error
func UnmarshalRead(in io.Reader, out any, opts ...Options) error
```

#### 3.1 基本用法对比

```go
// v1
b, _ := json.Marshal(User{Name: "Alice", Age: 30})
var user User
json.Unmarshal(b, &user)

// v2（API 几乎一样）
b, _ := v2.Marshal(User{Name: "Alice", Age: 30})
var user User
v2.Unmarshal(b, &user)
```

#### 3.2 严格模式选项

```go
// v2：可以用 Options 配置严格行为
opts := []v2.Options{
    v2.RequireUTF8(),           // 严格 UTF-8（默认开启）
    v2.RejectDuplicateNames(), // 拒绝重复 key（默认开启）
}

// 遇到非法 UTF-8 → err
// 遇到重复 key → err
err := v2.Unmarshal(data, &user, opts...)
```

#### 3.3 流式写入（MarshalWrite）

```go
// v1：必须先 Marshal 到 []byte，再 Write
b, _ := v1.Marshal(user)
w.Write(b)

// v2：直接流式写入 io.Writer，避免中间 []byte 分配
v2.MarshalWrite(w, user)
```

#### 3.4 MarshalJSON 的新接口

v2 引入 `MarshalJSONTo` 和 `UnmarshalJSONFrom`，解决递归性能问题：

```go
// v2 新接口：不返回 []byte，而是直接写入 Encoder
type MarshalerV2 interface {
    MarshalJSONTo(*Encoder, Options) error
}

// 示例：流式序列化大链表
func (n *Node) MarshalJSONTo(enc *jsontext.Encoder, _ v2.Options) error {
    enc.WriteToken(jsontext.ObjectStart)
    enc.WriteValue([]byte(`"value"`))
    enc.WriteValue([]byte(`"` + n.Value + `"`))
    if n.Next != nil {
        enc.WriteValue([]byte(`"next"`))
        n.Next.MarshalJSONTo(enc, nil)
    }
    enc.WriteToken(jsontext.ObjectEnd)
    return nil
}
```

### 4. 性能对比

> 数据来源：Go 官方 benchmark（Go 1.25）

| 操作 | v1 | v2 | 提升 |
|------|-----|-----|------|
| Marshal struct | 基准 | ~同水平 | - |
| Unmarshal struct | 基准 | 10-30%↑ | 显著 |
| 流式解析大 JSON | N/A | 零拷贝 | 巨大 |
| MarshalJSON 递归 | O(n²) | O(n) | 质变 |

### 5. 迁移策略

```go
// 渐进式迁移：用 import 别名
import (
    jsonv2 "encoding/json/v2"     // 新代码用 v2
    "encoding/json"               // 旧代码保留 v1
)

// 用 go fix 自动迁移（Go 1.26+）
// $ go fix ./...
// 会自动将 encoding/json 的调用替换为 v2
```

> 注意：v2 目前是 **experimental**，正式版预计在 Go 1.N (N≥27)。生产项目如果对 JSON 性能不敏感，可以继续用 v1。

---

## 高频追问

**Q：什么时候应该用 v2，什么时候用 v1？**

如果满足以下任一条件，优先考虑 v2：
- 处理**超大 JSON**（MB 级别），需要流式解析
- 对**安全性要求高**（拒绝非法 UTF-8、拒绝重复 key）
- 自定义 `MarshalJSON` **递归调用**（链表/树等递归结构）
- 性能敏感，v1 的反射开销成为瓶颈

如果 JSON 很小、业务简单，v1 完全够用，不必追新。

**Q：v2 和 `github.com/json-iterator/go` 哪个好？**

json-iterator 是社区高性能 JSON 库，性能比 v1 好很多。v2 的优势是**标准库**，有 Go 团队维护，API 更干净。性能上各有千秋，但 v2 的**安全性改进**是 json-iterator 没有的。

**Q：jsontext 的 Value 和 v1 的 RawMessage 有什么区别？**

`RawMessage` 是 `json.RawMessage`（= `[]byte`），只是延迟解析的容器。
`jsontext.Value` 是专门设计的零拷贝 JSON 值类型，配有 `Kind()` 方法判断类型（`null`/`bool`/`string`/`number`/`array`/`object`），以及 `Append()` 等工具方法，更适合做底层解析。

**Q：v2 会破坏 Go 1 兼容性承诺吗？**

不会。v2 在独立的 `encoding/json/v2` 路径下，不影响原有 `encoding/json`（v1）。Go 1 兼容性承诺只保证 `encoding/json` 不变，v2 是额外的新包。

**Q：Go 1.25 中 v2 是 experimental，如何启用？**

```go
// 在 go.mod 中添加
require encoding/json/v2 v0.0.0-experimental

// 或者使用 GOEXPERIMENT
// $ GOEXPERIMENT=jsonv2 go build
```

---

## 延伸阅读

- [Go 官方博客：A new experimental Go API for JSON](https://go.dev/blog/jsonv2-exp)（2025-09-09）
- [encoding/json/jsontext 文档](https://pkg.go.dev/encoding/json/jsontext)
- [encoding/json/v2 文档](https://pkg.go.dev/encoding/json/v2)
- [Go JSON v2 实验提案](https://go.dev/issue/71845)
- [Anton's Blog: Go JSON v2 实战](https://antonz.org/go-json-v2/)

---

**[← 上一篇：Go 1.27 标准库新特性](./06-go1.27-stdlib.md)**
