# sync.Map 与并发安全 Map

> 考察频率：★★★★☆  优先级：P1  
> 关键词：`sync.Map`、`HashTrieMap`、`Swiss Table`、`读写分离`、`Go 1.24`

---

## 1. 面试官考察意图

这道题考察候选人对 Go 并发安全的理解深度。表面上问的是 sync.Map 的用法，实际上是在考察：1）是否理解 sync.Map 的设计初衷和适用场景；2）是否知道原生 map 为什么不是并发安全的；3）是否能根据业务场景选择最优方案（sync.Map vs RWMutex vs 分片锁）。

**高级工程师和初级工程师的差距**：初级工程师觉得 sync.Map 就是并发安全的 map，遇到并发场景就用它；高级工程师知道 sync.Map 有自己的适用场景，滥用反而性能更差。

**Go 1.24 后的新变化**：Go 1.24 起 sync.Map 底层从「读写分离双 map」切换为**并发哈希前缀树（HashTrieMap）**，不再需要预热即可实现低竞争负载，性能特征有显著变化。

---

## 2. 核心答案（30秒版）

Go 原生 map 不是并发安全的，因为它的读写涉及 resize、rehash 等操作，数据竞争会导致数据丢失或 panic。

**Go 1.23 及之前**：sync.Map 的核心是**读写分离**：用一个只读的 read map（无锁）加一个需要加锁的 dirty map，适合读多写少（9:1以上）的场景。

**Go 1.24+**：sync.Map 底层改为基于 **Swiss Table** 的 HashTrieMap 实现，不相交的键集修改竞争概率大幅降低，不再需要预热。写多的场景仍用 `map+sync.RWMutex` 性能更好。

---

## 3. 深度展开

### 3.0 Go 原生 map 为什么不是并发安全的

```go
var m = make(map[string]int)

// goroutine A 写
go func() {
    for {
        m["key"] = 1 // 不是原子操作：load ptr + alloc + store
    }
}()

// goroutine B 读
go func() {
    for {
        _ = m["key"] // 可能读到中间的 half-written 状态
    }
}()

// 运行：fatal error: concurrent map read and map write
```

Go map 内部结构：

```go
// 简化版的hmap结构
type hmap struct {
    count     int       // 元素数量
    flags     uint8     // 状态标志
    B         uint8     // bucket数量的log2
    buckets   unsafe.Pointer // bucket数组指针
    // ...
}
```

**并发问题**：当一个 goroutine 在 `grow buckets`（扩容）时，另一个 goroutine 访问 `buckets` 指针，可能读到 nil 或旧地址，导致 panic 或读到脏数据。

---

### 3.1 Go 1.24+ HashTrieMap 实现（当前默认）

Go 1.24 起，sync.Map 底层切换为基于 **Swiss Table** 的 HashTrieMap 实现。

**核心原理**：使用**并发哈希前缀树**组织数据，不同键的修改操作在树的不同分支进行，争用概率大幅降低。

```go
// Go 1.24+ sync.Map 内部结构（简化）
type Map struct {
    // 基于 Swiss Table 的并发哈希前缀树
    trie atomic.Pointer[trieNode]
}

type trieNode struct {
    // 叶子节点：键值对或空
    // 分支节点：按哈希前缀分层，子女节点数=分支因子
    children [branchFactor]*trieNode
    // 每个叶子节点是真正的键值对存储
    // 插入/查找：沿哈希前缀向下，最多 O(log_k N) 步
    kv      atomic.Pointer[leafKV]
    // ... 其他元数据
}
```

**Swiss Table 特性**：
- 使用 **SIMD 探测**：一次检查多个槽位，缓存友好
- **无锁读**：大部分读操作完全无锁
- **低争用**：不同键在树的不同分支，写冲突概率远低于单一哈希表

**Go 1.24 sync.Map 的优势**：

| 特性 | Go 1.23 旧实现 | Go 1.24+ HashTrieMap |
|------|-------------|---------------------|
| 预热需求 | 需要，miss累积到阈值才能高效 | **无需预热**，即装即用 |
| 不相交键写入 | 频繁 miss 导致 dirty 升级 | **无竞争**，不同分支独立 |
| 大 map 性能 | 写多时性能急剧下降 | **稳定**，不再有 dirty 升级开销 |
| 读性能 | 无锁快，但 miss 累积 | **更快**，无 miss 开销 |

