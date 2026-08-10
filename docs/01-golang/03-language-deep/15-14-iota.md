# Go iota 枚举：计数器原理、进阶用法与高频面试题

> 考察频率：★★★★☆  优先级：P0（基础必考）
> 关键词：iota、枚举、const、计数器、位掩码、网球问题

## 面试官考察意图

考察候选人对 Go 语言常量计数器 iota 的理解深度。
初级只能说出"iota 是从 0 开始的计数器"，高级要能讲清楚**iota 在每个 const 块重置的行为、跳跃与省略规则、连续与非连续使用场景**，以及在**位掩码、枚举值、协议字段**中的实际工程应用。iot a 看似简单，却是 Go 基础语法中出现频率最高、细节最多的知识点之一。

---

## 核心答案（30 秒版）

`iota` 是 Go 语言在 `const` 声明块内的**连续整数计数器**：

| 特性 | 说明 |
|------|------|
| 重置规则 | 每个 `const` 关键字使 iota 重置为 0 |
| 递增规则 | 每一行（包括空行）使 iota 递增 1 |
| 跳跃规则 | 用 `_` 跳过某值，下一行 iota 继续递增 |
| 省略规则 | 用空标识符 `_` 或省略表达式继承上一行 |
| 表达式 | iota 可参与表达式计算，不只是简单的整数 |

**最核心的面试坑点：** 很多人以为 `iota` 是"遇到的第几个非空常量行"，但实际上是"遇到的第几个常量行（含空行）"——这是理解一切 iota 行为的关键。

---

## 深度展开

### 1. iota 基础行为：为什么是"第几行"而非"第几个常量"

```go
const (
    A = iota  // iota = 0
    B         // iota = 1，简写继承 A
    C         // iota = 2，简写继承 B
)
```

iota 的值 = **当前行在 const 块中的从 0 开始行号**（不含 const 关键字那行）。

**最常见的面试坑：**

```go
const (
    A = iota  // 0
    _         // 1（跳过）
    B         // 2，而不是 1！
)
```

很多候选人以为 B = 1，因为"A 占了 0，_ 跳过所以 B 是 1"。实际上 _ 是一行，所以 B 是第 3 行（iota = 2）。

### 2. 继承与跳跃：三种模式的对比

#### 模式一：显式继承（最常用）

```go
const (
    Monday    = iota  // 0
    Tuesday            // 1
    Wednesday          // 2
    Thursday           // 3
    Friday            // 4
    Saturday          // 5
    Sunday            // 6
)
```

#### 模式二：跳跃（用 _ 占位跳过）

```go
const (
    _          = iota  // 0（跳过）
    Monday              // 1
    Tuesday             // 2
    Wednesday           // 3
    Thursday            // 4
    Friday              // 5
    Saturday            // 6
    Sunday              // 7
)
```

#### 模式三：位掩码（工程高频场景）

```go
const (
    ReadPerm  = 1 << iota  // 1 << 0 = 1
    WritePerm               // 1 << 1 = 2
    ExecutePerm             // 1 << 2 = 4
)

const (
    FlagLocked   = 1 << iota  // 1 << 0 = 1
    FlagForward  = 1 << iota  // 1 << 1 = 2
    FlagRoot     = 1 << iota  // 1 << 2 = 4
    FlagSticky   = 1 << iota  // 1 << 3 = 8
)
```

位掩码模式是**权限控制、协议字段、标志位**场景下的标准写法，支持位运算组合：

```go
perm := ReadPerm | WritePerm  // 1 | 2 = 3
hasRead := perm & ReadPerm != 0  // true
hasExec := perm & ExecutePerm != 0  // false
```

### 3. iota 在同一行中的多次使用

iota 在同一行只计一次值：

```go
const (
    BitZero, MaskZero = iota, iota << 8  // iota = 0 → 0, 0
    BitOne,  MaskOne  = iota, iota << 8  // iota = 1 → 1, 256
    BitTwo,  MaskTwo  = iota, iota << 8  // iota = 2 → 2, 512
)
```

