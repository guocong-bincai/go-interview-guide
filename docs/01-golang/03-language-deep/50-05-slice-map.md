[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [💎 语言机制](../README.md)

---

# slice 与 map 底层原理：扩容策略、并发安全、与 Go 1.24 Swiss Table

## 面试官考察意图

考察候选人对 Go 最常用数据结构的底层理解深度。
初级只能说出"slice 是数组包装，map 是哈希表"，高级要能讲清楚**slice 扩容的具体倍数与容量计算、map 的 Go 1.24 Swiss Table 改进、哪些操作是并发不安全的、以及如何正确使用**，结合生产中的性能问题给出分析。

---

## 核心答案（30 秒版）

| 结构 | 底层实现 | 关键特性 |
|------|----------|----------|
| **slice** | 三个字段：ptr + len + cap，扩容时 2× 增长（大量元素时降为 1.25×）| 引用语义，非指针数组 |
| **map** | Go 1.24 前：runtime.hmap + bucket；Go 1.24+：**Swiss Table**，O(1) 均摊，无链表 | **读写不并发安全**，需要 sync.RWMutex 或 sync.Map |

**Swiss Table（Go 1.24+）**：Go 1.24 将 map 底层从传统的桶式哈希（bucket 链表）改为 Swiss Table，查找时通过 SIMD 指令批量探测，性能提升 30%+，内存局部性更好。

---

## 深度展开

### 1. slice 底层结构

```go
// 伪代码：slice header
type slice struct {
    array unsafe.Pointer  // 指向底层数组的指针
    len   int             // 当前长度
    cap   int             // 容量（底层数组大小）
}
```

```
slice 结构：
┌─────────────────────┐
│  ptr ───────────────┼──→ [slot0][slot1][slot2]...[slotN]
│  len = 3            │              ↑
│  cap = 8            │         已用，已分配
└─────────────────────┘
```

**关键特性**：
- slice 本身是 24 字节的"视图"，底层数组在堆（或栈，Go 1.25/1.26）
- 拷贝 slice 变量只复制 3 个字段，不复制底层数组
- `append` 超过 cap 时触发扩容，分配新数组并复制

### 2. slice 扩容策略

```go
// src/runtime/slice.go（核心逻辑）
func growslice(et *_type, old slice, cap int) slice {
    newcap := old.cap
    doublecap := old.cap * 2

    if cap > doublecap {
        newcap = cap  // 需要容量 > 2× 旧容量：新容量 = 所需容量
    } else {
        if old.len < 1024 {
            newcap = doublecap  // 长度 < 1024：翻倍增长
        } else {
            // 长度 >= 1024：增长 1.25×
            newcap = old.cap + old.cap*3/4
        }
    }

    // 分配新底层数组
    var overflow bool
    var lenmem, capmem uintptr
    newcap, overflow = growcap(et, old.cap, newcap)
    capmem = roundupsize(uintptr(newcap) * et.size) // 按 size class 对齐
    newarray := mallocgc(capmem, et, true)

    // 复制旧数据
    memmove(newarray, old.array, lenmem)

    return slice{newarray, old.len, newcap}
}
```

**容量增长规律**：

| 旧容量 | 新容量（cap < 1024）| 新容量（cap ≥ 1024）|
|--------|---------------------|---------------------|
| 1 | 2 | 2 |
| 2 | 4 | 4 |
| 8 | 16 | 16 |
| 1024 | 2048 | 1280 (= 1024 + 1024×3/4) |
| 2048 | 4096 | 2560 (= 2048 + 2048×3/4) |

**扩容后的内存布局**：

```
旧 slice：cap=4
[ A ][ B ][ C ][ D ]
        ↑
     append(E) → 扩容

新 slice：cap=8
[ A ][ B ][ C ][ D ][ E ][ _ ][ _ ][ _ ]
        ↑           ↑
     复制到这里   新增元素
```

### 3. slice 常见坑

#### 坑 1：append 返回值被忽略

```go
// 错误：s 的底层数组不变
s := make([]int, 0, 10)
append(s, 1, 2, 3)
fmt.Println(len(s)) // 输出 0！

// 正确：必须用返回值
s = append(s, 1, 2, 3)
fmt.Println(len(s)) // 输出 3
```

#### 坑 2：slice 引用了意外的大底层数组

```go
// 创建 10MB 数组
buf := make([]byte, 10*1024*1024)

// 只用了前 100 字节，但整个 10MB 都在引用中，无法 GC
smallBuf := buf[:100]
// 即使 smallBuf = nil，10MB 数组仍无法回收！
```

#### 坑 3：for-range 的拷贝行为

```go
// 切片值拷贝，非元素引用
s := []int{1, 2, 3}
for _, v := range s {
    v *= 2 // 修改副本，原 slice 不变
}
// s 仍是 [1, 2, 3]

// 正确修改方式
for i := range s {
    s[i] *= 2
}
// s 变为 [2, 4, 6]
```

### 4. map 底层结构（Go 1.24 之前）

```go
// src/runtime/map.go
type hmap struct {
    count      int          // 元素数量
    flags      uint8
    B          uint8        // bucket 数量 = 2^B
    nbuckets   uintptr      // bucket 数组大小
    key斗      *_type       // key 类型
    value斗    *_type       // value 类型
    hash0      uintptr      // 哈希种子
    buckets    unsafe.Pointer // bucket 数组指针
    oldbuckets unsafe.Pointer // 扩容时的旧 bucket
    nevacuate  uintptr
    overflow    []unsafe.Pointer // 溢出 bucket 链表
}

// bucket 结构（每个 bucket 8 个 key-value 对）
type bmap struct {
    tophash [8]uint8  // 每个 key 的哈希高 8 位（快速判断）
    keys    [8]keytype
    values  [8]valuetype
    overflow unsafe.Pointer // 溢出 bucket 指针
}
```

```
map 底层结构（bucket 数组）：
┌─────────────────────────────────────────────────┐
│ hmap.buckets ──→ [Bucket0][Bucket1][...][BucketN]│
│                   │                             │
│                   └→ [Overflow Bucket] ──→ ...  │
└─────────────────────────────────────────────────┘

每个 Bucket：
┌────────────┬────────┬────────┬───────────────────┐
│ tophash[8]│ keys[8]│ vals[8]│ overflow pointer │
└────────────┴────────┴────────┴───────────────────┘
```

**查找流程**：
1. 计算 `hash = hash(key) % 2^B`
2. 取 `tophash = hash >> 56`（高 8 位）
3. 定位到 bucket，遍历 8 个 slot，匹配 tophash + key

### 5. Go 1.24 Swiss Table：革命性改进

Go 1.24 将 map 实现从**桶式哈希**切换为 **Swiss Table**，这是 Go 历史上最大的 map 实现变更。

#### 传统桶式 vs Swiss Table

| 维度 | 桶式哈希（Go 1.23 及之前）| Swiss Table（Go 1.24+）|
|------|-------------------------|-------------------------|
| 探测方式 | 线性探测 + overflow 链表 | 开放地址 + 群组探测（group probing）|
| 冲突处理 | 溢出 bucket 链表 | 更密集的开放地址探测 |
| 内存局部性 | 差（溢出链表跳转）| 好（所有数据在连续内存）|
| 查找性能 | O(1) 均摊，但有链表跳转 | O(1) 均摊，无链表跳转 |
| 性能特征 | 更稳定的下界 | 更高的平均值，接近最优 |
| SIMD 支持 | 否 | 是（批量探测）|

#### Swiss Table 原理

```
传统哈希：
  key1 → bucket[3] → [满] → overflow → [满] → ...
  key2 → bucket[3] → overflow 链表深处！

Swiss Table（群组探测）：
  每个 bucket 是一组 16 个 slot：
  ┌────────────────────────────────────────┐
  │ Group: [  ][  ][  ][  ][  ][  ][  ][  ] │ ← 8个实际slot
  │        [  ][  ][  ][  ][  ][  ][  ][  ] │ ← 8个影子slot（for SIMD）
  └────────────────────────────────────────┘
  哈希后得到 group base，SIMD 同时探测 16 个 slot
```

```go
// Swiss Table 核心思想
// 用 SIMD 指令一次比较 16 个 slot 的 tophash
// 比链表探测快得多，缓存命中率更高
```

#### 生产影响

```go
// Go 1.24+ map 操作自动受益
m := make(map[string]int, 1000000)

// 查找、插入、删除性能提升约 30%
for i := 0; i < 1000000; i++ {
    m[fmt.Sprintf("key%d", i)] = i
}
```

**注意**：Swiss Table 的改进对使用者完全透明，无需修改任何代码。

### 6. map 扩容

map 也会扩容，但与 slice 不同：

```go
// map 扩容触发条件
// 当 count > B * 6.5（负载因子过高）时，翻倍 bucket 数组

// 与 slice 不同的点：
// 1. map 扩容是渐进式的，每次最多迁移一部分 bucket（旧 bucket → 新 bucket）
// 2. 迁移期间会同时查询新旧两个 bucket
// 3. 不会一次性复制所有数据，避免长时间 STW
```

### 7. map 并发安全

**map 读写并发不安全！**

```go
m := make(map[string]int)

// 并发写入 → 致命错误
go func() { m["a"] = 1 }()
go func() { m["b"] = 2 }()
// fatal error: concurrent map writes
```

**解决方案**：

```go
// 方案 1：sync.RWMutex
var mu sync.RWMutex
m := make(map[string]int)

go func() {
    mu.Lock()
    m["a"] = 1
    mu.Unlock()
}()
go func() {
    mu.RLock()
    _ = m["b"]
    mu.RUnlock()
}()

// 方案 2：sync.Map（适合读多写少场景）
var sm sync.Map
sm.Store("a", 1)
sm.Load("a") // 返回 (value, true)

// 方案 3：分片 map（极高并发）
shards := make([]map[string]int, 256)
for i := range shards {
    shards[i] = make(map[string]int)
}
shard := shards[int(hash(key)) % 256]
shard[key] = value
```

**sync.Map vs RWMutex+map**：

| 场景 | 推荐方案 |
|------|----------|
| 读多写少（read-heavy）| `sync.Map`，无锁读取 |
| 写多读多 | `RWMutex` + `map` |
| 极高并发 | 分片 `map`（`~shard[-hash(key)%N]`) |
| 需要按序遍历 | `RWMutex` + `map`（`sync.Map` 遍历无序）|

