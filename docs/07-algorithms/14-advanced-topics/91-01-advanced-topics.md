# 进阶高频题：LFU / 前缀树 / 最短路 / 海量数据 / 手写排序

> 考察频率：★★★★☆  优先级：P2  语言：Go（核心题附 Python 对照）
> 题型识别：设计类数据结构、字符串匹配、单源最短路、海量数据分治、分治排序与选择
> 说明：本篇补齐 07 模块此前未覆盖的 9 道大厂高频题。LRU 见 [05-linked-list/43-05-linked-list.md](../05-linked-list/43-05-linked-list.md)，TopK 见 [08-heap/45-08-heap.md](../08-heap/45-08-heap.md)。

---

## 【1】🔥🔥🔥🔥🔥 手写 LFU 缓存（LeetCode 460）

### 面试官考察意图

LRU 太常见，很多大厂会直接加码考 LFU。核心不是"能不能写出来"，而是你能不能把**淘汰规则、复杂度、边界**三件事说清楚：
- LRU 淘汰"最久未使用"，LFU 淘汰"**使用频率最低**"；频率相同再淘汰"最久未使用"（LRU 兜底）。
- 要求 get / put 都是 **O(1)**。

### 核心答案（30 秒版）

- 数据结构：`key → node` 哈希表 + `freq → 双向链表` 哈希表 + `minFreq` 变量。
- 每个 node 记录自己的 `freq`；同 freq 的节点挂在同一条链表里，**链表头部最新、尾部最旧**。
- 访问/更新一个 key：从旧 freq 链表摘下，挂到 `freq+1` 链表头部；若旧链表空了且 `minFreq == 旧freq`，则 `minFreq++`。
- 淘汰：取 `freqs[minFreq]` 的**尾节点**删除；插入新 key 后 `minFreq = 1`。
- 关键点：因为每次访问频率只 +1，且新 key 频率恒为 1，所以 `minFreq` 只需"清空时 +1 / 插入时置 1"，**不需要遍历找最小值**，这才保证 O(1)。

### Go 代码

```go
type lfuNode struct {
    key, val, freq int
    prev, next     *lfuNode
}

// 双向链表：head 侧最新，tail 侧最旧
type dll struct {
    head, tail *lfuNode
    size       int
}

func newDLL() *dll {
    h, t := &lfuNode{}, &lfuNode{}
    h.next, t.prev = t, h
    return &dll{head: h, tail: t}
}

func (d *dll) addFront(n *lfuNode) {
    n.prev, n.next = d.head, d.head.next
    d.head.next.prev = n
    d.head.next = n
    d.size++
}

func (d *dll) remove(n *lfuNode) {
    n.prev.next = n.next
    n.next.prev = n.prev
    d.size--
}

func (d *dll) removeLast() *lfuNode {
    if d.size == 0 {
        return nil
    }
    n := d.tail.prev
    d.remove(n)
    return n
}

type LFUCache struct {
    cap     int
    minFreq int
    nodes   map[int]*lfuNode // key -> node
    freqs   map[int]*dll     // freq -> 同频链表
}

func NewLFUCache(capacity int) *LFUCache {
    return &LFUCache{cap: capacity, nodes: make(map[int]*lfuNode), freqs: make(map[int]*dll)}
}

func (c *LFUCache) touch(n *lfuNode) {
    old := n.freq
    c.freqs[old].remove(n)
    if c.freqs[old].size == 0 {
        delete(c.freqs, old)
        if c.minFreq == old {
            c.minFreq++ // 原最小频链表已空，下一个必然存在的是 old+1
        }
    }
    n.freq++
    d := c.freqs[n.freq]
    if d == nil {
        d = newDLL()
        c.freqs[n.freq] = d
    }
    d.addFront(n)
}

func (c *LFUCache) Get(key int) int {
    n, ok := c.nodes[key]
    if !ok {
        return -1
    }
    c.touch(n)
    return n.val
}

func (c *LFUCache) Put(key, value int) {
    if c.cap == 0 {
        return
    }
    if n, ok := c.nodes[key]; ok {
        n.val = value
        c.touch(n)
        return
    }
    if len(c.nodes) == c.cap {
        victim := c.freqs[c.minFreq].removeLast() // 同频中最久未使用
        delete(c.nodes, victim.key)
        if c.freqs[c.minFreq].size == 0 {
            delete(c.freqs, c.minFreq)
        }
    }
    n := &lfuNode{key: key, val: value, freq: 1}
    c.nodes[key] = n
    if c.freqs[1] == nil {
        c.freqs[1] = newDLL()
    }
    c.freqs[1].addFront(n)
    c.minFreq = 1
}
```

### 复杂度

| 操作 | 时间 | 空间 |
|------|------|------|
| Get / Put | O(1) | O(capacity) |

### 高频追问

