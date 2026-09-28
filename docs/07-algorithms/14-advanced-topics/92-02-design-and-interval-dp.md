# 进阶高频题（二）：中位数二分 / 设计类数据结构 / 区间 DP / 原地算法

> 考察频率：★★★★☆  优先级：P2  语言：Go
> 题型识别：二分分割、O(1) 设计类、区间 DP、原地哈希置换、大数模拟、概率拒绝采样、三次反转
> 说明：本篇补齐 07 模块此前未覆盖的 9 道大厂高频题。LRU/LFU 见 [91-01-advanced-topics.md](91-01-advanced-topics.md)，前缀和模板见 [02-golang-impl/78-01-go-tips.md](../02-golang-impl/78-01-go-tips.md)。

---

## 【1】🔥🔥🔥🔥🔥 寻找两个正序数组的中位数（LeetCode 4）

### 面试官考察意图

这是二分查找的**天花板题**：把"在一个数组里二分"升级成"在两个数组上二分分割点"。面试官主要看两点——你能不能想到"**分割**"而不是"归并"，以及边界（分割点在端点）处理得干不干净。

### 核心答案（30 秒版）

- 要求在 **O(log(m+n))** 内完成，所以只能二分，不能用"合并两个有序数组"的归并 O(m+n)。
- 核心：在两个数组上找一个**分割线**，使左半部分元素个数 = `(m+n+1)/2`（左偏，方便处理奇偶），且**左半最大值 ≤ 右半最小值**。
- 记 `nums1` 左侧取 `i` 个、`nums2` 左侧取 `j = totalLeft - i` 个，则只需满足：
  - `nums1[i-1] <= nums2[j]`
  - `nums2[j-1] <= nums1[i]`
- 若 `nums1[i-1] > nums2[j]`，说明 `nums1` 切得太靠右 → `hi = i-1`；反之 `lo = i+1`。
- **始终保证 `nums1` 更短**，这样 j 不会越界，二分区间也更小。
- 边界用 ±∞ 兜底（`i==0` 时左半最大为 `-∞`，`i==m` 时右半最小为 `+∞`），避免写一堆 if。
- 复杂度：时间 O(log min(m,n))，空间 O(1)。

### Go 代码

```go
func findMedianSortedArrays(nums1, nums2 []int) float64 {
    // 保证 nums1 是较短的数组，使二分区间最小
    if len(nums1) > len(nums2) {
        nums1, nums2 = nums2, nums1
    }
    m, n := len(nums1), len(nums2)
    totalLeft := (m + n + 1) / 2 // 左半部分应包含的元素个数（含中位数本身）

    lo, hi := 0, m
    for lo <= hi {
        i := lo + (hi-lo)/2 // nums1 的左半取 i 个
        j := totalLeft - i  // nums2 的左半取 totalLeft-i 个

        // 越界一侧视为正负无穷，省去大量条件分支
        nums1Left, nums1Right := math.MinInt, math.MaxInt
        if i > 0 {
            nums1Left = nums1[i-1]
        }
        if i < m {
            nums1Right = nums1[i]
        }
        nums2Left, nums2Right := math.MinInt, math.MaxInt
        if j > 0 {
            nums2Left = nums2[j-1]
        }
        if j < n {
            nums2Right = nums2[j]
        }

        switch {
        case nums1Left <= nums2Right && nums2Left <= nums1Right:
            // 分割合理：左半最大 max(nums1Left,nums2Left)，右半最小 min(nums1Right,nums2Right)
            if (m+n)%2 == 1 {
                return float64(max(nums1Left, nums2Left))
            }
            return float64(max(nums1Left, nums2Left)+min(nums1Right, nums2Right)) / 2.0
        case nums1Left > nums2Right:
            hi = i - 1 // nums1 左侧取多了，往左收
        default:
            lo = i + 1 // nums1 左侧取少了，往右扩
        }
    }
    return 0 // 逻辑上不可达（输入保证有序）
}
```

### 高频追问

**Q：为什么 `totalLeft` 取 `(m+n+1)/2` 而不是 `(m+n)/2`？**
> 让左半部分"多一个"，这样奇数长度时中位数天然落在左半最大值上，偶数时左右各占一半，两种情况共用同一套 `max/min` 公式，省掉奇偶分支。

