# Go init 函数：执行时机、顺序规则与高频面试题

> 考察频率：★★★☆☆  优先级：P0（基础必考）
> 关键词：init、包的初始化、导入顺序、副作用、main

## 面试官考察意图

考察候选人对 Go 程序初始化流程的理解。
初级只知道"init 在 main 之前执行"，高级要能讲清楚**包依赖图的拓扑排序、同一包内多文件的执行顺序、init 函数的限制（不能被调用、无参数返回值）、以及 init 在工程中的典型应用场景和坑点**。init 函数是 Go 初始化机制的核心，腾讯等公司在面试中会重点追问执行顺序细节。

---

## 核心答案（30 秒版）

Go 程序初始化遵循严格的两层顺序：

**第一层（包级别）：** 按导入顺序，深度优先遍历依赖图。先初始化被导入的包，再初始化导入者。

**第二层（包内部）：** const → var → init()。同一包内多文件按文件名升序执行，init 之间互不影响。

**最重要的一条规则：** `init` 函数**不能被显式调用**，只能由 Go 运行时自动执行；**不能有参数和返回值**；**每个文件可以有多个 init**。

---

## 深度展开

### 1. 初始化流程全景图

```
Go 程序启动顺序
═══════════════════════════════════════════════════════════

1. 初始化 runtime 包（runtime.Init）
   ↓
2. 按依赖顺序初始化所有被导入的包（深度优先 + 拓扑排序）
   每个包内部：const → var → init()
   ↓
3. 执行用户 main 包的所有 init()
   ↓
4. 执行 runtime.main() → 调用用户 main.main()
   ↓
5. 主 goroutine 开始执行

⚠️ 注意：init() 是在所有包 var 初始化之后执行的
    这意味着 init() 中可以使用已初始化的包级变量
```

### 2. 包依赖顺序：深度优先遍历

```go
// main.go
import "lib/a"  // main 依赖 a
// init order: a → main

// lib/a/a.go
import "lib/b"  // a 依赖 b
// init order: b → a → main
```

**面试追问：以下包的 init 执行顺序是什么？**

```go
// main.go
import "pkg1"
import "pkg2"

// pkg1/pkg1.go
import _ "pkg3"  // pkg1 依赖 pkg3

// pkg2/pkg2.go
import _ "pkg3"  // pkg2 也依赖 pkg3
```

**答案：** pkg3 只初始化一次（Go 运行时会去重）。完整顺序：pkg3 → pkg1 → pkg2 → main

**原理：** Go 使用**拓扑排序**处理依赖图。pkg3 被两个包同时依赖，初始化一次即可（幂等初始化）。

### 3. 同一包内多文件的执行顺序

```go
// pkg/aaa.go
package pkg
func init() { println("aaa init") }

// pkg/bbb.go
package pkg
func init() { println("bbb init") }

// pkg/zzz.go
package pkg
func init() { println("zzz init") }
```

**执行顺序：aaa → bbb → zzz（按文件名升序）**

**重要：** 同一包内的多个 init() 执行顺序**与导入顺序无关**，只与文件名有关。这是有史以来最容易被误解的 Go 规则之一——很多候选人以为先 import 的包先 init。

### 4. init 函数的核心限制（面试必问）

```go
// ❌ 编译错误：init 不能有参数
func init(a int) { }

// ❌ 编译错误：init 不能有返回值
func init() int { return 0 }

// ❌ 编译错误：init 不能被显式调用
func main() {
    init()  // 编译错误！
}

// ✅ 可以有多个 init
func init() { println("init 1") }
func init() { println("init 2") }
```

### 5. 包级别变量与 init 的执行时机

**const → var → init，三阶段顺序严格保证：**

```go
const (
    a = getVal("const")  // 第一步：const 初始化
)

var (
    b = getVal("var")    // 第二步：var 初始化
)

func init() {
    println("init")       // 第三步：init
}

func getVal(s string) string {
    println("get " + s)   // 跟踪执行顺序
    return s
}

func main() {}
```

**输出：**
```
get const
get var
init
```

### 6. init 的典型工程应用场景

#### 场景一：驱动注册模式（最常见）

```go
// database/mysql/driver.go
package mysql

import "database/sql"

func init() {
    sql.Register("mysql", &MySQLDriver{})
}

// database/pg/driver.go
package pg

import "database/sql"

func init() {
    sql.Register("postgres", &PGDriver{})
}
```

