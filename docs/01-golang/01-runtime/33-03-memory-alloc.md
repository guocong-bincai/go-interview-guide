[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 运行时原理](../README.md)

---

# Go 内存分配器：mspan/mcache/mheap 与内存分配流程

## 面试官考察意图

考察候选人对 Go 运行时内存管理子系统的理解深度。
初级只知道"栈上分配快、堆上分配慢"，高级要能讲清楚 **Go 内存分配器的层级结构（mcache → mcentral → mheap）、mspan 的组织方式、分级缓存对分配性能的优化**，以及如何在生产中诊断内存分配问题。结合逃逸分析和 GC 调优，这是 P0 级知识点。

---

## 核心答案（30 秒版）

Go 内存分配采用**多级缓存 + 对象大小分桶**策略：

```
Goroutine 本地分配（mcache）
    ├── 无需加锁 → 极快 ~1-2ns
    ├── 每 P 一个
    └── 负责 ≤ 32KB 对象

mcentral（全局共享）
    ├── 需要加锁
    └── 按 size class 组织 mspan

mheap（堆内存映射）
    ├── 全局锁
    └── 负责大对象（> 32KB）和 mspan 调度
```

**mspan** 是内存分配的基本单位，按对象大小分为约 70 个 size class（8B~32KB）。分配时先查 mcache，找不到则从 mcentral 拿 mspan，再从 OS 拿内存时需要 mheap 向 OS 映射。

---

## 深度展开

### 1. 为什么要多级缓存

直接向 OS 申请内存代价极高（系统调用 + 映射）。Go 采用多级缓存策略：

| 层级 | 访问速度 | 作用域 | 锁竞争 |
|------|----------|--------|--------|
| mcache | ~1ns（无锁） | 每 P 一个 | 无 |
| mcentral | ~100ns | 全局，按 size class | 有（但比 OS 调用快） |
| mheap | ~1000ns | 全局 | 全局锁（最慢） |

**关键数据**：mcache 命中时分配 ~1-2ns，从 mheap 分配 ~100ns+，差距百倍。

### 2. mspan 结构

```go
// runtime/mspan.go（简化）
type mspan struct {
    next          *mspan
    prev          *mspan
    startAddr     uintptr         // 起始地址
    npages        uintptr         // 页数
    nelems        uintptr         // 对象数量
    allocBits     *uint8          // 分配位图（每位表示一个对象槽是否占用）
    allocCount    uint16          // 已分配对象数
    sizeclass     int8             // size class 索引，0=大对象
    elemsize      uintptr          // 每个对象大小
    // ...
}
```

**mspan 按对象大小分为两类**：
- **normal mspan**：存放小对象（≤ 32KB），按 size class 划分
- **large mspan**：单次分配 > 32KB，直接由 mheap 管理

### 3. 分配流程

```
用户代码：make([]int, 100)
    │
    ▼
mcache.Alloc(sizeclass)     ← P0 锁，无竞争
    │
    ├── 有空闲槽位 → 直接返回指针（~1ns）
    │
    └── 无空闲槽位 → 从 mcentral 获取新 mspan
                        │
                        ├── mcentral 有空 mspan → 返回（加锁 ~100ns）
                        │
                        └── mcentral 空 → 向 mheap 申请新 mspan
                                            │
                                            └── mheap 有空间 → 映射（加全局锁 ~1000ns）
                                                    │
                                                    └── mheap 不够 → 向 OS 申请（sbrk/mmap）
```

### 4. size class 分桶机制

Go 将小对象按大小分为约 70 个 size class：

| Size Class | 对象大小范围 | 每槽大小 |
|------------|-------------|---------|
| 1 | 1~8 B | 8 B |
| 2 | 9~16 B | 16 B |
| 3 | 17~32 B | 32 B |
| ... | ... | ... |
| ~67 | 28KB~32KB | 32 KB |

> 实际分桶更细（1, 8, 16, 32, 48, 64, 80... 递增到 32KB），共约 70 个 class。

**为什么这样设计**：减少内存碎片。对象按"够用的最小桶"分配，空闲浪费在桶内，不产生外部碎片。

### 5. mcache 结构

```go
// runtime/mcache.go（简化）
type mcache struct {
    // 约 70 个 size class 对应的 mspan 指针
    spans [_NumSizeClasses]*mspan
    
    // 分配器指针（在 span 内移动）
    allocBytes  uintptr  // 剩余空间起始位置
    
    // 其他字段...
}
```