**Q：为什么一定要先交换成 `len(nums1) <= len(nums2)`？**
> 因为 `j = totalLeft - i` 必须落在 `[0, n]`。当 m ≤ n 时，`i ∈ [0,m]` 对应的 j 一定合法，无需额外判 `j` 越界；且二分区间取较短数组，复杂度 O(log min(m,n))。

**Q：中位数和"两个有序数组的第 K 小"是什么关系？**
> 同一类问题的特例。求第 K 小可用"每次排除 `k/2` 个元素"的二分，或上面的分割二分。面试若被追问第 K 小，直接说"分割法改一下 `totalLeft = K` 即可"。

### 🗣️ 面试话术

"这题不能用归并，必须在两个数组上二分出一个分割线：左边一共 `(m+n+1)/2` 个，且左半最大 ≤ 右半最小。保证 nums1 更短可以让 j 不越界，边界用正负无穷兜底，整体 O(log min(m,n))。"

---

## 【2】🔥🔥🔥🔥🔥 最小栈 / 栈队列互转（LeetCode 155 / 232 / 225）

### 面试官考察意图

设计类"送分题"，但**答不全就是送命**。面试官想看：① 最小栈的辅助栈为什么要用"单调不增"而不是"每次只存当前最小"；② 栈队列互转的**摊还复杂度**能不能说清。

### 核心答案（30 秒版）

- **最小栈**：主栈 + **辅助栈**，辅助栈与主栈**等长**，每个位置存"压入该元素后的栈内最小值"（单调不增）。
  - 压栈：`x <= mins.top` 时压 `x`，否则压 `mins.top`（重复当前最小）。
  - 关键：必须用 `<=` 而不是 `<`，否则**重复的最小值**出栈时会把最小值提前弹掉。
  - getMin / push / pop / top 全部 **O(1)**，空间 O(n)。
- **用栈实现队列**：两个栈 `in` / `out`。push 进 `in`；pop/peek 时若 `out` 为空，把 `in` 全部倒入 `out` 再弹。
  - 每个元素最多被搬运一次 → **摊还 O(1)**，均摊到每次操作。
- **用队列实现栈**：单队列，push 时把新元素入队后，将队头的前 n-1 个元素依次"出队再入队"（旋转），使新元素始终在队头。push O(n)，pop/top O(1)。

### Go 代码

```go
// ===== 155. 最小栈 =====
type MinStack struct {
    data []int
    mins []int // mins[i] = 压入 data[i] 之后栈内的最小值（单调不增）
}

func (s *MinStack) Push(x int) {
    s.data = append(s.data, x)
    if len(s.mins) == 0 || x <= s.mins[len(s.mins)-1] {
        s.mins = append(s.mins, x) // 更小或相等：更新最小值
    } else {
        s.mins = append(s.mins, s.mins[len(s.mins)-1]) // 否则复制当前最小
    }
}

func (s *MinStack) Pop() {
    s.data = s.data[:len(s.data)-1]
    s.mins = s.mins[:len(s.mins)-1]
}

func (s *MinStack) Top() int    { return s.data[len(s.data)-1] }
func (s *MinStack) GetMin() int { return s.mins[len(s.mins)-1] }
```

```go
// ===== 232. 用栈实现队列 =====
type MyQueue struct {
    in, out []int
}

func (q *MyQueue) Push(x int) { q.in = append(q.in, x) }

func (q *MyQueue) transfer() {
    if len(q.out) == 0 {
        for len(q.in) > 0 {
            q.out = append(q.out, q.in[len(q.in)-1])
            q.in = q.in[:len(q.in)-1]
        }
    }
}

func (q *MyQueue) Pop() int {
    q.transfer()
    v := q.out[len(q.out)-1]
    q.out = q.out[:len(q.out)-1]
    return v
}

func (q *MyQueue) Peek() int  { q.transfer(); return q.out[len(q.out)-1] }
func (q *MyQueue) Empty() bool { return len(q.in) == 0 && len(q.out) == 0 }
```

```go
// ===== 225. 用队列实现栈 =====
type MyStack struct{ q []int }

func (s *MyStack) Push(x int) {
    s.q = append(s.q, x)
    // 把 x 之前的所有元素从队头搬到队尾，使 x 位于队头
    for i := 0; i < len(s.q)-1; i++ {
        s.q = append(s.q, s.q[0])
        s.q = s.q[1:]
    }
}

func (s *MyStack) Pop() int {
    v := s.q[0]
    s.q = s.q[1:]
    return v
}

func (s *MyStack) Top() int    { return s.q[0] }
func (s *MyStack) Empty() bool { return len(s.q) == 0 }
```