**Q：为什么 `minFreq` 不需要重新扫描？**
> 因为频率是**连续递增**的：访问一次只 +1，新元素频率恒为 1。所以最小频率要么是 1（刚插入过），要么在最小频链表被清空时恰好 +1。这依赖"每次只 +1"这一前提，如果支持批量提频就不能这么做了。

**Q：LFU 缓存污染怎么解决？**
> LFU 的经典缺陷是"历史热点"长期霸占缓存（比如昨天的爆款文章频率极高，今天已无人访问仍在缓存里）。工业界用 **LFU-Aging / TinyLFU**：周期性对所有计数右移一位（减半衰减），或用滑动窗口（Count-Min Sketch + 窗口重置）。Guava Cache 的 W-TinyLFU、Redis 的 `allkeys-lfu`（8bit 概率计数 + 衰减）都是这个思路。

**Q：和 LRU 相比谁更好？**
> 没有绝对优劣。LFU 命中率在"访问分布稳定"的负载下更高，但对突发新热点响应差、实现与内存成本高；LRU 简单、对突发友好，但容易被一次性的全表扫描污染（可用 LRU-K / 2Q 缓解）。Redis 4.0 起两种策略都可选。

### 🗣️ 面试话术

"LFU = 哈希表 + 频率分桶双向链表 + minFreq 指针，因为频率每次只 +1，minFreq 不用扫描就能维护，get/put 都是 O(1)；同频内部用链表尾淘汰最旧的，实现 LRU 兜底。"

---

## 【2】🔥🔥🔥🔥🔥 前缀树 Trie：实现与通配符 / 敏感词过滤（LeetCode 208 / 212）

### 面试官考察意图

Trie 是"字符串前缀"场景的标配：搜索框联想、敏感词过滤、IP 路由表（压缩 Trie）、自动补全。面试常从"实现 Trie"开始，追问"前缀匹配""通配符 `.`""海量敏感词怎么过滤"。

### 核心答案（30 秒版）

- Trie 每个节点持有 `children`（数组或 map）+ `isEnd` 标记。
- 插入/查询都是**逐字符下沉**，时间 O(L)（L 为串长），与词典规模无关——这是它相对哈希表的唯一优势。
- 空间换时间：数组版 `[26]*Trie` 定长好写但浪费（26 指针 × 8B = 208B/节点）；中文/大字符集、稀疏场景用 `map[byte]*Trie`。
- 海量敏感词：把词典建成 Trie，文本做**多模式匹配**时配合 AC 自动机（在 Trie 上补 fail 指针，把复杂度从 O(n·L) 降到 O(n)）。

### Go 代码

```go
type Trie struct {
    children [26]*Trie
    isEnd    bool
}

func NewTrie() *Trie { return &Trie{} }

func (t *Trie) Insert(word string) {
    node := t
    for i := 0; i < len(word); i++ {
        c := word[i] - 'a'
        if node.children[c] == nil {
            node.children[c] = &Trie{}
        }
        node = node.children[c]
    }
    node.isEnd = true
}

func (t *Trie) searchPrefix(s string) *Trie {
    node := t
    for i := 0; i < len(s); i++ {
        c := s[i] - 'a'
        if node.children[c] == nil {
            return nil
        }
        node = node.children[c]
    }
    return node
}

func (t *Trie) Search(word string) bool {
    n := t.searchPrefix(word)
    return n != nil && n.isEnd
}

func (t *Trie) StartsWith(prefix string) bool { return t.searchPrefix(prefix) != nil }
```

**追问变体：支持 `.` 通配符（LeetCode 211，DFS 回溯）**

```go
func (t *Trie) SearchWildcard(word string) bool {
    var dfs func(node *Trie, i int) bool
    dfs = func(node *Trie, i int) bool {
        if node == nil {
            return false
        }
        if i == len(word) {
            return node.isEnd
        }
        if word[i] == '.' {
            for _, ch := range node.children {
                if dfs(ch, i+1) {
                    return true
                }
            }
            return false
        }
        return dfs(node.children[word[i]-'a'], i+1)
    }
    return dfs(t, 0)
}
```

### 敏感词过滤实战（Go 版简化 AC 思想）

```go
// 在 Trie 上做朴素多模式匹配：外循环文本，内循环从每个位置沿 Trie 走 —— O(n*L)
// 生产环境（百万级词库）应使用 AC 自动机，把 fail 指针预计算好，扫描一遍文本即可命中所有模式串
func (t *Trie) Match(text string, hit func(string)) {
    for i := 0; i < len(text); i++ {
        node := t
        for j := i; j < len(text); j++ {
            node = node.children[text[j]-'a']
            if node == nil {
                break
            }
            if node.isEnd {
                hit(text[i : j+1])
            }
        }
    }
}
```

### 高频追问

**Q：Trie vs 哈希表，什么时候用 Trie？**
> 只做"整串精确查询"用哈希表（O(1) 且实现简单）。只要涉及**前缀/范围/通配**语义（联想词、前缀统计、最长前缀匹配、通配符），哈希表做不到有序性，必须用 Trie。代价是内存与缓存不友好（指针跳转随机访问）。

