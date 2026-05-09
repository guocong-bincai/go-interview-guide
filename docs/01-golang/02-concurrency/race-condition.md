[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 并发编程](../README.md)

---

# Data Race 与竞争条件：原理、检测与生产实战

## 面试官考察意图

区分"数据竞争（data race）"和"竞争条件（race condition）"这两个概念是高级工程师的基本功。
初级候选人往往混为一谈，高级要能讲清楚：**data race 是内存层面的并发访问问题，race condition 是业务逻辑层面的执行顺序问题**，两者既有联系又有本质区别。更重要的是，高级工程师要能用 `-race` 工具检测 data race，理解其检测原理，并说出 map 并发读写为什么直接崩溃。

---

## 核心答案（30 秒版）

**Data Race（数据竞争）** ≠ **Race Condition（竞争条件）**，但两者经常同时出现。

| 概念 | 本质 | 检测方式 |
|------|------|----------|
| **Data Race** | 两个 goroutine 并发访问同一内存，至少一个写操作 | `go test -race` / `go build -race` |
| **Race Condition** | 程序结果依赖执行时序，不一定是并发问题 | 代码审查、场景分析 |

Data Race 的判定需同时满足三个条件（ Happens-Before 模型）：
1. 两个 goroutine 并发访问同一变量
2. 至少一个访问是写操作
3. 没有 Happens-Before 关系（无同步原语保护）

**Map 并发读写直接崩溃**：`go run -race` 下 map 并发写会触发 `fatal error: concurrent map writes`，因为 map 内部用 hash 桶，并发写入会破坏桶链表结构。

---

## 深度展开

### 1. Data Race 的精确定义：Happens-Before 模型

Go 的内存模型定义了 **Happens-Before 关系**。当两个操作之间不存在 Happens-Before 关系，且它们并发访问同一内存且至少一个是写操作时，就构成了 data race。

```go
// 例1：明确的 data race
var x int
go func() { x = 1 }()      // 写
go func() { print(x) }()   // 读（无同步）

// 例2：被 channel 保护，无 data race
var x int
var mu sync.Mutex
go func() {
    mu.Lock()
    x = 1
    mu.Unlock()
}()
go func() {
    mu.Lock()
    print(x)
    mu.Unlock()
}()

// 例3：仅发送/接收操作之间有 Happens-Before
ch := make(chan int, 1)
go func() { ch <- 1 }()
go func() { <-ch }()  // 第二个 receive 在第一个 send 之后，无 data race
```

**Go 中建立 Happens-Before 关系的操作：**

| 同步原语 | 建立的 Happens-Before |
|----------|----------------------|
| `chan send` | 在对应 `chan receive` **之前** |
| `chan close` | 在从已关闭 channel 接收（返回零值）**之前** |
| `Lock()` | 在后续 `Unlock()` **之前** |
| `sync.Once.Do()` | 在任意后续调用**之前** |
| `atomic` 操作 | 提供宽松的 Happens-Before |

### 2. Race Condition：业务逻辑层面的执行顺序问题

Race Condition 不一定有 data race — 它描述的是"程序行为依赖执行顺序"。

```go
// Race Condition（业务层面）：转账场景
// 账户余额 100，两人同时转出 50
// 如果执行顺序不确定，最终可能是 0 或 50，而不是预期的 0
// 但这是业务逻辑问题，不是内存层面的 data race（可以用锁保护）
var balance int = 100

func withdraw(amount int) {
    // 正确：用 atomic 或 mutex 保护
    atomic.AddInt32(&balance, -int32(amount))
}
```

**关键区别：**
- Data race → **编译器/运行时**可以检测（`-race` 工具）
- Race condition → **逻辑错误**，只能靠代码审查和测试覆盖

### 3. 常见 Data Race 场景

#### 3.1 Map 并发读写（最常见）