### 高频追问

**Q：最小栈的辅助栈为什么不能只存 `min(x, mins.top)` 的"当前全局最小"？**
> 那样当最小值被弹出后，你无法恢复"次小值"。让辅助栈与主栈等长、每个位置都记录"该时刻的栈内最小"，出栈时同步弹出即可正确回退。也可以用"只压入更小值 + 记录频率"的压缩写法（space 更省），但实现更容易写错。

**Q：为什么用 `<=` 而不是 `<`？**
> 反例：依次 push `0, 0, 1`。若用 `<`，第二次 push `0` 不更新 mins，mins 只有 `[0]`；此时 pop 掉栈顶 `1` 不影响，但再 pop 掉一个 `0` 时 mins 会弹出唯一的 `0`，导致 `GetMin` 出错（另一个 `0` 还在栈里）。用 `<=` 让两个 `0` 都在 mins 里，弹出才一一对应。

**Q：栈实现队列的复杂度到底是多少？**
> 均摊（摊还）O(1)：每个元素一生中最多从 `in` 搬到 `out` 一次，总搬运次数 ≤ n，所以 n 次操作的摊还代价是 O(1)/次。最坏单次（触发搬运）是 O(n)。

### 🗣️ 面试话术

"最小栈用辅助栈与主栈等长、每个位置记录该时刻的最小值，注意比较用 `<=` 才能正确处理重复最小值；栈实现队列用 in/out 两个栈，元素最多搬一次所以是摊还 O(1)。"

---

## 【3】🔥🔥🔥🔥🔥 和为 K 的子数组（LeetCode 560）

### 面试官考察意图

**前缀和 + 哈希**的招牌题。面试官要看你能不能意识到："**元素可以为负，所以滑动窗口失效**"，以及哈希表里为什么要预置 `{0: 1}`。

### 核心答案（30 秒版）

- 子数组和 = `prefix[j] - prefix[i]`，要它等于 k，即找**前面出现过多少个 `prefix[j] - k`**。
- 一趟遍历：维护当前前缀和 `prefix`，用哈希表记录"每个前缀和出现的次数"，答案累加 `seen[prefix-k]`。
- **必须预置 `seen[0] = 1`**：表示"空前缀和"，用于统计从下标 0 开始的子数组。
- 复杂度：时间 O(n)，空间 O(n)。
- **不能用滑动窗口**：数组含负数时窗口右扩不保证和增大、左缩不保证和减小，单调性被破坏。

### Go 代码

```go
func subarraySum(nums []int, k int) int {
    count := 0
    prefix := 0
    seen := map[int]int{0: 1} // 前缀和 -> 出现次数；0:1 代表空子数组
    for _, v := range nums {
        prefix += v
        count += seen[prefix-k] // 之前有多少个前缀和 == prefix-k，就有多少个子数组和为 k
        seen[prefix]++
    }
    return count
}
```

### 高频追问

**Q：如果数组全是正数，还有更优解吗？**
> 全是正数时窗口有单调性，可以用**滑动窗口**把空间降到 O(1)。但题目未保证正数，所以通用解还是前缀和 + 哈希 O(n)/O(n)。

**Q：`seen[0] = 1` 不加会怎样？**
> 会漏掉所有"从下标 0 开始"的子数组。例如 `nums=[1,1,1], k=2`，正确结果是 2；不预置时只统计到中间的 `[1,1]`，漏掉前两个。

**Q：变种"和可被 K 整除的子数组"（LC 974）怎么做？**
> 同样是前缀和，但把哈希表的 key 换成**前缀和模 K 的余数**（负数取模要修正为 `((prefix % K) + K) % K`），统计相同余数出现次数组合数即可。

### 🗣️ 面试话术

"子数组和等于 k 等价于 `prefix[j] - prefix[i] = k`，所以边扫边用哈希表统计历史前缀和，答案加 `seen[prefix-k]`，并预置 `seen[0]=1`。因为存在负数，滑动窗口的单调性不成立，只能用前缀和。"

---

