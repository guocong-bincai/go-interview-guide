# CPU 飙升排查

> 考察频率：★★★★☆  难度：★★★★☆

## 排查思路

```
CPU 100% → 定位热点函数 → 分析代码逻辑 → 修复
```

## 工具：pprof CPU 分析

### 开启 CPU profiling

```go
import _ "net/http/pprof"

// 或主动采样
import (
    "runtime/pprof"
    "os"
)
f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
```

### 采集 CPU profile

```bash
# 采集 30 秒
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof

# 或使用 go-torch
go-torch -duration=30 http://localhost:6060
```

### 分析 CPU profile

```bash
# 进入交互式分析
go tool pprof cpu.prof

# 查看 top 10 热点
(pprof) top 10

# 查看特定函数
(pprof) list 函数名

# 生成火焰图
(pprof) web
```

---

## 常见 CPU 飙升原因

### 1. 热循环中的字符串拼接

```go
// 泄漏代码：O(n²) 复杂度
func concat(strings []string) string {
    result := ""
    for _, s := range strings {
        result += s  // 每次创建新字符串
    }
    return result
}

// 修复：使用 strings.Builder
func concatFixed(strings []string) string {
    var sb strings.Builder
    for _, s := range strings {
        sb.WriteString(s)
    }
    return sb.String()
}
```

### 2. 正则表达式预编译

```go
// 泄漏代码：每次调用编译正则
func validateEmail(email string) bool {
    matched, _ := regexp.MatchString(`^[a-z]+@[a-z]+\.[a-z]+$`, email)
    return matched
}

// 修复：全局预编译
var emailRegex = regexp.MustCompile(`^[a-z]+@[a-z]+\.[a-z]+$`)

func validateEmailFixed(email string) bool {
    return emailRegex.MatchString(email)
}
```

### 3. 大量小对象分配

```go
// 泄漏代码：频繁创建小结构体
func process(items []int) []*Result {
    results := make([]*Result, 0, len(items))
    for _, item := range items {
        results = append(results, &Result{Value: item}) // 大量小对象
    }
    return results
}

// 修复：复用对象池
var resultPool = sync.Pool{
    New: func() any {
        return &Result{}
    },
}

func processFixed(items []int) []*Result {
    results := make([]*Result, 0, len(items))
    for _, item := range items {
        r := resultPool.Get().(*Result)
        r.Value = item
        results = append(results, r)
        resultPool.Put(r) // 复用
    }
    return results
}
```

### 4. Goroutine 死循环

```go
// 泄漏代码：错误的循环逻辑
func process() {
    for {
        select {
        case msg := <-ch:
            go handle(msg) // 每次都启动新 goroutine
        default:
            // 没有消息时继续循环
        }
    }
}

// 修复：添加退出条件或使用 worker pool
func processFixed() {
    var wg sync.WaitGroup
    for msg := range ch {
        wg.Add(1)
        go func(m string) {
            defer wg.Done()
            handle(m)
        }(msg)
    }
    wg.Wait()
}
```

---

## CPU 热点分析实战

### 1. 采集 profile

```bash
# 在服务运行期间采集
curl -s http://localhost:6060/debug/pprof/profile?seconds=60 > cpu_60s.prof
```

### 2. 分析火焰图

```bash
# 安装 go-torch
pip install go-torch

# 生成火焰图
go-torch cpu_60s.prof

# 或使用 pprof web
go tool pprof -http=:8080 cpu_60s.prof
```

### 3. 解读火焰图

- **顶层**：消耗 CPU 最多的函数
- **宽度**：该函数占用的 CPU 比例
- **颜色**：通常无意义（可区分库/用户代码）

---

## 常见优化策略

| 问题 | 优化方案 |
|------|---------|
| 字符串拼接 | strings.Builder |
| 正则未编译 | regexp.MustCompile |
| 小对象分配 | sync.Pool |
| JSON 编解码 | json-iterator |
| Map 竞争 | 分片锁或 sync.Map |

---

## 总结：CPU 飙升排查流程

```
1. 采集：curl /debug/pprof/profile
2. 分析：go tool pprof -http=:8080
3. 查看：top / list 函数名
4. 定位：web 生成火焰图
5. 修复：优化算法、减少分配、预编译
6. 验证：重新采集对比
```

### 快速检查清单
- [ ] pprof 是否开启？
- [ ] top 中最高的是什么函数？
- [ ] 是否有大量小对象分配？
- [ ] 正则是否每次编译？
- [ ] 字符串拼接是否用了 +？