**Q：Trie 的内存怎么优化？**
> ① 双数组 Trie（Double-Array，两个 int 数组表达转移，极致紧凑、查询快，用于词典/分词）；② 压缩 Trie / Radix Tree（把单链路径合并成一条边，Linux 路由表、Redis 集群 slot 都用）；③ 用 map 只存实际出现的边，避免 26 空洞。

### 🗣️ 面试话术

"Trie 的本质是用共享前缀换查询稳定 O(L)，凡是带前缀语义的查询哈希表都做不了。敏感词过滤要在 Trie 上加 fail 指针变成 AC 自动机，这样文本只需扫一遍。"

---

## 【3】🔥🔥🔥🔥 海量数据 TopK / 去重：内存放不下怎么办（40 亿整数）

### 面试官考察意图

这是**后端岗与纯算法岗的分水岭题**：给 40 亿个 unsigned int（约 16 GB），内存只有 1 GB，找出现次数最多的 TopK / 找只出现一次的数 / 找出现两次的数。考的是"分治 + 位图 + 堆"的组合，以及能不能算出内存量级。

### 核心答案（30 秒版）

**通用公式：哈希分桶 → 桶内小顶堆 / 位图 → 合并**

1. **分治（Hash 分桶）**：`bucket = hash(x) % 1024`，把 16 GB 大文件拆成 1024 个小文件（每个约 16 MB）。**同一个数一定落在同一个桶**，所以桶内统计是完备的。
2. **桶内 TopK**：用 `map[int]int` 词频统计，再用**大小为 K 的小顶堆**求 TopK，复杂度 O(m log K)（m 为桶内元素数）。
3. **合并**：1024 个桶各自 TopK 再合并，最多 1024×K 个候选，一次堆排即可。

| 需求 | 方案 | 内存量级 |
|------|------|----------|
| TopK 高频 | Hash 分桶 + 每桶小顶堆 | 单桶 < 100 MB |
| 找只出现一次 | 分桶 + 桶内位图（2 bit/数） | 2 × 2³² bit = 1 GB ← 刚好卡线，需分桶降到 1 MB/桶 |
| 找出现两次 | 分桶 + 2 bit 计数（00 未出现 / 01 一次 / 10 多次） | 同上 |

**2-bit 位图计数（出现两次的数）**

```go
// 40 亿个数 ≈ 2^32 个可能值，用 2 bit 记录状态
// 内存 = 2^32 * 2 bit / 8 = 1 GB；分 1024 桶后每桶仅 1 MB
const totalBits = 1 << 32

type Bitmap2 struct {
    bits []uint64 // 每 32 个值占 1 个 uint64（64 bit / 2 bit）
}

func (b *Bitmap2) get(v uint32) uint8 {
    idx, off := v/32, (v%32)*2
    return uint8((b.bits[idx] >> off) & 0b11)
}

func (b *Bitmap2) inc(v uint32) {
    idx, off := v/32, (v%32)*2
    cur := b.get(v)
    if cur < 3 { // 3 表示"多次"，饱和不再累加
        b.bits[idx] &^= 0b11 << off
        b.bits[idx] |= uint64(cur+1) << off
    }
}
```

**TopK 小顶堆（Go 标准库实现）**

```go
type MinHeap []int

func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x any)         { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() any {
    old := *h
    n := len(old)
    v := old[n-1]
    *h = old[:n-1]
    return v
}

// 求最大的 K 个数：维护大小为 K 的小顶堆，堆顶就是当前第 K 大
func topK(nums []int, k int) []int {
    h := &MinHeap{}
    heap.Init(h)
    for _, v := range nums {
        if h.Len() < k {
            heap.Push(h, v)
        } else if v > (*h)[0] {
            (*h)[0] = v
            heap.Fix(h, 0) // 替换堆顶后下沉，O(log K)
        }
    }
    return *h
}
```

### 高频追问

**Q：为什么不直接用 `map[int]int` 统计？**
> 40 亿个 key 的哈希表内存开销是"key + value + 桶指针 + 负载因子冗余"，通常 20~50 字节/条，量级是 80~200 GB，远超内存。位图把"值域"当索引，2 bit/值才 1 GB，这就是**空间换法的选择取决于数据特征**（值域小 → 位图；值域大 → 哈希分桶）。

**Q：分桶数怎么定？**
> 让单桶可被内存容纳并留出哈希表开销余量。比如 1 GB 内存，目标单桶 ≤ 50 MB，则桶数 ≥ 16 GB / 50 MB ≈ 328，取 2 的幂 512 或 1024（取模可用位运算，且便于并行多机器处理）。

**Q：怎么判断数据倾斜（某个桶特别大）？**
> 好的哈希函数保证基本均匀；若业务 key 本身倾斜，可在 hash 中加盐（`hash(x + salt)`）或改用范围分桶。工程上要监控各桶大小，桶大小超阈值就二次分桶。