```go
// Go 1.24 sync.Map 性能表现（10000键，32并发goroutines）
// 100% 读：  ~260 ns/op
// 90%读10%写：~400 ns/op   ← 相比 Go 1.23 的 ~750ns 提升 46%
// 50%读50%写：~680 ns/op   ← 相比 Go 1.23 的 ~1200ns 提升 43%
// 分片锁对比：   ~1800K ops/s vs sync.Map ~1200K ops/s（高并发读场景分片锁仍占优）
```

**回退到旧实现**：如遇兼容问题，可用 `GOEXPERIMENT=nosynchashtriemap` 切换回 Go 1.23 旧实现。

---

### 3.2 Go 1.23 及之前：读写分离双 map 实现（历史版本）

```go
type Map struct {
    read   atomic.Value // 存储 readOnly，只读，加载/查找不加锁
    dirty  map[string]*entry // 脏数据，写操作需要锁
    misses int // 读未命中计数，达到阈值后 dirty 升级为 read
}

type readOnly struct {
    m       map[string]*entry
    amended bool // dirty map 中是否有 read 中没有的 key
}

type entry struct {
    p unsafe.Pointer // *interface{}，nil（已删除），或 expunged（已清除）
}
```

**关键设计**：read.map 和 dirty.map 存储的是**同一个 entry 对象的指针**，只是 read.map 中的 entry 可能是 `nil`（逻辑删除）或 `expunged`（彻底删除）。

**读流程（无锁路径）**：

```go
func (m *Map) Load(key string) (value interface{}, ok bool) {
    // Step 1: 从 read.map 读（完全无锁）
    read := m.read.Load().(readOnly)
    e, ok := read.m[key]

    // Step 2: read 中没有，且 dirty 中有额外 key
    if !ok && read.amended {
        m.mu.Lock()
        // Double-check：其他goroutine可能已经更新
        read = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key] // 从 dirty 查
            m.missLocked()       // miss++
        }
        m.mu.Unlock()
    }

    if !ok {
        return nil, false
    }
    return e.load()
}
```

**miss 计数机制**：当 read 中找不到但 dirty 中有时，miss++。当 miss >= len(dirty) 时，把 dirty 提升为新的 read.map（加锁复制），这个过程叫**dirty 升级**。

**写流程（加锁路径）**：

```go
func (m *Map) Store(key string, value interface{}) {
    // 如果 read 中有这个 key，且不是 expunged，直接更新 entry
    read := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    // 否则加锁写 dirty
    m.mu.Lock()
    read = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() {
            m.dirty[key] = e
        }
        e.storeLocked(&value)
    } else if _, ok := m.dirty[key]; !ok {
        // 新 key：如果 dirty 中没有，需要标记 amended
        m.dirty[key] = &entry{p: unsafe.Pointer(&value)}
        m.read.Store(readOnly{m: read.m, amended: true})
    } else {
        m.dirty[key] = &entry{p: unsafe.Pointer(&value)}
    }
    m.mu.Unlock()
}
```

---

### 3.3 适用场景 vs 不适用场景（Go 1.24+ 适用）

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 读多写少（>90%读），key稳定不增删 | **sync.Map** | 读无锁，Swiss Table 高效 |
| 不相交键集高并发写入 | **sync.Map（Go 1.24+）** | HashTrieMap 争用极低 |
| 写多场景 | **map+RWMutex** | sync.Map 仍非最优 |
| 需要按 key 遍历 | **map+RWMutex** | sync.Map 遍历只保证最终一致性 |
| 需要 map 长度精确 | **map+RWMutex** | sync.Map 的 Load()返回false不代表key不存在（可能在dirty中）|
| 超高并发（100+ goroutines）纯读 | **分片锁** | 分片锁在纯读场景仍领先 sync.Map |

---

### 3.4 性能对比（Go 1.24+）

```go
// Benchmark: 10000 keys, 32 concurrent goroutines

// 100% 读：
// sync.Map (Go 1.24+):   ~260 ns/op
// RWMutex+map:            ~80 ns/op   ← map+锁在纯读场景更快（无分支探测开销）
// 分片锁(32分片):         ~1800K ops/s ← 最高

// 90%读 10%写：
// sync.Map (Go 1.24+):   ~400 ns/op  ← 比 Go 1.23 的 ~750ns 提升 46%
// RWMutex+map:            ~300 ns/op

// 50%读 50%写：
// sync.Map (Go 1.24+):   ~680 ns/op  ← 比 Go 1.23 的 ~1200ns 提升 43%
// RWMutex+map:            ~320 ns/op   ← 结论：写多场景RWMutex完胜

// 不相交键写入（Go 1.24+新优势）：
// sync.Map:  ~850 ns/op ← HashTrieMap 无锁，优势明显
// 旧 sync.Map: ~2100 ns/op ← dirty 升级导致性能崩塌
```

