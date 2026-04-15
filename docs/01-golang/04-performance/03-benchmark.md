[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚡ 性能调优](../README.md)

---

# 基准测试规范：benchmark 写法、避免编译器优化干扰、性能对比模板

## 面试官考察意图

考察候选人对 Go 性能测试规范的理解深度。初级只会用 `time.Since()` 手动计时，高级要能讲清楚 **`testing.B` 的正确用法、如何避免编译器把被测代码优化掉、如何使用 benchstat 做统计显著性检验、以及何时不该用 benchmark**。

---

## 核心答案（30 秒版）

Go 标准库 `testing.B` 提供可靠的性能测试：

```go
func BenchmarkJsonMarshal(b *testing.B) {
    data := buildTestData()
    b.ReportAllocs()  // 报告内存分配
    for i := 0; i < b.N; i++ {
        json.Marshal(data)
    }
}
```

**三条黄金法则**：
1. **`-benchmem`**：必须加 `-benchmem` 看内存分配，否则 benchmark 不完整
2. **`b.ReportAllocs()`**：在 benchmark 开始前调用，报告每次迭代的内存分配
3. **避免编译器优化**：用 `blackhole` 消费结果，防止 Dead Code Elimination

---

## 深度展开

### 1. 基准测试基础

```go
// 基础 benchmark
func BenchmarkFoo(b *testing.B) {
    for i := 0; i < b.N; i++ {
        foo()
    }
}

// 运行：
// go test -bench=BenchmarkFoo -benchtime=3s -benchmem
// -benchtime=3s：每个 benchmark 运行 3 秒（更精确）
// -benchmem：显示内存分配统计
```

**输出解读**：

```
BenchmarkJsonMarshal-8    500000    2840 ns/op    1024 B/op    12 allocs/op
        │                    │           │              │              │
函数名                 总迭代次数       每操作耗时    每次操作字节    每次操作分配次数
（-8=CPU核数）
```

### 2. 避免编译器优化（Dead Code Elimination）

**最常见错误**：被测代码结果被编译器发现没有副作用，直接被优化掉。

```go
// ❌ 错误：编译器发现 result 没使用，直接删掉了整个函数调用
func BenchmarkBad(b *testing.B) {
    for i := 0; i < b.N; i++ {
        result := json.Marshal(data)  // 被优化掉！
    }
}

// ✅ 正确：使用 runtime.KeepAlive 或 _ = result
func BenchmarkGood(b *testing.B) {
    var result []byte
    for i := 0; i < b.N; i++ {
        result, _ = json.Marshal(data)
    }
    _ = result  // 防止优化
}

// ✅ 正确：用 blackhole 消费结果
var sink interface{}

// ✅ 正确：用 copy 比对
func BenchmarkGood2(b *testing.B) {
    dst := make([]byte, len(data))
    for i := 0; i < b.N; i++ {
        copy(dst, data)
    }
}
```

### 3. 内存分配相关的 Benchmark

```go
// 使用 sync.Pool 优化前后的对比
func BenchmarkWithoutPool(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        buf := make([]byte, 4096)  // 每次分配
        process(buf)
    }
}

var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

func BenchmarkWithPool(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        buf := bufPool.Get().([]byte)
        process(buf)
        bufPool.Put(buf)  // 归还
    }
}
```

**输出对比**：

```
BenchmarkWithoutPool-8     100000    1200 ns/op    4096 B/op    1 allocs/op
BenchmarkWithPool-8       5000000    380 ns/op       0 B/op    0 allocs/op
                                              ↑
                              Pool 避免了每次分配
```

### 4. 统计显著性：benchstat

手动运行一次 benchmark 不够，需要统计检验：

```bash
# 安装 benchstat
go install golang.org/x/perf/cmd/benchstat@latest

# 运行 5 次，取统计结果
$ for i in {1..5}; do go test -bench=BenchmarkFoo -benchtime=3s -benchmem 2>&1 | grep BenchmarkFoo; done

BenchmarkFoo-8     500000    2840 ns/op    1024 B/op    12 allocs/op
BenchmarkFoo-8     490000    2890 ns/op    1024 B/op    12 allocs/op
BenchmarkFoo-8     510000    2820 ns/op    1024 B/op    12 allocs/op

$ benchstat /dev/stdin
name             time/op
BenchmarkFoo-8   2.85µs ± 1%

p=0.05 的置信区间：±1%，结果可信
```

### 5. CPU vs Memory vs Goroutine Profile in Benchmark

```go
// 在 benchmark 中生成 CPU profile
// 运行：go test -bench=BenchmarkFoo -cpuprof=cpu.prof
func BenchmarkFoo(b *testing.B) {
    // 开启 CPU profile
    defer profile.Start(profile.CPUProfile, profile.ProfilePath(".")).Stop()

    for i := 0; i < b.N; i++ {
        foo()
    }
}
```