### 🗣️ 面试话术

"内存放不下就分治：Hash 分桶保证同 key 同桶，桶内再用堆或位图统计，最后合并。值域小就用位图（2 bit 计数），值域大就用哈希分桶 + 小顶堆，先算清楚量级再动手。"

---

## 【4】🔥🔥🔥🔥 排序链表（LeetCode 148）：归并排序 O(1) 空间

### 面试官考察意图

"快排/归并原理"人人会背，但**链表场景下快排为何不适用、归并怎么做到 O(1) 额外空间**能筛掉一大批人。考点：链表找中点（快慢指针）、归并的哨兵节点、自底向上迭代合并。

### 核心答案（30 秒版）

- 链表的痛点是**不支持随机访问**：快排需要 `partition` 前后指针任意跳，链表里退化严重；堆排需要下标索引，更不行。→ 链表排序首选**归并**。
- 递归版：快慢指针找中点 → 断开 → 递归排序两边 → 合并。时间 O(n log n)，**空间 O(log n)**（递归栈）。
- 要 O(1) 空间就写**自底向上迭代版**：先两两合并（step=1），再四四合并（step=2）……直到 step ≥ n。
- 边界三件套：空链表/单节点直接返回；`dummy` 哨兵简化头节点处理；合并完记得把 `cur` 推进到尾部。

### Go 代码（自底向上，O(1) 额外空间）

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

func sortList(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    n := 0
    for p := head; p != nil; p = p.Next {
        n++
    }

    dummy := &ListNode{Next: head}
    for step := 1; step < n; step <<= 1 {
        prev, cur := dummy, dummy.Next
        for cur != nil {
            left := cur
            right := split(left, step) // 左段长度 step
            cur = split(right, step)   // 右段长度 step，返回值是下一轮的起点
            prev = merge(prev, left, right)
        }
    }
    return dummy.Next
}

// 从 head 起保留 step 个节点，返回剩余部分的头
func split(head *ListNode, step int) *ListNode {
    if head == nil {
        return nil
    }
    for i := 1; i < step && head.Next != nil; i++ {
        head = head.Next
    }
    next := head.Next
    head.Next = nil
    return next
}

// 把 l1、l2 归并接到 prev 后面，返回新的尾部
func merge(prev, l1, l2 *ListNode) *ListNode {
    cur := prev
    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            cur.Next, l1 = l1, l1.Next
        } else {
            cur.Next, l2 = l2, l2.Next
        }
        cur = cur.Next
    }
    if l1 != nil {
        cur.Next = l1
    } else {
        cur.Next = l2
    }
    for cur.Next != nil {
        cur = cur.Next
    }
    return cur
}
```

### 高频追问

**Q：递归版和迭代版怎么选？**
> 时间都是 O(n log n)。递归版好写好讲，空间 O(log n)；迭代版空间 O(1)，但指针操作容易写错。面试建议先讲递归思路（显逻辑），再给迭代版本（显工程素养），并说明"链表长度 10⁵ 时递归栈不会溢出，但生产代码里 O(1) 更稳"。

**Q：稳定吗？**
> 归并是稳定排序（代码里用 `l1.Val <= l2.Val` 保证相等时取左段）。链表节点重排只改指针不搬迁数据，这也是它相对数组排序的优势。

### 🗣️ 面试话术

"链表不支持随机访问，所以快排/堆排都不合适，首选归并；要 O(1) 空间就写成自底向上，把 1、2、4、8 的步长一路合并上去。"

---

## 【5】🔥🔥🔥🔥 数组中第 K 大的元素：快速选择 vs 小顶堆（LeetCode 215）

### 面试官考察意图

同一道题有三种解法（排序 / 小顶堆 / 快速选择），面试官想看你能**按数据规模和 K 的大小做选型**，以及会不会随机化避免 O(n²) 退化。

### 核心答案（30 秒版）

| 方案 | 时间 | 空间 | 适用 |
|------|------|------|------|
| 全排序 | O(n log n) | O(log n) | 写起来最快，面试能过但拿不到加分 |
| 小顶堆（大小 K） | O(n log K) | O(K) | **数据流 / K 远小于 n / 内存受限** |
| 快速选择（Quickselect） | 平均 O(n)，最坏 O(n²) | O(1) | 静态数组，追求平均最优 |

- 第 K 大 = 升序数组下标 `n-K` 处。快速选择只递归**一侧**：`T(n) = T(n/2) + O(n) = O(n)`。
- **必须随机化 pivot**，否则面对有序输入每次划分极不均衡，退化成 O(n²)（这也是快排的经典退化点）。

### Go 代码

```go
import "math/rand/v2"