## 【4】🔥🔥🔥🔥 戳气球 / 区间 DP（LeetCode 312）

### 面试官考察意图

区间 DP 的经典代表。难点在于**逆向思考**：正向"先戳哪个"无法拆分，逆向"最后戳哪个"才能划分子问题。

### 核心答案（30 秒版）

- 正向枚举"第一个戳的气球"，左右两边会因相邻关系耦合，无法独立；**改成枚举"最后一个戳的气球 k"**，则戳破 k 时它的左右邻居已固定为区间边界 `i`、`j`，问题拆成 `(i,k)` 和 `(k,j)` 两个独立子区间。
- 在数组首尾各补一个值为 `1` 的哨兵，得到 `val[0..n+1]`。
- 定义 `dp[i][j]` = 戳破开区间 `(i,j)` 内所有气球的最大收益（i、j 保留）。
- 转移：`dp[i][j] = max( dp[i][k] + val[i]*val[k]*val[j] + dp[k][j] )`，`k ∈ (i,j)`。
- 按**区间长度从小到大**枚举（区间 DP 通用顺序），答案 `dp[0][n+1]`。
- 复杂度 O(n³) 时间，O(n²) 空间。

### Go 代码

```go
func maxCoins(nums []int) int {
    n := len(nums)
    // 首尾补哨兵 1，避免处理边界越界
    val := make([]int, n+2)
    val[0], val[n+1] = 1, 1
    copy(val[1:], nums)

    // dp[i][j]: 戳破开区间 (i,j) 内所有气球的最高分
    dp := make([][]int, n+2)
    for i := range dp {
        dp[i] = make([]int, n+2)
    }

    // 区间 DP 顺序：先枚举长度，再枚举起点
    for length := 3; length <= n+2; length++ {
        for i := 0; i+length-1 <= n+1; i++ {
            j := i + length - 1
            for k := i + 1; k < j; k++ { // k 为区间内"最后一个被戳破"的气球
                total := dp[i][k] + val[i]*val[k]*val[j] + dp[k][j]
                if total > dp[i][j] {
                    dp[i][j] = total
                }
            }
        }
    }
    return dp[0][n+1]
}
```

### 高频追问

**Q：为什么区间 DP 一定要按"长度"枚举？**
> 因为 `dp[i][j]` 依赖更短的区间 `dp[i][k]`、`dp[k][j]`（k 在 i、j 之间）。按长度从小到大枚举，能保证计算大区间时所有子区间已算好。按 i 或 j 枚举都会遇到"依赖未计算"的问题。

**Q：为什么正向思路行不通？**
> 若枚举"第一个戳的气球 k"，戳掉后左右两半会**变成相邻**，它们的得分互相影响，子问题不独立。逆向枚举"最后一个"相当于把 k 当作分割点，戳 k 时左右两半已清空，边界固定为 i、j，子问题才独立可分。

**Q：还有哪些区间 DP 高频题？**
> 矩阵链乘、最长回文子序列（LC 516）、戳气球（LC 312）、移除盒子（LC 546，难度更高）。识别信号：**答案与"移除/合并区间"相关，且边界选择会改变子问题**。

### 🗣️ 面试话术

"这题要逆向想：枚举区间内**最后一个**被戳的气球 k，这样它左右邻居已固定为区间边界，问题拆成两个独立子区间，就是标准的区间 DP。首尾补哨兵 1，按区间长度从小到大枚举，O(n³)。"

---

## 【5】🔥🔥🔥🔥 O(1) 时间插入、删除和获取随机元素（LeetCode 380 / 381）

### 面试官考察意图

哈希 + 数组的经典组合设计题。考"**O(1) 删除**"的实现技巧——用"尾部元素覆盖被删位置"，以及支持重复元素（381）时哈希值改为集合的处理。

### 核心答案（30 秒版）

- 需求：insert / remove / getRandom 全 **O(1)**。
- 结构：**动态数组 `nums`（存值） + 哈希表 `idx`（值 → 数组下标）**。
- 插入：值不存在则追加到 `nums` 末尾，记录下标。
- **删除的关键**：把 `nums` 的**最后一个元素搬到被删元素的位置**，更新其下标，再 `nums = nums[:n-1]`。这样避免数组中间删除引发的整体搬移 O(n)。
- 随机：`nums[rand.IntN(len(nums))]`，数组天然支持等概率随机访问。
- 复杂度：三个操作均 O(1)（哈希表期望 O(1)）。
- **LC 381（允许重复）**：哈希表存 `值 → 下标集合（map[int]map[int]struct{} 或 []int）`，删除时从集合中任取一个下标与末尾交换。

