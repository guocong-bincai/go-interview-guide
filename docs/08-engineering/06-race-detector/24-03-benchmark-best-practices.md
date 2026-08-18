# Benchmark：Go 性能测试方法论

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：testing.B、基准测试、优化策略、编译器优化、常见陷阱

## 面试官考察意图

压测（load testing）文章讲了系统级的性能验证，但**没有覆盖代码级的 benchmark**。高级工程师必须会写 benchmark、读懂 benchmark 结果、理解 benchmark 的局限性。

这道题考的是：**你能不能用科学的方法证明"我的代码比你的快"，而不是凭感觉说"我觉得更快了"。**

---

## 核心答案（30 秒版）

Go 的 `go test -bench` 是最基础也最常用的性能测试工具。写好 benchmark 的关键是三点：一是**跑足够多的迭代次数**（让稳态样本占主导），二是**使用 b.StopTimer()/b.StartTimer() 排除 setup/teardown 时间**，三是**注意编译器优化（不要 benchmark 死代码）**。输出关注 ns/op（每操作耗时）和 B/op + allocs/op（内存分配）。一个合格的 benchmark 应该同时报告这三个指标。

---

## Go Benchmark 基础

### 基本结构

```go
package main_test

import "testing"

// 基准函数必须以 BenchmarkXxx(b *testing.B) 签名
func BenchmarkStringConcat(b *testing.B) {
    for i := 0; i < b.N; i++ { // b.N 由 Go 自动调节
        result := ""
        for j := 0; j < 100; j++ {
            result += string(rune('a' + j%26))
        }
    }
}
```

### 运行方式

```bash
# 跑所有 benchmark
go test -bench=. ./...

# 只跑特定 benchmark
go test -bench=BenchmarkStringConcat ./...

# 跑 n 个迭代后停止
go test -bench=. -benchtime=5s ./...

# 跑 m 次重复
go test -bench=. -count=5 ./...

# 看内存分配
go test -bench=. -benchmem ./...
```

### 输出解读

```
pkg: github.com/example/myapp
BenchmarkStringConcat-8         	  100000	   12345 ns/op	    5120 B/op	      20 allocs/op
BenchmarkStringBuilder-8        	  500000	     3456 ns/op	    1024 B/op	       3 allocs/op

解读：
- CPU cores: -8 表示用了 8 核
- Iterations: 100000 表示跑了 10 万次
- ns/op: 每次操作平均 12345ns = 12.3μs
- B/op: 每次操作分配 5120 bytes
- allocs/op: 每次操作分配 20 次
```

> **关键公式：每秒吞吐量 = 1,000,000,000 / ns/op**
> 比如 12345 ns/op → ~81 Kops/s

---

## Benchmark 最佳实践

### 规则 1：用 StopTimer/StartTimer 排除 setup 时间

```go
// ❌ setup 时间会计入基准
func BenchmarkBadSetup(b *testing.B) {
    data := loadDataFromFile("large_data.json") // 每次都执行
    for i := 0; i < b.N; i++ {
        process(data)
    }
}

// ✅ 在循环外加载数据
func BenchmarkGoodSetup(b *testing.B) {
    data := loadDataFromFile("large_data.json") // 只加载一次
    
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        copy := make([]byte, len(data))
        copy(copy, data) // reset state
        b.StartTimer()
        
        process(copy)
    }
}
```

### 规则 2：防止编译器优化掉工作

```go
// ❌ 编译器可能直接优化掉整个循环（如果结果没被使用）
func BenchmarkDeadCode(b *testing.B) {
    for i := 0; i < b.N; i++ {
        x := sort.Slice( []int{3,1,2}, ...) // 返回值没用
    }
}

// ✅ 把结果写到 b.ReportMetric 或全局变量
var sink int // global variable prevents optimization

func BenchmarkPreventOptimize(b *testing.B) {
    for i := 0; i < b.N; i++ {
        x := computeSomething()
        sink += x // prevent dead code elimination
    }
}

// 更推荐的方式：用 b.ReportAllocs() + 返回结果
func BenchmarkWithReport(b *testing.B) {
    for i := 0; i < b.N; i++ {
        result := computeSomething()
        if i == b.N-1 {
            b.ReportMetric(float64(result), "result")
        }
    }
}
```

### 规则 3：使用并行 Benchmark 做对比