func findKthLargest(nums []int, k int) int {
    target := len(nums) - k // 升序第 target 下标
    lo, hi := 0, len(nums)-1
    for lo <= hi {
        p := partition(nums, lo, hi)
        switch {
        case p == target:
            return nums[p]
        case p < target:
            lo = p + 1
        default:
            hi = p - 1
        }
    }
    return -1
}

// 三路划分思路可换成二分，这里用最经典的 Lomuto 单路 + 随机 pivot
func partition(a []int, lo, hi int) int {
    r := lo + rand.IntN(hi-lo+1)
    a[r], a[hi] = a[hi], a[r]
    pivot := a[hi]
    i := lo
    for j := lo; j < hi; j++ {
        if a[j] < pivot {
            a[i], a[j] = a[j], a[i]
            i++
        }
    }
    a[i], a[hi] = a[hi], a[i]
    return i
}

// 变体：数据流中第 K 大 —— 维护大小为 K 的小顶堆
// Push：入堆；堆满且新值 > 堆顶则替换。TopK = 堆顶。
```

### 高频追问

**Q：为什么快速选择是 O(n)？**
> 快排是 `T(n) = 2T(n/2) + O(n) = O(n log n)`，因为两侧都要排。快速选择只处理包含目标下标的一侧，递推式变成 `T(n) = T(n/2) + O(n)`，等比级数求和收敛到 O(n)。注意是**平均**复杂度，最坏 O(n²)。

**Q：大数组里有大量重复元素怎么办？**
> 用**三路划分**（`< pivot` / `= pivot` / `> pivot`）。如果目标下标落在"等于区"里，直接返回 pivot，避免许多等于 pivot 的元素被反复划分——这也是 LeetCode 215 卡测试用例的常见写法。

**Q：K 很小（比如 K=10，n=1亿）选哪个？**
> 小顶堆 O(n log K) ≈ 1亿 × 3.3 次比较，且空间 O(K)；快速选择平均 O(n) 但最坏退化风险与递归/迭代跳转多。工程上更推荐**堆**：稳定、可流式、内存可控。

### 🗣️ 面试话术

"三种解法我都会，选型看场景：静态数组追求平均性能用随机化快速选择 O(n)；数据流或 K 很小用大小为 K 的小顶堆 O(n log K)。关键是要说清快速选择为什么平均 O(n) 以及为什么要随机 pivot。"

---

## 【6】🔥🔥🔥🔥 单源最短路：Dijkstra 与网络延迟时间（LeetCode 743）

### 面试官考察意图

07 模块此前只覆盖 BFS/DFS/并查集，**带权图最短路**是实打实的缺口。Dijkstra 是后端场景（RPC 路由、拓扑感知调度、限流链路选路、地图导航）的高频考点。要能说清：为什么不能有负权边、堆优化后复杂度、与 BFS/Bellman-Ford 的关系。

### 核心答案（30 秒版）

- Dijkstra 本质是**贪心 + BFS 的带权推广**：每次从未确定的点里取**当前距离最小**的点，用它松弛邻居；一旦确定就不再更新。
- 朴素实现 O(V²)（邻接矩阵）；**小顶堆优化 O((V+E) log V)**（邻接表）。
- **不能处理负权边**：贪心前提是"距离最小的点不可能再被更短路径更新"，负权边会打破这个假设 → 用 Bellman-Ford O(VE) 或 SPFA。
- 堆优化两个必备细节：① 用**懒删除**（弹出时比较 `d > dist[to]` 就跳过），不必实现 decrease-key；② 只有真正变小时才入堆，控制堆规模。

### Go 代码

```go
type edge struct{ to, w int }

type item struct {
    to, d int
}

type minHeap []item

func (h minHeap) Len() int            { return len(h) }
func (h minHeap) Less(i, j int) bool  { return h[i].d < h[j].d }
func (h minHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *minHeap) Push(x any)         { *h = append(*h, x.(item)) }
func (h *minHeap) Pop() any {
    old := *h
    n := len(old)
    v := old[n-1]
    *h = old[:n-1]
    return v
}

const inf = 1 << 30