mcache 实际上持有每个 size class 对应的**当前活跃 mspan**，分配时直接从中取槽位。

### 6. 释放流程（归还 OS）

```
Golang GC (mark + sweep)
    │
    ▼
mspan 中存活对象复制到新 mspan
不存活对象标记为空闲
    │
    ├── mspan 还有空闲对象 → 保留在 mcentral
    │
    └── mspan 全部空闲超过一定时间 → 归还 mheap
                                        │
                                        └── mheap 累积一定量 → 释放给 OS（madise）
```

**注意**：Go 不会立即将内存归还 OS（避免频繁映射/取消映射），而是保留在 mheap 中供后续使用。`debug.FreeOSMemory()` 可强制归还。

### 7. 与 GC 的关系

GC 的 mark 阶段会遍历所有 mspan 中的对象，sweep 阶段重置 mspan 的 allocBits 标记哪些槽位可用。GC 结束后 mcache 中的 mspan 被清空（防止泄漏对象被误认为存活）。

**Go 1.26 新增的 goroutine leak profile** 利用 GC 追踪每个 goroutine 阻塞位置，如果阻塞点本身不可达，则 goroutine 判定为泄漏（配合 mspan 的 allocBits 分析）。

---

## 生产问题与排查

### 问题 1：大量小对象导致 mcache 频繁换入

**现象**：高 QPS 服务，GC 频率高，每次 GC 后 mcache 被清空，分配变慢。

**分析**：
```go
// 观察分配情况
go build -ldflags='-压测后用 pprof 分析：
// 1. 查看 allocations
go tool pprof -http=:6060 http://localhost:6060/debug/pprof/allocs

// 2. 对比 GC 前后的分配速率
```

**优化**：减少小对象数量，用 `sync.Pool` 复用短期对象（但注意 GC 会清空）。

### 问题 2：大对象分配拖慢 mheap

**现象**：偶尔出现毫秒级分配延迟。

**原因**：> 32KB 的对象直接从 mheap 分配，需要加全局锁。

**优化**：
```go
// 反例：频繁分配大 slice
func process(b []byte) {
    // b 每次都是新的大对象
}

// 优化：复用大 buffer
var bufPool = sync.Pool{
    New: func() any {
        return make([]byte, 64*1024)
    },
}
```

### 问题 3：内存碎片

**现象**：RSS 高，但实际存活对象少。

**原因**：mspan 按固定 size class 分配，小对象用不完的槽位成为内部碎片。

**解决**：
- 按对象大小调整分配模式（如用 `[]byte` 池化代替频繁小分配）
- Go 1.24 Swiss Table 改善了 map 的内存局部性

---

## 高频追问

**Q：为什么 Go 不采用 glibc 的 ptmalloc？**

ptmalloc 每个线程有独立 arena，内存浪费严重（每个 arena 最小 1MB）。Go 的 mcache 方案锁竞争更少，且 mheap 统一管理，按需映射，避免预先占用大量虚拟地址空间。

**Q：mcache 是 per-P 还是 per-G？**

per-P。每个 P（逻辑处理器）有一个 mcache，G 执行时直接用当前 P 的 mcache，无锁分配。

**Q：Go 的栈分配和 mcache/mheap 有什么关系？**

没关系！goroutine 初始栈（2~8KB）在 mspan 中分配（堆上），后续按需增长/收缩。Go 1.25/1.26 对栈分配有进一步优化，小对象尽量在栈上分配，减少堆压力。

**Q：如何看一个对象的分配位置（堆/栈）？**

```bash
go build -gcflags='-m -m' . 2>&1 | grep "escapes to heap"
```

或者用 pprof：
```go
import _ "net/http/pprof"
_, _ = http.DefaultServeMux, nil // 启用 pprof
// 访问 http://localhost:6060/debug/pprof/heap 查看 allocations
```

**Q：mspan 的大小是如何计算的？**

mspan 大小 = `npages * pageSize`，pageSize = 8KB（大多数平台）。一个 mspan 最多包含 `npages * 8KB / elemsize` 个对象槽。

---

## 延伸阅读

- [Go 内存分配器设计](https://www.cncf.io/wp-content/uploads/2020/08/Go-Talk-Stealing-from-the-Go-allocator.pdf)（窃取式分配器）
- `src/runtime/mheap.go`、`src/runtime/mcache.go`、`src/runtime/mspan.go`（源码）
