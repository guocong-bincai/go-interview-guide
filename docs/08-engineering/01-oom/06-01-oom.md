# OOM 排查实战

> 考察频率：★★★★☆  难度：★★★★☆

## 常见 OOM 场景

1. **堆内存泄漏**：对象被持续引用无法回收
2. **Goroutine 泄漏**：大量 goroutine 无法退出
3. **大对象分配**：单次分配超过阈值
4. **内存碎片**：可用内存足够但分配失败

---

## 排查工具

### 1. pprof 内存分析

```go
import _ "net/http/pprof"
```

```bash
# 访问内存 profile
curl http://localhost:6060/debug/pprof/heap > heap.out

# 或在 Go 程序中
import (
    "runtime/pprof"
    "os"
)
f, _ := os.Create("heap.out")
pprof.WriteHeapProfile(f)
f.Close()
```

### 2. 分析 heap profile

```bash
# 查看火焰图
go tool pprof -http=:8080 heap.out

# 命令行分析
go tool pprof heap.out
(pprof) top 10    # 查看内存占用前10
(pprof) web      # 生成 SVG 火焰图
```

### 3. 查看存活对象

```bash
go tool pprof heap.out
(pprof) alloc_space    # 累计分配（含已回收）
(pprof) inuse_space    # 活跃使用
(pprof) list 函数名    # 查看特定函数详情
```

---

## 堆内存泄漏定位

### 案例：未关闭的 channel 导致泄漏

```go
// 泄漏代码
func leak() {
    ch := make(chan int)
    go func() {
        for i := range ch {  // 永远不会关闭
            fmt.Println(i)
        }
    }()
    // ch 永远不会被 close，goroutine 泄漏
}

// 修复
func fixed() {
    ch := make(chan int)
    go func() {
        for i := range ch {
            fmt.Println(i)
        }
    }()
    close(ch) // 关闭 channel
}
```

### 案例：map 持续写入不清理

```go
// 泄漏代码
var cache = make(map[string][]byte)

func addCache(key string, data []byte) {
    cache[key] = data  // 只增不减
}

// 修复：定期清理或使用 TTL
func cleanCache() {
    for k, v := range cache {
        if time.Since(v.timestamp) > 10*time.Minute {
            delete(cache, k)
        }
    }
}
```

### 案例：闭包引用大对象

```go
// 泄漏代码
var bigData []byte

func process() {
    result := make([]byte, 1024*1024)
    go func() {
        // 闭包引用了外部的 bigData
        processBig(bigData, result)
    }()
}

// 修复：避免不必要的引用
func processFixed() {
    result := make([]byte, 1024*1024)
    data := bigData // 复制需要的部分
    go func(d []byte) {
        processBig(d, result) // 只传递需要的
    }(data)
}
```

---

## 内存泄漏排查流程

### Step 1：确认是否真的泄漏

```bash
# 多次采样对比，内存持续增长说明泄漏
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/heap?seconds=30
```

### Step 2：定位泄漏点

```bash
# 对比两个时间点的 heap
curl -s http://localhost:6060/debug/pprof/heap > heap1.out
# 等待一段时间 / 跑更多请求
curl -s http://localhost:6060/debug/pprof/heap > heap2.out

go tool pprof -diff_base=heap1.out heap2.out
```

### Step 3：查看具体函数

```bash
(pprof) top
(pprof) list processRequest
(pprof) web  # 生成调用链路
```

---

## GOGC 调优

Go 默认 GOGC=100（内存翻倍时触发 GC）。

```bash
# 减少 GC 频率（占用更多内存，但 GC 更少）
GOGC=200 ./app

# 主动触发 GC（不影响正常请求）
runtime.GC()
```

### 内存限制（Go 1.19+）

```bash
GOMEMLIMIT=8GiB ./app  # 限制最大内存
```

---

## 总结：OOM 排查清单

| 步骤 | 操作 |
|------|------|
| 1. 确认 OOM | 查看日志、Dmesg |
| 2. 开启 pprof | 添加 `import _ "net/http/pprof"` |
| 3. 采样 heap | `curl heap > heap.out` |
| 4. 分析 | `go tool pprof heap.out` |
| 5. 对比 | 两次采样 diff_base |
| 6. 定位 | `list 函数名` 查看详情 |
| 7. 修复 | 关闭资源、清理缓存、修复闭包 |

### 常见泄漏模式
1. **全局 map/channel** 只增不减
2. **Goroutine 阻塞** 无法退出
3. **闭包引用** 大对象
4. **定时任务** 累积未清理
5. **Listener/Connection** 未关闭