// LeetCode 743：从 k 出发，所有节点都收到信号的最短时间
func networkDelayTime(times [][]int, n int, k int) int {
    g := make([][]edge, n+1)
    for _, t := range times {
        g[t[0]] = append(g[t[0]], edge{to: t[1], w: t[2]})
    }

    dist := make([]int, n+1)
    for i := range dist {
        dist[i] = inf
    }
    dist[k] = 0

    h := &minHeap{{to: k, d: 0}}
    heap.Init(h)

    for h.Len() > 0 {
        cur := heap.Pop(h).(item)
        if cur.d > dist[cur.to] {
            continue // 过期堆元素，懒删除
        }
        for _, nx := range g[cur.to] {
            if nd := cur.d + nx.w; nd < dist[nx.to] {
                dist[nx.to] = nd
                heap.Push(h, item{to: nx.to, d: nd})
            }
        }
    }

    ans := 0
    for i := 1; i <= n; i++ {
        if dist[i] == inf {
            return -1 // 有节点不可达
        }
        if dist[i] > ans {
            ans = dist[i]
        }
    }
    return ans
}
```

### 高频追问

**Q：复杂度到底是多少？**
> 堆优化 O((V+E) log V)：每条边最多触发一次入堆（O(log V)），每个点出堆一次。稀疏图（E ≈ V）接近 O(V log V)，稠密图（E ≈ V²）退化到 O(V² log V)，此时朴素 O(V²) 反而更好。

**Q：和 BFS、A* 的关系？**
> 边权全为 1 时 Dijkstra 就退化成 BFS（队列换成优先队列、比较值恒等）。A* 是 Dijkstra 加启发函数 `f = g + h`，只往目标方向扩展；h 必须可采纳（不高估真实距离）才保证最优。

**Q：多源 / 全源最短路怎么办？**
> 多源（多个起点同时扩散）：初始化时把所有源点都塞进堆，dist=0，常用于"离最近的医院"类问题。全源：Floyd O(V³) 适用于 V ≤ 400 的稠密小图；稀疏图跑 V 次 Dijkstra（O(V·E log V)）更划算。

### 🗣️ 面试话术

"Dijkstra 就是带权的 BFS，用优先队列每次取当前最近的未确定点做松弛，堆优化是 O((V+E) log V)。有负权边必须换 Bellman-Ford，因为贪心的正确性前提被打破了。"

---

## 【7】🔥🔥🔥🔥 二叉树的序列化与反序列化（LeetCode 297）

### 面试官考察意图

这是"设计 + 递归 + 边界"的综合题，也是真实工程问题（RPC 传输树结构、配置快照、缓存序列化）。核心难点不在递归，而在**如何让空节点可表达**、**分隔符怎么选**、**能不能做到流式解析**。

### 核心答案（30 秒版）

- 单靠"前序/层序"遍历**无法唯一还原**二叉树，必须把 `null` 也写进序列：用 `#` 占位。
- 前序 + `#` 占位是最短实现：序列化 O(n)，反序列化也 O(n)。
- 关键细节：
  1. **分隔符要选不会出现在值里的字符**（如 `,`），值用 `strconv.Itoa` / `strconv.Quote` 保证转义。
  2. 反序列化用**共享游标**（或闭包捕获的 index），前序天然"先根、后左、再右"，递归一次就能还原。
  3. 字符串拼接用 `strings.Builder`（避免 O(n²) 的 `+=` 拷贝）。
- 层序（BFS）版本同样可行，对"按层传输/流式还原"更友好；但要注意省略末尾连续的 `#`。

### Go 代码（前序遍历 + 占位符）

```go
type TreeNode struct {
    Val         int
    Left, Right *TreeNode
}

type Codec struct{}

func (c *Codec) serialize(root *TreeNode) string {
    var sb strings.Builder
    var dfs func(n *TreeNode)
    dfs = func(n *TreeNode) {
        if n == nil {
            sb.WriteString("#,") // 空节点必须占位，否则无法唯一还原
            return
        }
        sb.WriteString(strconv.Itoa(n.Val))
        sb.WriteByte(',')
        dfs(n.Left)
        dfs(n.Right)
    }
    dfs(root)
    return sb.String()
}

func (c *Codec) deserialize(data string) *TreeNode {
    parts := strings.Split(data, ",")
    i := 0
    var dfs func() *TreeNode
    dfs = func() *TreeNode {
        if i >= len(parts) {
            return nil
        }
        token := parts[i]
        i++
        if token == "" || token == "#" {
            return nil
        }
        v, err := strconv.Atoi(token)
        if err != nil {
            return nil // 生产代码应返回 error，而不是静默 nil
        }
        n := &TreeNode{Val: v}
        n.Left = dfs()
        n.Right = dfs()
        return n
    }
    return dfs()
}
```

### 高频追问

**Q：不加 `null` 占位为什么不行？**
> 反例：树 `[1,2]`（2 是左孩子）与 `[1,null,2]`（2 是右孩子），前序都是 `1,2`，无法区分。加上占位后分别是 `1,2,#,#,#` 和 `1,#,2,#,#`，唯一。

**Q：怎么优化序列化体积？**
> ① 层序 + 省略尾部 `#`；② 用二进制编码（4 字节 int + 1 字节 tag，比字符串小一个数量级）；③ 生产环境直接用 Protobuf/MessagePack 定义节点结构，跨语言兼容且带 schema 演进能力——手写字符串格式最大的隐患就是**没有任何向后兼容机制**。

**Q：深度很大会栈溢出吗？**
> 会。退化链状树递归深度 = n，Go 栈虽动态增长，但仍有上限（默认最大 1 GB）。生产环境应改为**显式栈迭代**，或用层序 + 队列（天然无深度风险）。

### 🗣️ 面试话术

"序列化的关键不是遍历方式，而是空节点必须显式占位，否则同一串能还原出多棵树；前序 + `#` + 共享游标是最简洁的解，工程上我更倾向 Protobuf 这种带 schema 的二进制格式。"