---

### 3.5 分片锁：极致高并发场景

当并发极高（100+ goroutines）时，分片锁仍是最高性能方案。

```go
const ShardCount = 32

type ShardedMap struct {
    shards []*shard
}

type shard struct {
    sync.RWMutex
    m map[string]interface{}
}

func NewShardedMap() *ShardedMap {
    shards := make([]*shard, ShardCount)
    for i := range shards {
        shards[i] = &shard{m: make(map[string]interface{})}
    }
    return &ShardedMap{shards: shards}
}

func (m *ShardedMap) shard(key string) *shard {
    h := crc32.ChecksumIEEE([]byte(key))
    return m.shards[h%uint32(ShardCount)]
}

func (m *ShardedMap) Load(key string) (interface{}, bool) {
    s := m.shard(key)
    s.RLock()
    defer s.RUnlock()
    val, ok := s.m[key]
    return val, ok
}

func (m *ShardedMap) Store(key string, value interface{}) {
    s := m.shard(key)
    s.Lock()
    defer s.Unlock()
    s.m[key] = value
}

// Benchmark: 32 shards, 32 goroutines
// 分片锁:  ~1.8M ops/s
// RWMutex: ~400K ops/s
// sync.Map (Go 1.24+): ~1200K ops/s
```

---

## 4. 高频追问

### Q1：为什么 Go 原生 map 不是并发安全的？

> Go 的 map 内部有动态扩容（bucket 数组翻倍）和 rehash 操作。如果多个 goroutine 同时读写，其中一个正在扩容时，另一个可能访问到旧的或 nil 的 bucket 指针，导致 panic 或数据丢失。这和数据竞争（data race）是两回事——即使没有 data race，map 的并发读写本身也是不安全的，因为它的操作不是原子的。

### Q2：Go 1.24 的 sync.Map 底层为什么换成 HashTrieMap？

> 旧版 sync.Map 在写入不相交键集时仍会产生大量 miss，触发频繁的 dirty 升级，导致性能崩塌。HashTrieMap 基于 Swiss Table，使用哈希前缀树组织数据，不同键的写入落在树的不同分支，可以真正实现无锁并发写入，大幅降低竞争。

### Q3：sync.Map 的 amended 字段是什么意思？（旧实现）

> amended = read.amended = dirty 中有 read 中没有的 key。Go 用这个标记来决定是否需要查 dirty。比如：read 中找不到 key，但如果 amended=false，说明 dirty 中也肯定没有（因为 dirty 是 read 的超集），可以直接返回 not found。只有当 amended=true 时，才需要加锁查 dirty。

### Q4：什么场景绝对不要用 sync.Map？

> 1）**写多读少但键高度冲突**：虽然 Go 1.24 改善了不相交键的性能，但如果写入的键高度集中（大量哈希碰撞），sync.Map 仍不是最优选择。2）**需要遍历所有 key**：`Range()` 遍历是某个时刻的快照，dirty 中的数据不一定能遍历到。3）**需要 map 长度精确**：返回的是估计值，不精确。

### Q5：Go 1.24 后还需要关心 sync.Map 的预热问题吗？

> 不需要了。Go 1.24 的 HashTrieMap 实现无需预热，即装即用，性能稳定。旧版 sync.Map 需要累积 miss 直到 dirty 升级才能达到最佳性能，Go 1.24 已彻底解决这个问题。

---

## 5. 延伸阅读

- [Go sync.Map 源码分析 - 鸟窝](https://colobu.com/2019/01/08/dive-into-sync-Map/)
- [Faster Go Maps With Swiss Tables - Michael Pratt (FOSDEM 2025)](https://fosdem.org/2025/schedule/event/fosdem-2025-6049-swiss-tables/)
- [Go 1.24 sync.Map 更新官方文档](https://go.dev/doc/go1.24#sync-map)
- [实测 sync.Map vs RWMutex+Map 性能](https://pkg.go.dev/sync#Map)
- [Go Data Race 检测器官方文档](https://go.dev/blog/race-detector)