```go
// ❌ 运行时直接崩溃：fatal error: concurrent map writes
m := make(map[string]int)
var wg sync.WaitGroup
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        m["key"] = i
    }(i)
}
wg.Wait()
_ = m["key"]

// ✅ 修复1：sync.Mutex
var mu sync.Mutex
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        mu.Lock()
        m["key"] = i
        mu.Unlock()
    }(i)
}

// ✅ 修复2：sync.Map（适合读多写少）
var sm sync.Map
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        sm.Store("key", i)
    }(i)
}
val, _ := sm.Load("key")

// ✅ 修复3：RWMutex（读多写少）
var rwmu sync.RWMutex
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        rwmu.Lock()
        m["key"] = i
        rwmu.Unlock()
    }(i)
}
```

#### 3.2 Slice 并发append

```go
// ❌ data race：slice 内部 array 指针被并发修改
var s []int
var wg sync.WaitGroup
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        s = append(s, i)  // 底层 array 可能扩容，导致 data race
    }(i)
}
wg.Wait()

// ✅ 修复：RWMutex + copy-on-write
var mu sync.RWMutex
var snapshot []int
for i := 0; i < 100; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        mu.Lock()
        s = append(s, i)
        snapshot = make([]int, len(s))
        copy(snapshot, s)
        mu.Unlock()
    }(i)
}
```

#### 3.3 全局变量 + 多个 goroutine

```go
// ❌ data race
var counter int
var done bool

func process() {
    counter++
    done = true  // 写 done 和读 done 之间可能被其他 goroutine 插入
}

func check() {
    if done {  // 可能读到 stale value
        print(counter)  // data race: 读 counter 和写 counter
    }
}
```

#### 3.4 循环变量捕获（Go 1.22 前的经典坑）

```go
// Go 1.22 之前：闭包捕获的是变量地址，不是值
for _, v := range []int{1, 2, 3} {
    go func() {
        print(v) // 全部打印 3（或 1/2/3 不确定）
    }()
}

// ✅ Go 1.22+：循环变量每次迭代是独立的
for _, v := range []int{1, 2, 3} {
    go func(val int) {
        print(val) // 正确打印 1, 2, 3
    }(v)
}
```

### 4. `-race` 检测器：原理与使用

#### 4.1 原理：编译时插桩 + 线程安全检测

`-race` 通过编译器插桩，在每次内存访问时添加逻辑：

```go
// 原始代码
x = 1

// 插桩后（伪代码）
RaceWrite(&x, 1)   // 记录这次写访问
atomic.StoreInt32(&x, 1)
```

插桩后的代码维护一个**影子内存（shadow memory）**，记录每个内存位置最近被哪个 goroutine 访问、访问类型（读/写）。当两个 goroutine 同时访问同一内存且至少一个写操作，且无 Happens-Before 关系时，报告 data race。

**`-race` 的开销：**
- 内存：2~5 倍额外内存
- CPU：运行速度降低 5~10 倍
- **不能用于生产**，仅用于开发和测试

#### 4.2 实际使用

```bash
# 单元测试加 -race
go test -race ./...

# 构建加 -race
go build -race ./cmd/myapp

# 运行加 -race（直接跑二进制）
./myapp（已用 -race 构建）

# 查看 race report 示例
# ==========================
# WARNING: DATA RACE
# Write at 0x00c0000a8008 by goroutine 8:
#   main.func·002()
#       /path/to/file.go:15 +0x44
#
# Previous read at 0x00c0000a8008 by goroutine 7:
#   main.func·001()
#       /path/to/file.go:14 +0x3c
#
# Goroutine 8 (running) created at:
#   ...
# ==========================
```

#### 4.3 `go race` 命令（Go 1.21+）

```bash
# Go 1.21+ 内置 race detector 命令
go race ./...

# 查看详细报告
go test -race -trace race.out .
```

#### 4.4 生产中的替代方案

`-race` 损耗太大无法上线，以下是生产级替代：

```go
// 方案1：go tooling -race 在 CI 中运行
// .github/workflows/test.yml
// - name: Race Test
//   run: go test -race ./...

// 方案2：定期在 staging 环境运行 race 检测
// 方案3：用 atomic 替代锁（无锁方案）
var counter atomic.Int64
counter.Add(1)

// 方案4：启用 runtime/debug.RaceMutexWarnings()（Go 1.22+）
runtime debug.RaceMutexWarnings(true)
```