### Go 代码

```go
type RandomizedSet struct {
    nums []int
    idx  map[int]int // val -> 在 nums 中的下标
}

func NewRandomizedSet() *RandomizedSet {
    return &RandomizedSet{idx: make(map[int]int)}
}

func (s *RandomizedSet) Insert(val int) bool {
    if _, ok := s.idx[val]; ok {
        return false
    }
    s.idx[val] = len(s.nums)
    s.nums = append(s.nums, val)
    return true
}

func (s *RandomizedSet) Remove(val int) bool {
    i, ok := s.idx[val]
    if !ok {
        return false
    }
    last := s.nums[len(s.nums)-1]
    s.nums[i] = last
    s.idx[last] = i // 更新被搬过来的元素的下标
    s.nums = s.nums[:len(s.nums)-1]
    delete(s.idx, val)
    return true
}

func (s *RandomizedSet) GetRandom() int {
    return s.nums[rand.IntN(len(s.nums))] // math/rand/v2
}
```

### 高频追问

**Q：为什么用数组就能 O(1) 随机，而链表不行？**
> "等概率随机"要求 O(1) 定位到第 k 个元素，只有**连续内存 + 下标计算**才能做到（数组）。链表访问第 k 个要 O(k)。这也是为什么这里不选链表。

**Q：删除时"与末尾交换"有什么坑？**
> ① 被删元素**恰好就是末尾元素**时，`last == val`，先更新 `idx[last]` 再 `delete` 会把刚更新的下标删掉——要注意顺序（上面代码先赋值后 delete，此时 `delete(idx, val)` 删的正是同一个 key，等价正确）。② 删除后必须 `delete(s.idx, val)`，否则还会被判定为"存在"。③ 空集合时不能调用 `GetRandom`。

**Q：381（允许重复）怎么改？**
> 哈希表存 `val -> 下标集合`。删除时从集合取一个下标 `i`，把末尾元素 `last` 搬到 `i`，更新 `last` 的集合（删掉末尾下标、加入 `i`），再缩小数组。`getRandom` 逻辑不变。注意集合里同名元素的多个下标要一致维护。

### 🗣️ 面试话术

"核心是数组 + 哈希表：数组保证 O(1) 随机访问，哈希表记下标保证 O(1) 查找。删除时把末尾元素搬过来覆盖被删位置再缩容，避开了数组 O(n) 的中间删除，三个操作都是 O(1)。"

---

## 【6】🔥🔥🔥🔥 缺失的第一个正数（LeetCode 41）

### 面试官考察意图

考"**原地哈希**"：不给额外空间（要求 O(1) 空间），要你想到用数组本身当下标映射表。同时也考对"答案一定在 `[1, n+1]`"这一性质的洞察。

### 核心答案（30 秒版）

- 关键观察：长度为 n 的数组，缺失的第一个正数**一定在 `[1, n+1]`** 内（1..n 全占满则答案是 n+1）。
- 思路：把数值 `v ∈ [1, n]` **交换到下标 `v-1`** 上，让 `nums[i] == i+1` 尽量成立。
- 一轮置换后，从头扫描第一个 `nums[i] != i+1` 的位置，答案就是 `i+1`；全部匹配则返回 `n+1`。
- 交换条件要写全：`nums[i] > 0 && nums[i] <= n && nums[nums[i]-1] != nums[i]`——最后一条防止**死循环**（目标位置已经是正确的值）。
- 复杂度：时间 O(n)（每个元素最多被交换到正确位置一次），空间 O(1)。

### Go 代码

```go
func firstMissingPositive(nums []int) int {
    n := len(nums)
    for i := 0; i < n; i++ {
        // 不断把 nums[i] 放到它"应该在"的下标 nums[i]-1 上
        for nums[i] > 0 && nums[i] <= n && nums[nums[i]-1] != nums[i] {
            nums[nums[i]-1], nums[i] = nums[i], nums[nums[i]-1]
        }
    }
    for i := 0; i < n; i++ {
        if nums[i] != i+1 {
            return i + 1
        }
    }
    return n + 1
}
```