### 8. slice 与 map 的组合陷阱

```go
// 常见错误：map 的 value 是 slice，需要单独初始化
m := make(map[string][]int)

// m["a"] = append(m["a"], 1) // 错误！m["a"] 是 nil slice，append 会 panic

// 正确做法：
m["a"] = append(m["a"], 1) // 自动初始化（nil slice append 在 Go 1.22+ 可用）

// 或者更安全的写法：
if m["a"] == nil {
    m["a"] = []int{}
}
m["a"] = append(m["a"], 1)
```

---

## 高频追问

**Q：slice 扩容是 2× 还是 1.25×？**
> 初始阶段（cap < 1024）使用 2× 增长；cap ≥ 1024 后使用 1.25× 增长。这是为了减少大 slice 的内存浪费。

**Q：Go 1.24 的 Swiss Table 性能提升多少？**
> 官方 benchmark 显示平均 30% 提升，主要来自更好的缓存局部性和 SIMD 批量探测。实际提升取决于 key/value 大小和哈希函数性能。

**Q：为什么 Go map 不能用 value type 直接修改？**
> map 返回的是 value 的拷贝，不是引用：
> ```go
> m := map[string]User{"a": {Name: "bin"}}
> m["a"].Name = "bin2" // 编译错误：cannot assign to struct field
> ```
> 必须整体替换：`m["a"] = User{Name: "bin2"}` 或使用指针 `m["a"].(ptr).Name`