### 5. Data Race vs 竞态条件：深层联系

Data race 是竞态条件的一种**实现形式**，但竞态条件的范围更广：

```
Race Condition（竞争条件）
    │
    ├── Data Race（内存层面）← 编译器可检测
    │       多个并发写者无同步访问同一内存
    │
    └── Logical Race Condition（逻辑层面）← 编译器无法检测
            业务状态依赖执行时序
            例：双重检查锁定（double-checked locking）
                if obj == nil {      // 读1（无锁）
                    mu.Lock()
                    if obj == nil {  // 读2（有锁）
                        obj = new()
                    }
                    mu.Unlock()
                }
            // 如果不用第二层检查，两个 goroutine 可能同时创建对象
```

---

## 高频追问

**Q：Map 并发读写为什么会崩溃，而不是产生不确定值？**

Go 的 map 实现包含指针链表结构的 hash 桶，并发写入可能破坏桶指针，导致：
- 指针链断裂 → 某个 key 永久丢失
- 指针指向非法地址 → 触发 panic（Go 选择直接崩溃，而不是静默错误）

这是 Go 设计哲学：**fail fast**，快速暴露并发错误，避免隐蔽的内存损坏。

**Q：`atomic.Value` 怎么用？它比 mutex 有什么优势？**

```go
type Config struct {
    Host string
    Port int
}

var config atomic.Value
config.Store(&Config{Host: "localhost", Port: 8080})

// 读取（无锁）
cfg := config.Load().(*Config)

// 注意：存入的值必须是指针或不可变结构体
// 否则其他 goroutine 可能同时修改同一对象
```

`atomic.Value` 优势：**读操作完全无锁**，比 `sync.Mutex` 的 `RLock()` 更快。适合读多写少、值整体替换的场景。

**Q：`go test -race` 通过了，线上还会有 data race 吗？**

有可能。`-race` 是**概率性检测**，依赖于测试覆盖率和并发度。真实线上可能出现：
- 测试中未覆盖的 goroutine 组合
- 低概率时序问题（stress test 测不出来）
- Race condition 导致的逻辑问题（`atomic.Value` 存入了被并发修改的对象）

建议：CI 中始终加 `-race`，有条件的话 staging 环境也开启。

**Q：什么是 TSAN（ThreadSanitizer）？Go 的 `-race` 和它有什么关系？**

TSAN 是 LLVM/Google 开发的线程安全检测工具，Go 的 `-race` 底层就是基于 TSAN 的 shadow memory 机制。GCC/Clang 的 `-fsanitize=thread` 与 Go `-race` 检测原理相同，都是在编译器插桩层面检测并发访问。

**Q：为什么 `-race` 会让程序慢 5~10 倍？**

每次内存访问都要记录访问者 goroutine ID、地址、访问类型到影子内存，并维护全局 goroutine 栈。这些操作本身就需要原子指令和锁，而且大量的内存访问会产生巨大的数据量（约 2~5 倍正常内存占用）。

**Q：如何在生产环境检测 data race？**

生产环境不能开 `-race`（开销太大）。替代方案：
1. **CI 强制** `go test -race`：每次 PR 必须通过
2. **定期 staging 测试**：在 staging 环境用 `-race` 跑流量
3. **静态分析**：用 `go vet -race` 做初步检查
4. **Prometheus 监控 goroutine 数量**：异常增长可能是泄漏/死锁的信号

---

## 延伸阅读

- [Go Memory Model - The Go Programming Language](https://go.dev/ref/mem)（官方内存模型文档）
- [Data Race Detector - The Go Programming Language](https://go.dev/doc/articles/race_detector)（官方 race 检测器文档）
- [The Go Race Detector - ACM Queue](https://queue.acm.org/detail.cfm?id=3534855)（TSAN 原理）
- [Go 1.21 sync/atomic Value 新增方法](https://go.dev/blog/atomic-values)
- [Race condition vs Data race - Wikipedia](https://en.wikipedia.org/wiki/Race_condition#Software)

---

**[← 上一篇：context 原理](./05-context.md)** · **[下一篇：并发模式 →](./04-patterns.md)**