### 高频追问

**Q：为什么不会死循环？**
> 内层循环的条件里 `nums[nums[i]-1] != nums[i]` 保证"只有当目标位置放的不是同一个值时才交换"。每次交换都会把一个元素放到它的最终位置，因此每个位置最多被正确填充一次，总交换次数 ≤ n，内层循环摊还 O(1)。

**Q：Go 的 `a, b = b, a` 在含下标的交换里安全吗？**
> 安全。Go 规定先对右侧表达式**全部求值**，再从左到右赋值。这里右侧 `nums[i]` 和 `nums[nums[i]-1]` 用的是交换前的值，因此等价于标准的临时变量交换。

**Q：如果允许 O(n) 额外空间呢？**
> 可以用哈希集合标记 `[1,n]` 内出现的数，再扫一遍找缺失值；或者建一个 `bool[n+1]`。更简洁但空间不达最优。原地置换法是这题的"最优解姿势"。

### 🗣️ 面试话术

"答案是 1..n+1 之间的某个数，所以把值为 v 的元素换到下标 v-1，让 nums[i]==i+1 尽量成立；扫到第一个不匹配的位置返回 i+1。关键是判断目标位置是否已正确来避免死循环，时间 O(n)、空间 O(1)。"

---

## 【7】🔥🔥🔥🔥 字符串相乘（LeetCode 43）

### 面试官考察意图

考**大数运算**和数组下标的精细控制。不允许用 `big.Int` 或直接转 int（数字可能超出 64 位），面试官看的是你能不能"按位竖式模拟"并处理好进位。

### 核心答案（30 秒版）

- 结果长度最多 `m + n` 位，创建结果数组 `res[m+n]`。
- 双层循环模拟竖式：`num1[i] * num2[j]` 的结果应落在 `res[i+j]` 和 `res[i+j+1]` 两个位置（**低位在 i+j+1**）。
- 每次先 `sum = mul + res[p2]`，`res[p2] = sum % 10`，`res[p1] += sum / 10`。
- **不单独维护进位变量**——把进位直接累加到高位 `res[p1]` 上，写法最简洁。
- 最后跳过前导零拼成字符串；`num1` 或 `num2` 为 `"0"` 时直接返回 `"0"`。
- 复杂度：时间 O(m·n)，空间 O(m+n)。

### Go 代码

```go
func multiply(num1, num2 string) string {
    if num1 == "0" || num2 == "0" {
        return "0"
    }
    m, n := len(num1), len(num2)
    res := make([]int, m+n) // 结果最多 m+n 位

    for i := m - 1; i >= 0; i-- {
        for j := n - 1; j >= 0; j-- {
            mul := int(num1[i]-'0') * int(num2[j]-'0')
            p1, p2 := i+j, i+j+1 // p2 是低位，p1 是进位落点
            sum := mul + res[p2]
            res[p2] = sum % 10
            res[p1] += sum / 10 // 进位累加到高位
        }
    }

    var b strings.Builder
    for i, d := range res {
        if i == 0 && d == 0 {
            continue // 跳过前导零
        }
        b.WriteByte(byte('0' + d))
    }
    return b.String()
}
```

### 高频追问

**Q：为什么用 `strings.Builder` 而不是 `+=`？**
> Go 的字符串不可变，`s += x` 每次都会分配新内存并复制，循环里是 O(n²)。`strings.Builder` 内部是 `[]byte` 追加，最后一次性转字符串，是 O(n)。

**Q：为什么不需要单独的进位变量？**
> 因为 `res[p1] += sum / 10` 把进位直接加在了高位格子上。下一轮计算同一格子时，`res[p2]` 读取的已经是它包含的既有值，进位自然被并入，相当于"进位随竖式逐位累积"。

**Q：变种"字符串相加"（LC 415）怎么写？**
> 双指针从末尾同步前进，用一个 `carry` 变量，逐位相加取模、拼字符，最后反转字符串即可。乘法的区别在于要处理"每一位乘完后落入两个格子"的二维累积。

### 🗣️ 面试话术

"用 m+n 长度的结果数组按竖式模拟：`num1[i]*num2[j]` 落在 i+j 和 i+j+1 两个位置，低位取模、高位加进位，不需要单独进位变量；最后跳过前导零用 strings.Builder 拼接，时间 O(mn)。"