---

## 【8】🔥🔥🔥🔥 数组中的逆序对（剑指 Offer 51）：归并分治与树状数组

### 面试官考察意图

归并排序的**副产品**是逆序对计数，属于"模板题秒变加分题"的典型。面试官想看你能不能把"排序"与"统计"结合起来，以及数组规模到 10⁵ 时会不会漏掉 `int64` 溢出。

### 核心答案（30 秒版）

- 逆序对：`i < j` 且 `a[i] > a[j]`。暴力 O(n²) 必挂，正解 **归并排序中顺带统计 O(n log n)**。
- 统计时机：合并两个**已排序**子数组时，若取右段的 `a[j]`，说明左段从 `i` 到 `mid-1` 的**所有元素都 > a[j]**，一次性贡献 `mid - i` 个逆序对。
- 必须用 `int64`：n = 10⁵ 时逆序对最多约 n(n-1)/2 ≈ 5×10⁹，超出 `int32`（LeetCode 该题结果需对 1e9+7 取模正因如此）。
- 另一种解法是**树状数组/线段树**：从右往左扫描，统计已出现的比当前元素小的个数（离散化后 O(n log n)），同样好用但不是首选。

### Go 代码

```go
func reversePairs(nums []int) int64 {
    n := len(nums)
    if n < 2 {
        return 0
    }
    buf := make([]int, n)
    var cnt int64

    var mergeSort func(lo, hi int)
    mergeSort = func(lo, hi int) {
        if hi-lo <= 1 {
            return
        }
        mid := lo + (hi-lo)/2 // 防溢出写法，数组场景同样适用
        mergeSort(lo, mid)
        mergeSort(mid, hi)

        i, j, k := lo, mid, lo
        for i < mid && j < hi {
            if nums[i] <= nums[j] {
                buf[k] = nums[i]
                i++
            } else {
                buf[k] = nums[j]
                j++
                cnt += int64(mid - i) // 左段剩余元素均 > nums[j]
            }
            k++
        }
        for i < mid {
            buf[k] = nums[i]
            i++
            k++
        }
        for j < hi {
            buf[k] = nums[j]
            j++
            k++
        }
        copy(nums[lo:hi], buf[lo:hi])
    }

    mergeSort(0, n)
    return cnt
}
```

### 高频追问

**Q：为什么 `nums[i] <= nums[j]` 取左段，而不用 `<`？**
> 用 `<=` 保证相等时优先取左段，即"相等不算逆序对"且排序保持稳定。若题目要求把相等也算逆序对，改成 `<` 并计数。

**Q：能用快排的思路做吗？**
> 不能。快排的划分会打乱"下标顺序"这一前提，而逆序对定义依赖原始下标。必须用**不改变相对顺序语义**的分治（归并）或基于下标的统计结构（树状数组）。

**Q：树状数组版怎么做？**
> 对数组做离散化得到 rank，从右往左遍历，每次查询"已经出现的元素中 rank 小于当前的个数"累加，再把当前 rank 插入 BIT。时间 O(n log n)，常数比归并大，但可扩展到"区间逆序对"等变形。

### 🗣️ 面试话术

"逆序对就是归并排序的副产品：合并时右段元素先出，说明左段剩下的都比他大，直接累加 `mid - i`。要注意结果会超 int32，用 int64；树状数组也能做，但归并常数更小。"

---

## 【9】🔥🔥🔥 KMP 字符串匹配：手写 next 数组（LeetCode 28）

### 面试官考察意图

KMP 是"知道思想但写不出代码"的重灾区。面试官通常只要求你讲清楚**next 数组的物理含义**和**为什么能 O(n+m)**，然后手写 `strStr`。

### 核心答案（30 秒版）

- 暴力匹配的浪费点：失配时主串指针要**回退**，已匹配的信息被丢弃。KMP 的核心是"**主串指针 i 永不回退**"，失配时只滑动模式串（`j = next[j-1]`）。
- `next[i]` 的含义：模式串 `p[0..i]` 的**最长相等前后缀长度**（真前后缀，不含自身）。
- 预处理 next 是 O(m)，匹配是 O(n)，总计 **O(n + m)**，空间 O(m)。
- 写法要点：构建 next 时用 `k` 表示"当前最长相等前后缀长度"，`k` 同时充当模式串里的比较下标，与匹配阶段代码结构完全对称。

### Go 代码