### 6. Go 1.24 新增：`testing.B.Loop` — 更精确的基准测试循环

> Go 1.24 引入 `b.Loop` 是对传统 `for i := 0; i < b.N; i++` 的全面升级，解决 benchmark 中两个长期痛点：setup 重复执行、编译器优化干扰。

**传统写法的问题：**

```go
// ❌ Go 1.24 之前的传统写法
func BenchmarkOld(b *testing.B) {
    for i := 0; i < b.N; i++ {
        // 问题 1：编译器可能将循环体优化/向量化
        // 问题 2：setup 如果在循环内，需要手动 StopTimer
        // 问题 3：b.N 参与运算，编译器优化路径多
        SomeFunction()
    }
}
```

**`b.Loop` 写法：**

```go
// ✅ Go 1.24+：b.Loop 解决上述所有问题
func BenchmarkNew(b *testing.B) {
    setupOnce()              // setup 只在 benchmark 开始前执行一次
    b.ResetTimer()

    for b.Loop() {            // 每次迭代由 benchmark 框架精确控制
        SomeFunction()
    }

    // 或者用 b.Cleanup() 处理清理
    b.Cleanup(func() {
        teardownOnce()
    })
}
```

**`b.Loop` 的两大核心优势：**

| 优势 | 说明 |
|------|------|
| **setup 只执行一次** | `for b.Loop()` 保证无论 `-count` 设多少次，setup 只跑 1 次，计时更准确 |
| **返回值不被优化** | 循环变量天然保持活跃，编译器无法优化掉循环体 |

**典型对比场景：**

```go
// 测量带类型断言的函数的真实性能

// ❌ 旧写法：编译器可能把断言结果优化掉
func BenchmarkTypeAssert(b *testing.B) {
    var iface interface{} = &MyStruct{}
    for i := 0; i < b.N; i++ {
        _ = iface.(*MyStruct) // 编译器认为结果无用
    }
}

// ✅ 新写法：b.Loop 保证返回值被使用
func BenchmarkTypeAssertLoop(b *testing.B) {
    var iface interface{} = &MyStruct{}
    for b.Loop() {
        _ = iface.(*MyStruct) // b.Loop 保证返回值活跃
    }
}
```

**高频追问：**

**Q：为什么 Go 1.24 要引入 `b.Loop`？**
> 解决「**基准测试结果不稳定**」和「**setup 耗时干扰测量**」两个痛点。旧写法中 setup 和被测代码混在一起需要手动 `StopTimer`，容易出错；`b.Loop` 自动保证 setup 在循环外执行、返回值不被优化，让基准测试更精确。

**Q：`b.Loop` 和 `for range b.N` 哪个更好？**
> Go 1.24+ 推荐 `b.Loop`。`for range b.N` 编译器优化空间更大（尤其函数参数和返回值），`b.Loop` 专为基准测试设计，行为更可预测。

---

### 6. micro-benchmark 的局限性

**重要警告**：micro-benchmark 不等于真实性能。

```
micro-benchmark 失真场景：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. 忽略了缓存效应：真实系统有多个组件竞争缓存
2. 忽略了 GC 压力：benchmark 单独运行，真实系统 GC 会叠加
3. 数据规模失真：1KB vs 1MB 数据，算法行为完全不同
4. 热路径 vs 冷路径：真实代码有分支预测失败、TLB miss
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

正确做法：
1. micro-benchmark 用来优化"热函数"
2. 真实性能用 Profiling（pprof）在生产环境验证
3. 两者结合：benchmark 找方向，pprof 验证效果
```

---

## 高频追问

**Q：`b.N` 为什么不需要手动设置？**

> `b.N` 是 benchmark 框架自动调整的迭代次数。框架从 N=1 开始，每次翻倍，直到运行时间超过 1 秒（默认），然后用最终 N 计算每次操作的平均时间。手动设置 `-benchtime=10s` 可增加运行时间，提高精度。

**Q：benchmark 和 profiling 怎么结合？**

> 先用 benchmark 找到最慢的函数（`go test -bench=.`），再用 `go test -cpuprof=cpu.prof` 生成 profile，用 `go tool pprof` 分析热函数。benchmark 适合做"A 比 B 快多少"的定量对比，pprof 适合找"哪个函数最慢"的根因。

---

## 延伸阅读

- [Go benchmark 官方文档](https://pkg.go.dev/testing#B)
- [benchstat 工具](https://golang.org/x/perf/cmd/benchstat)
- [Athens 性能调优案例](https://www.pgly.com/2020/12/08/how-we-cut-our-go-test-times-in-half-with-benchmarks/)