---

## 【8】🔥🔥🔥🔥 用 Rand7() 实现 Rand10()（LeetCode 470）

### 面试官考察意图

概率类高频题的标杆，考察**拒绝采样**与期望复杂度分析。也常被问"能不能不用循环/怎么减少拒绝"。

### 核心答案（30 秒版）

- 单次 `rand7()` 只有 7 个等概率取值，无法直接均匀映射到 10 个值（10 不是 7 的倍数）。
- **拒绝采样**：调用两次 `rand7()` 构造 `num = (row-1)*7 + col`，得到 **1..49 的均匀分布**。
- `num <= 40` 时，`(num-1) % 10 + 1` 均匀落在 1..10（40 = 4×10，能整除）。
- `num ∈ [41,49]` 时**拒绝并重采**。
- 每次采样的成功率 = 40/49 ≈ 81.6%，期望调用次数 = 49/40 ≈ 1.225 轮 → **期望 O(1)**，最坏情况无上界但概率指数衰减。

### Go 代码

```go
func rand10() int {
    for {
        // 两次 rand7 构造 1..49 的均匀分布
        num := (rand7()-1)*7 + rand7() // 1..49
        if num <= 40 {                 // 40 = 4*10 能被 10 整除
            return (num-1)%10 + 1
        }
        // 41..49 落在"尾部区间"，拒绝后重采
    }
}
```

### 高频追问

**Q：为什么是 1..49 而不是 1..14？**
> `(a-1)*7 + b` 是标准"进制拼接"：a 作高位（1..7）、b 作低位（1..7），组合出 49 个等概率状态。只用一次加法或乘法无法保证均匀性；用两数相乘（如 `rand7()*rand7()`）取值分布也不均匀（乘积不是等概率）。

**Q：期望调用次数怎么算？**
> 每轮成功率 p = 40/49，轮数服从几何分布，期望轮数 = 1/p = 49/40 = 1.225 轮，每轮 2 次调用，故期望约 2.45 次 `rand7()` 调用。

**Q：怎么提高利用率（减少拒绝）？**
> 把拒绝的 `num ∈ [41,49]`（9 个值）**再利用**：例如再采一次构造 `num' = (num-41)*7 + rand7() ∈ [1,63]`，其中 60 个可用 → 成功率从 40/49 提升到约 0.816 × ... 逐级逼近 1（这就是"拒绝采样 + 缓存剩余随机性"的优化思路）。工程上还可直接用 `rand.IntN(10)` 避免自己实现。

### 🗣️ 面试话术

"两次 rand7 拼成 1..49 的均匀分布，取前 40 个用模 10 映射到 1..10，剩下 9 个拒绝重采。成功率 40/49，期望轮数 1.225，整体期望 O(1)。"

---

## 【9】🔥🔥🔥 轮转数组：三次反转（LeetCode 189）

### 面试官考察意图

考"**原地 O(1) 空间**"的技巧题。面试官常要求"能不能不用额外数组"，以及追问"还有哪些做法、各自复杂度"。

### 核心答案（30 秒版）

- 要求原地、O(1) 额外空间，把数组右移 k 位（`k %= n` 先取模，防止 k > n 越界）。
- **三次反转**：① 反转整个数组 → ② 反转前 k 个 → ③ 反转后 n-k 个，结果即为右移 k 位。
- 原理：整体反转把末尾 k 个"翻到"前面（但顺序反了），再分别反转两段把各自顺序恢复。
- 复杂度：时间 O(n)，空间 O(1)。
- 其他解法：额外数组 `(i+k)%n` 放置（O(n) 空间）；循环替换（O(1) 空间但实现繁琐、需处理 gcd 分组）。

### Go 代码

```go
func rotate(nums []int, k int) {
    n := len(nums)
    if n == 0 {
        return
    }
    k %= n // 防止 k > n
    reverse(nums)
    reverse(nums[:k])
    reverse(nums[k:])
}

func reverse(a []int) {
    for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
        a[i], a[j] = a[j], a[i]
    }
}
```

### 高频追问

**Q：为什么三次反转能得到"右移 k 位"？**
> 设原数组 `A B`（A 长 n-k，B 长 k），目标是 `B A`。整体反转得 `rev(B) rev(A)`；再反转前 k 个得 `rev(rev(B)) rev(A) = B rev(A)`；再反转后 n-k 个得 `B A`。三步完成目标排列。