**Q：slice 传函数时会发生什么？**
> slice 按值传递，但 slice 本身只包含 3 个字段（ptr, len, cap），拷贝成本极低。**但**函数内部对 slice 的修改（如 `s = append(s, x)`）不会影响调用者，除非 `append` 触发了扩容返回了新 slice 并赋值给 `s`。

**Q：如何安全地并发读写 slice？**
> 没有并发安全的 slice。解决方案：RWMutex 保护、或使用 `sync.Pool` 复用 slice、或在写入前复制（`newSlice := append(slice{}, oldSlice...)`）

---

## 总结

| 知识点 | 核心结论 |
|--------|----------|
| slice 扩容 | cap < 1024 时 2×，≥ 1024 时 1.25× |
| append 行为 | 超过 cap 才扩容，否则原地修改 len |
| map 底层 | Go 1.24 前：bucket + overflow 链表；Go 1.24+：Swiss Table + SIMD |
| Swiss Table | 批量群组探测，无 overflow 链表，缓存友好，~30% 性能提升 |
| map 并发 | **绝对禁止并发读写**，使用 RWMutex 或 sync.Map |
| map vs slice 性能 | slice 用于有序/连续访问，map 用于键值随机访问 |

---

> 📚 相关内容：[逃逸分析](./04-escape.md) · [Go 内存模型](./03-memory-model.md) · [sync 原语](../02-concurrency/02-sync.md)
