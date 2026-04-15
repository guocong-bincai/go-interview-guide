# Go 基准测试规范：从入门到生产级

> 面试频率：★★★☆☆  考察角度：Benchmark 写法、性能对比、避免编译器优化

---

## 1. 基准测试基础

### 1.1 基准测试函数命名与结构

```go
import "testing"

// 基准测试函数必须以 Benchmark 开头
// 参数必须是 *testing.B
func BenchmarkHello(b *testing.B) {
    for i := 0; i < b.N; i++ {
        // 测试代码
    }
}
```

### 1.2 运行基准测试

```bash
# 运行基准测试，默认迭代 1 秒
go test -bench=BenchmarkHello

# 运行所有基准测试
go test -bench=.

# 指定迭代时间（5秒）
go test -bench=. -benchtime=5s

# 跑 1000 次后停止（用于稳定性测试）
go test -bench=. -benchmem -count=1000

# 内存分配统计
go test -bench=. -benchmem
```

### 1.3 输出解读

```
BenchmarkHello-8    1000000000    0.3000 ns/op    0 B/op    0 allocs/op
   │       │                │            │          │
   函数名  CPU核心数       总迭代次数    每次耗时   每次内存分配    每次分配字节数
```

---

## 2. 避免编译器优化干扰

### 2.1 死代码消除（Dead Code Elimination）

```go
// ❌ 错误：编译器发现结果没用，直接跳过计算
func BenchmarkBad(b *testing.B) {
    for i := 0; i < b.N; i++ {
        hello()
    }
}

func hello() string {
    return "hello"
}
```

```go
// ✅ 正确：用 runtime.KeepAlive 或赋值给全局变量
var result string  // 包级变量，防止优化

func BenchmarkGood(b *testing.B) {
    for i := 0; i < b.N; i++ {
        result = hello()
    }
}

// 或者用 testing.B.Log
func BenchmarkGood2(b *testing.B) {
    for i := 0; i < b.N; i++ {
        r := hello()
        b.Log(r)
    }
}
```

### 2.2 循环不变代码外提（Loop-Invariant Code Motion）

```go
// ❌ 错误：slice 每次循环都重新分配
func BenchmarkBad(b *testing.B) {
    for i := 0; i < b.N; i++ {
        buf := make([]byte, 1024)  // 循环内分配
    }
}
```

```go
// ✅ 正确：提到循环外
func BenchmarkGood(b *testing.B) {
    buf := make([]byte, 1024)  // 提到循环外
    for i := 0; i < b.N; i++ {
        // 使用 buf
        _ = buf
    }
}
```

### 2.3 用 ResetTimer 重设计时器

```go
func BenchmarkWithSetup(b *testing.B) {
    // 准备大量数据的耗时不应计入基准测试
    largeData := make([]int, 1000000)
    for i := range largeData {
        largeData[i] = i
    }

    b.ResetTimer()  // 重置计时器
    for i := 0; i < b.N; i++ {
        process(largeData)
    }
}
```

---

## 3. 精确测量：并行基准测试

### 3.1 Parallel 模式

```go
func BenchmarkParallel(b *testing.B) {
    // GOMAXPROCS 自动设置为 CPU 核心数
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            // 并行执行的内容
            _ = someFunction()
        }
    })
}
```

```bash
go test -bench=BenchmarkParallel -benchrun=1/1 -cpu=1,2,4,8
# 观察不同并发度下的吞吐量
```

### 3.2 对比测试：子基准测试

```go
func BenchmarkStringConcat(b *testing.B) {
    b.Run("Sprintf", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = fmt.Sprintf("%s%s", "hello", "world")
        }
    })

    b.Run("Join", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = strings.Join([]string{"hello", "world"}, "")
        }
    })

    b.Run("Builder", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var sb strings.Builder
            sb.WriteString("hello")
            sb.WriteString("world")
            _ = sb.String()
        }
    })
}
```

```bash
go test -bench=BenchmarkStringConcat -benchmem
```

---

## 4. 生产级基准测试模板

```go
package main

import (
    "testing"
    "math/rand"
)

// 被测试的函数
func process(data []int) int {
    sum := 0
    for _, v := range data {
        if v > 0 {
            sum += v
        }
    }
    return sum
}

// 生成测试数据
func generateData(n int) []int {
    data := make([]int, n)
    for i := range data {
        data[i] = rand.Intn(100)
    }
    return data
}

// ✅ 完整的基准测试
func BenchmarkProcess(b *testing.B) {
    // 1. 准备测试数据（不计入耗时）
    data := generateData(1000000)
    b.ResetTimer()
    b.ReportAllocs()  // 报告内存分配

    // 2. 运行基准测试
    var result int
    for i := 0; i < b.N; i++ {
        result = process(data)
    }

    // 3. 防止编译器优化掉 result
    if result == -1 {
        panic("unreachable")
    }
}

// ✅ 子基准测试对比
func BenchmarkProcessSize(b *testing.B) {
    sizes := []int{100, 1000, 10000, 100000, 1000000}

    for _, size := range sizes {
        size := size // 避免闭包问题
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            data := generateData(size)
            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                _ = process(data)
            }
        })
    }
}
```

---

## 5. PProf 在基准测试中的应用

```go
import "runtime/pprof"

func BenchmarkWithCPUProfile(b *testing.B) {
    // 生成 CPU Profile
    f, _ := os.Create("cpu.prof")
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()

    for i := 0; i < b.N; i++ {
        SomeFunction()
    }
}

func BenchmarkWithMemProfile(b *testing.B) {
    // 生成内存分配 Profile
    f, _ := os.Create("mem.prof")
    defer f.Close()
    pprof.WriteHeapProfile(f)

    for i := 0; i < b.N; i++ {
        SomeFunction()
    }
}
```

---

## 6. 常见坑与避坑指南

### 坑 1：benchmark 时间太短，结果不稳定

```bash
# 默认只跑 1 秒，结果波动大
go test -bench=. -benchtime=10s  # 跑 10 秒更稳定
```

### 坑 2：CPU 频率调度干扰

```bash
# 固定 CPU 频率后测试更准确
# Linux: sudo cpufreq-set -g performance
go test -bench=. -count=5  # 多跑几次观察稳定性
```

### 坑 3：误用 b.StopTimer / b.StartTimer

```go
// ❌ 滥用 StopTimer
func BenchmarkBad(b *testing.B) {
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        setup() // 每次循环都停计时
        b.StartTimer()
        run()
    }
}

// ✅ 用 ResetTimer + 循环外准备
func BenchmarkGood(b *testing.B) {
    setup()  // 循环外
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        run()
    }
}
```

---

## 7. 面试高频追问

**Q：如何避免编译器把被测代码优化掉？**
> 用 `testing.B.Log` 输出结果；或赋值给包级全局变量；或用 `runtime.KeepAlive`。关键是让编译器知道结果是被使用的。

**Q：go test -benchmem 各个字段含义？**
> `0 B/op` 是每次操作分配的字节数；`0 allocs/op` 是每次操作分配了几次。`ns/op` 是每次操作的纳秒数。

**Q：如何对比两个实现的性能？**
> 用子基准测试 `b.Run` 对比；或者用 `benchstat` 工具（`go install golang.org/x/perf/cmd/benchstat`）对比多次运行结果。