### 4. "网球问题"：iota 不连续递增的经典陷阱

**面试高频题：以下代码输出什么？**

```go
const (
    a = iota  // 0
    b = 5     // 直接赋值，iota 不变，仍是 1
    c = iota  // 2！不是 1
    d         // 3，简写继承 c
)
```

```go
// 答案：a=0, b=5, c=2, d=3
```

**原理：** b 直接赋值为 5，没有用 iota，但 iota 在遇到非 iota 表达式时**仍然递增**。这是最容易出错的点。

### 5. 多 const 块的行为：每个 const 关键字都重置 iota

```go
const a = iota  // a = 0（新块，重置）
const b = iota  // b = 0（新块，重置）

const (
    c = iota  // c = 0（新块开始）
    d         // d = 1
)

const (
    e = iota  // e = 0（新块开始）
    f         // f = 1
)
```

**总结：** `const` 关键字 = iota 重置信号。同一 `const` 块内连续递增。

### 6. 工程实践：iota + stringer 的枚举模式

```go
//go:generate stringer -type=Status -linecomment=true
const (
    StatusPending   Status = iota
    StatusRunning
    StatusCompleted
    StatusFailed
)
```

运行时转换：

```go
func (s Status) String() string {
    return [...]string{
        StatusPending:   "pending",
        StatusRunning:   "running",
        StatusCompleted: "completed",
        StatusFailed:    "failed",
    }[s]
}
```

**进阶：iota + iota 的非线性用法**

```go
// 协议字段：每个字段占不同 bit 位置
const (
    ProtocolICMP ICMPType = iota
    ProtocolTCP  ICMPType = 6  // 直接指定，iota 继续递增
    ProtocolUDP  ICMPType = 17
)
```

### 7. 常见面试追问

**追问 1：iota 能用在函数内部吗？**
答：不能。iota 是 const 声明块的专有语法，只能在 const 作用域内使用。

**追问 2：iota 和 enum 的区别是什么？**
答：Go 没有原生 enum，iota + const 是 Go 实现枚举的标准方式。与 Java/Python 的 enum 不同，Go 的 iota 枚举值只是整数常量，没有类型安全性（可以用自定义类型改善）。

**追问 3：为什么 Go 不提供原生 enum？**
答：Go 设计哲学追求简洁。iota + const 已经能覆盖枚举场景，原生 enum 需要额外的类型系统开销。Go 1.18+ 的泛型可以部分弥补枚举的类型安全问题。

**追问 4：如何用 iota 实现类型安全的枚举？**
答：

```go
type Weapon int

const (
    Sword Weapon = iota
    Arrow
    Cannon
)

func (w Weapon) IsRanged() bool {
    return w >= Cannon  // 利用iota值的有序性
}
```

---

## 面试真题实战

### 题目：以下代码输出什么？

```go
const (
    x = iota           // ?
    y                  // ?
    z = 9              // ?
    w                  // ?
    v = iota           // ?
)
```

### 答案

```go
// x = 0, y = 1, z = 9, w = 9, v = 4
```

**解析：**
- `x = iota`：第 1 行，iota=0
- `y`：第 2 行，iota=1，继承 x
- `z = 9`：第 3 行，iota=2，直接赋 9，iota 继续递增
- `w`：第 4 行，iota=3，继承 z=9
- `v = iota`：第 5 行，iota=4

---

## Go 版本演进

- **Go 1.0+**：iota 行为稳定，无变更
- **Go 1.18+**：iota 不受泛型影响，行为不变
- **Go 1.24+**：iota 与泛型类型别名无关，行为不变

---

## 总结

| 场景 | 推荐写法 |
|------|---------|
| 连续枚举 | `A = iota` 逐行继承 |
| 跳跃枚举 | `_ = iota` 占位 |
| 权限位掩码 | `1 << iota` |
| 非连续枚举 | 直接赋值，iota 仍递增 |
| 类型安全枚举 | 自定义类型 + iota |

**核心记忆点：** iota = 行号计数器，每个 const 块重置。遇到非 iota 表达式，iota 仍递增但不使用。