```go
func strStr(haystack, needle string) int {
    if len(needle) == 0 {
        return 0
    }
    if len(needle) > len(haystack) {
        return -1
    }

    next := buildNext(needle)
    j := 0
    for i := 0; i < len(haystack); i++ {
        for j > 0 && haystack[i] != needle[j] {
            j = next[j-1] // 失配：模式串滑动，主串指针不动
        }
        if haystack[i] == needle[j] {
            j++
        }
        if j == len(needle) {
            return i - len(needle) + 1
        }
    }
    return -1
}

// next[i] = p[0..i] 的最长相等前后缀长度
func buildNext(p string) []int {
    next := make([]int, len(p))
    k := 0
    for i := 1; i < len(p); i++ {
        for k > 0 && p[i] != p[k] {
            k = next[k-1]
        }
        if p[i] == p[k] {
            k++
        }
        next[i] = k
    }
    return next
}
```

### 高频追问

**Q：为什么 KMP 是 O(n+m)？**
> 匹配阶段 `i` 单调递增（n 次），`j` 每次失配只能回退到 `next[j-1] < j`，而 `j` 的增量总和不超过 n，所以内层 `for` 的总回退次数被摊还到 O(n)。建表同理，`i` 走 m 步、`k` 摊还 O(m)。

**Q：只有 26 个小写字母，能不能更快？**
> 实务上可以换算法：**Boyer-Moore（坏字符 + 好后缀，实际文本搜索最快）**、**Rabin-Karp（滚动哈希，多模式/大文本友好）**，或 Go 标准库 `strings.Index` 用的 **Rabin-Karp 变体 + 小串优化**。Go 的 `strings.Index` 实现就是"长度短的串用暴力 + 长串用 Rabin-Karp 滚动哈希"，比 KMP 在真实负载下更快。

**Q：next 和"前缀函数"是同一个东西吗？**
> 是。有些写法用 `next[i]` 表示 `p[0..i-1]` 的最长前后缀（整体右移一位），两种定义只差一个下标偏移，面试时**先声明自己的定义**再写代码，避免和面试官对不上。

### 🗣️ 面试话术

"KMP 的关键是主串指针不回退，失配时按 next 数组滑动模式串；next[i] 是子串的最长相等前后缀长度，建表和匹配两段代码结构对称，整体 O(n+m)。工程上我会直接用 strings.Index，它是 Rabin-Karp 版，实际更快。"

---

## 题型速查表

| 题目 | 核心数据结构 | 时间 | 一句话识别 |
|------|--------------|------|-----------|
| LFU 缓存 | 哈希 + 频率分桶双向链表 | O(1) | "频率最低"淘汰 + minFreq 指针 |
| Trie / AC 自动机 | 26 叉树 | O(L) / O(n) | 前缀语义、多模式匹配 |
| 海量数据 TopK | 哈希分桶 + 小顶堆 / 位图 | O(N log K) | 内存放不下就先分治 |
| 排序链表 | 归并（自底向上） | O(n log n) | 链表不支持随机访问 |
| 第 K 大 | 快速选择 / 小顶堆 | 平均 O(n) / O(n log K) | 随机 pivot 防退化 |
| Dijkstra | 邻接表 + 小顶堆 | O((V+E) log V) | 带权单源最短路，无负权 |
| 树序列化 | 前序 + nil 占位 | O(n) | 空节点必须显式表达 |
| 逆序对 | 归并分治 / 树状数组 | O(n log n) | 统计与排序结合，int64 防溢出 |
| KMP | next 前缀函数 | O(n+m) | 主串指针不回退 |

## 高频追问（跨题）

**Q：设计类题（LRU/LFU/Trie/限流器）的通用答题框架？**
> 四步：① 明确接口与语义（容量、淘汰规则、并发要求）；② 选数据结构并说明为什么能到目标复杂度；③ 画一遍状态迁移（命中/未命中/淘汰/更新四条路径）；④ 主动提边界（容量 0/1、key 不存在、并发锁粒度）。面试官最看重第 2 和第 4 步。

**Q：怎么判断一道题该用分治还是 DP？**
> 看子问题是否**重叠**：重叠 → DP（记忆化/递推）；不重叠且可合并 → 分治（归并排序、逆序对、快速选择）。逆序对之所以用分治是因为左右子数组的答案独立，合并只需 O(n) 统计跨界部分。

## 延伸阅读

| 资源 | 链接 |
|------|------|
| LeetCode 460 LFU 缓存 | https://leetcode.cn/problems/lfu-cache/ |
| LeetCode 211 通配符前缀树 | https://leetcode.cn/problems/design-add-and-search-words-data-structure/ |
| LeetCode 743 网络延迟时间 | https://leetcode.cn/problems/network-delay-time/ |
| LeetCode 297 二叉树序列化 | https://leetcode.cn/problems/serialize-and-deserialize-binary-tree/ |
| LeetCode 148 排序链表 | https://leetcode.cn/problems/sort-list/ |
| 剑指 Offer 51 数组中的逆序对 | https://leetcode.cn/problems/shu-zu-zhong-de-ni-xu-dui-lcof/ |
| AC 自动机原理（Trie + fail 指针） | https://oi-wiki.org/string/ac-automaton/ |
| TinyLFU 论文解读（缓存淘汰前沿） | https://arxiv.org/abs/1512.00727 |