**Q：`nums[:k]` 和 `nums[k:]` 是拷贝吗？**
> 不是。Go 的切片是**共享底层数组**的视图，`reverse(nums[:k])` 直接修改原数组对应区间，这正是"原地"成立的原因。注意 `k=0` 时 `nums[:0]` 为空切片，`reverse` 不会执行，行为正确。

**Q：为什么必须先 `k %= n`？**
> 当 `k >= n` 时，`nums[:k]` 会越界 panic。取模后 `k ∈ [0,n)` 保证切片合法；且右移 n 位等于没动，取模也不改变结果。

### 🗣️ 面试话术

"先对 k 取模，然后整体反转、再分别反转前 k 个和后 n-k 个，等价于右移 k 位，时间 O(n)、空间 O(1)。Go 里切片是共享底层数组的视图，所以反转切片就是原地操作。"

---

## 题型速查表

| 题目 | 核心数据结构 / 技巧 | 时间 | 一句话识别 |
|------|--------------------|------|-----------|
| 两个正序数组的中位数 | 二分分割 + ±∞ 兜底 | O(log min(m,n)) | 不能归并，要"分割线" |
| 最小栈 | 主栈 + 等长辅助栈 | O(1) | 记录"每个时刻的最小值" |
| 栈实现队列 | 双栈 in/out | 摊还 O(1) | 元素最多搬一次 |
| 和为 K 的子数组 | 前缀和 + 哈希（预置 0:1） | O(n) | 含负数，滑动窗口失效 |
| 戳气球 | 区间 DP（枚举最后戳破） | O(n³) | 逆向枚举分割点 |
| O(1) 增删随机 | 数组 + 哈希下标 | O(1) | 删末尾元素覆盖被删位置 |
| 缺失的第一个正数 | 原地置换 | O(n) | 答案在 [1, n+1] |
| 字符串相乘 | 竖式模拟 + m+n 数组 | O(mn) | 大数不用 big.Int |
| Rand7 → Rand10 | 拒绝采样 | 期望 O(1) | 拼成 49 再取 40 |
| 轮转数组 | 三次反转 | O(n) | 原地 O(1) 空间 |

## 高频追问（跨题）

**Q：设计类题（栈 / 集合 / 缓存）的通用答题框架？**
> 四步：① 定接口语义与复杂度目标；② 选数据结构并说明**为什么能达标**（比如数组给随机、哈希给查找、双向链表给 O(1) 增删）；③ 画一遍边界路径（空结构、单元素、重复值、删除末尾）；④ 主动说出**空间开销**与并发假设。80% 的失分在第 ③ 步的边界。

**Q：什么时候用哈希 + 数组的组合？**
> 当需要同时满足"**按值 O(1) 查找**"和"**按下标 O(1) 随机访问 / 均匀采样**"时。哈希负责前者、数组负责后者，再用"尾部覆盖删除"保证 O(1) 删除。LRU/LFU、随机集合、多人共享对象池都是这个套路。

## 延伸阅读

| 资源 | 链接 |
|------|------|
| LeetCode 4 寻找两个正序数组的中位数 | https://leetcode.cn/problems/median-of-two-sorted-arrays/ |
| LeetCode 155 最小栈 | https://leetcode.cn/problems/min-stack/ |
| LeetCode 232 用栈实现队列 | https://leetcode.cn/problems/implement-queue-using-stacks/ |
| LeetCode 560 和为 K 的子数组 | https://leetcode.cn/problems/subarray-sum-equals-k/ |
| LeetCode 312 戳气球 | https://leetcode.cn/problems/burst-balloons/ |
| LeetCode 380 O(1) 时间插入、删除和获取随机元素 | https://leetcode.cn/problems/insert-delete-getrandom-o1/ |
| LeetCode 41 缺失的第一个正数 | https://leetcode.cn/problems/first-missing-positive/ |
| LeetCode 43 字符串相乘 | https://leetcode.cn/problems/multiply-strings/ |
| LeetCode 470 用 Rand7() 实现 Rand10() | https://leetcode.cn/problems/implement-rand10-using-rand7/ |
| LeetCode 189 轮转数组 | https://leetcode.cn/problems/rotate-array/ |