```go
// Go 1.7+ 支持并行 benchmark
func BenchmarkParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            doWork()
        }
    })
}

// 不同实现对比：Subbenchmark 模式
func BenchmarkSerializer(b *testing.B) {
    data := generateTestData()
    
    b.Run("json", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            json.Marshal(data)
        }
    })
    
    b.Run("msgpack", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            msgpack.Encode(data)
        }
    })
    
    b.Run("protobuf", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            proto.Marshal(data)
        }
    })
}
```

---

## Benchmark 常见陷阱

### 陷阱 1：warmup period 不够

```
Go benchmark 的 b.N 从 1 开始指数增长 (1, 2, 4, 8, 16...)
前几次迭代时间通常偏长（CPU cache miss、GC 等影响）

✅ 解决：忽略最初的几次迭代
```

Go 本身会自动跳过不稳定的早期迭代——它会在计算 ns/op 时剔除异常值。但你可以通过 `-benchtime` 显式指定较长的时间来获得更准确的结果。

### 陷阱 2：microbenchmark ≠ real-world performance

```
微基准测试结果好 ≠ 真实服务表现好

原因：
- microbenchmark 通常是纯 CPU 密集
- 真实服务涉及 DB/MQ/网络 IO 延迟
- Go scheduler、GC 压力在微 benchmark 中不明显
```

### 陷阱 3：忽略分配成本

```go
// ❌ 只看速度不看内存
func BenchmarkFastButLeaky(b *testing.B) {
    for i := 0; i < b.N; i++ {
        makeBigSlice(10000) // 很快，但每次分配 80KB
    }
}

// ✅ 同时看速度和内存
// go test -bench=. -benchmem
```

### 陷阱 4：测试环境不一致

```
⚠️ 不要在 running `make build` 的同时跑 benchmark
⚠️ 不要用 laptop 的 benchmark 决定生产容量
⚠️ CPU boost/hyperthreading 会让结果不稳定

✅ 用固定 CPU 频率的服务器跑
✅ 关其他进程/后台任务
✅ 多次取平均值（-count > 1）
```

---

## 进阶：使用 pprof 分析 benchmark

```bash
# 跑 benchmark 并采集 CPU profile
go test -bench=. -benchmem -cpuprofile=cpu.prof ./...
go tool pprof cpu.prof

# 跑 benchmark 并采集 memory profile
go test -bench=. -benchmem -memprofile=mem.prof ./...
go tool pprof mem.prof
```

示例：比较两个实现的详细差异

```bash
# 方案 A
go test -bench=BenchmarkSerialize -benchmem -run=^$ -cpuprofile=a.prof ./...
# 方案 B  
go test -bench=BenchmarkSerialize -benchmem -run=^$ -cpuprofile=b.prof ./...

# 对比
go tool pprof -base=a.prof b.prof
(pprof) top
```

---

## 高频追问

**Q：如何判断 benchmark 结果是可靠的？**
A：三个指标：1）n (>1000 次迭代)，2）stddev < 5%（通过 `-benchtime` 延长确保稳定），3）与上次相比的变化幅度远大于 stddev。如果改进很小（<1%）且接近 stddev，不算真实改进。

**Q：benchmark 能检测内存泄漏吗？**
A：不能直接检测。但如果你在 long-running 的 benchmark 中看到分配的总量持续增长（而非每次迭代重置），那可能是 leak 的迹象。可以用 `go test -bench=. -memprofile=mem.prof` 然后 `go tool pprof` 来确认。

**Q：`-benchmem` 的作用是什么？**
A：默认情况下 benchmark 只显示 ns/op。加上 `-benchmem` 后会额外显示 B/op（每次操作的字节分配量）和 allocs/op（每次操作的分配次数）。对评估 GC 压力至关重要。

**Q：benchmark 结果在不同机器上差距大怎么办？**
A：Microbenchmarks 的本质就是测量相对差异——只要同一台机器的两次运行可比较即可。跨机器比较时应该关注比例关系（比如"方案 A 比 B 快 2x"），而不是绝对数字。

---

## 延伸阅读

- [Go Blog: Writing Go Benchmarks](https://go.dev/doc/code/benchmark)
- [Go Testing Package Documentation](https://pkg.go.dev/testing#B)
- [Uber Go Guide: Benchmarking](https://github.com/uber-go/guide/blob/master/style.md#benchmarking)