主程序无需显式注册，import 触发了 init 自动注册。

#### 场景二：配置加载

```go
var config *Config

func init() {
    cfg, err := loadConfig("config.yaml")
    if err != nil {
        panic(err)
    }
    config = cfg
}
```

#### 场景三：标准库 init 的实际案例

```go
// encoding/json/stream.go
func init() {
    // 注册 marshal/unmarshal 行为
}

// net/http/client.go
func init() {
    // 设置默认 HTTP 客户端
}
```

### 7. init 的坑点与注意事项

#### 坑点一：init 初始化顺序不受 import 位置影响

```go
// main.go
import "lib/b"  // b 在文件顶部 import
import "lib/a"  // a 在文件底部 import

// 执行顺序：仍然按文件名或编译器发现的顺序，而非 import 顺序！
```

**结论：永远不要依赖 import 顺序来控制 init 执行顺序。**

#### 坑点二：init 失败导致整个程序无法启动

```go
func init() {
    if err := connectDB(); err != nil {
        panic("database connection failed: " + err.Error())
    }
}
```

**建议：** init 中的初始化失败应该用 panic，因为没有任何调用者可以处理这个错误。这是 Go 的设计选择。

#### 坑点三：init 在包级别 var 之后执行，但 var 初始化可能依赖其他包的 init

```go
// pkg/config.go
var Conf = loadConfig()  // var 初始化调用 loadConfig

// loadConfig 中可能依赖其他已初始化的包
```

这是合法的，因为包的 var 初始化发生在该包被导入时，此时依赖包已初始化完毕。

#### 坑点四：_ import 会触发 init 但不导入包名

```go
import _ "database/sql/driver"  // 只执行 init()，不导入任何符号
```

这正是驱动注册模式的核心：import the package for its side effect (init)。

### 8. main 包与 init 的关系

```go
// main.go
var mainVar = getVal("main var")

func init() {
    println("main init")
}

func main() {
    println("main main")
}

func getVal(s string) string {
    println("get " + s)
    return s
}
```

**输出：**
```
get main var
main init
main main
```

---

## 面试真题实战

### 题目：执行顺序是什么？

```go
// 文件：main.go
package main

import "fmt"

var _ = initVar()

func init() {
    fmt.Println("1. main init")
}

func initVar() int {
    fmt.Println("2. main var")
    return 0
}

func main() {
    fmt.Println("3. main main")
}
```

### 答案

```
2. main var
1. main init
3. main main
```

**解析：** 包级别变量（initVar）在 init 之前初始化。`var _ = initVar()` 是变量初始化语句，不是 init 函数，所以先执行。

---

## 常见面试追问

**追问 1：为什么 Go 选择 init 而不是全局构造函数？**
答：Go 没有类的概念，构造函数不适用。init 的设计简洁：每个文件可定义多个 init，自动按依赖顺序执行，适合包级别的初始化和副作用注册。

**追问 2：init 和 main 的区别是什么？**
答：
| 特性 | init | main |
|------|------|------|
| 作用域 | 每个包可多个 | 每个程序一个 |
| 调用者 | Go 运行时自动调用 | OS 直接调用 |
| 参数/返回值 | 不能有 | 必须 func main() |
| 执行时机 | main 之前 | 程序入口 |

**追问 3：如何测试依赖 init 初始化的代码？**
答：用 mock 替换依赖，或使用依赖注入。不要在 init 中做无法替换的全局状态。更好的做法是用 NewXxx() 工厂函数代替 init 做初始化，便于测试。

**追问 4：init 能用于单例模式吗？**
答：不推荐。init 只执行一次，但无法控制初始化的具体时机，且无法传递参数。Go 中单例应使用 sync.Once 或包级别 var + init 结合。

---

## Go 版本演进

- **Go 1.0+**：init 行为稳定，无重大变更
- **Go 1.14+**：import _ 的行为没有变化
- **Go 1.24+**：init 行为稳定不变

---

## 总结

**记忆口诀：**
- `const` 先，var 后，init 最后
- import 深度优先，同包按文件名
- init 不能调，不能有参数返回值
- 幂等可重入（同一包只 init 一次）

**工程建议：**
- 优先用包级别 var 初始化，少用 init
- init 用于驱动注册等纯副作用场景
- 测试场景用依赖注入替代 init 初始化
