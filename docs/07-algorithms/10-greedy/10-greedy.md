# 贪心算法高频题

> 考察频率：★★★★☆  优先级：P0  Go + Python 双语

[🏠 首页](../../../README.md) · [📚 算法模块](../README.md)

---

## 1. 贪心算法核心思想

### 什么是贪心

**贪心算法（Greedy Algorithm）**：在每一步决策时，都选择当前状态下的**最优解**（局部最优），期望通过一系列局部最优得到全局最优。

核心哲学：**"鼠目寸光"但"目光锐利"** —— 不考虑未来的后悔，只管眼前最好的选择。

```python
# 贪心算法的一般模式
def greedy(problem):
    solution = []
    while problem not solved:
        choice = select_best_local_option(problem)  # 选局部最优
        solution.append(choice)
        update(problem, choice)                      # 更新状态
    return solution
```

### 贪心 vs 动态规划

| 特征 | 贪心 | 动态规划 |
|------|------|----------|
| **选择方式** | 只选当前最优，不考虑之前的选择 | 考虑所有可能的选择（状态转移） |
| **后悔机制** | ❌ 不回头，**无后悔** | ✅ 会回头，重新考虑 |
| **时间复杂度** | 通常 O(N) 或 O(NlogN) | 通常 O(N²) 或 O(N×M) |
| **解的质量** | 不一定最优（需证明） | 一定最优 |
| **适用场景** | 问题具有**贪心选择性质** | 问题具有**最优子结构** |

**简单判断**：

- 动态规划问："**全局最优要不要包含这个子问题的最优解？**"（选或不选，都要比较）
- 贪心算法问："**当前这一步选谁最好？**"（只有一个正确方向，不需要比较所有历史）

### 证明贪心正确性：交换论证法（Exchange Argument）

交换论证是证明贪心算法正确性的最常用方法，分三步：

1. **假设**存在一个最优解 O，**不是**用贪心策略构造的
2. **比较**：将 O 中第一个与贪心选择不同的元素替换（交换）为贪心选择的元素
3. **证明**：这个交换**不会**使解变差，即替换后仍是全局最优解
4. **递推**：重复交换，最终将 O 转化为贪心解，且过程中解的质量不下降

> 通俗理解：最优解里排在第一位的如果不是贪心选的那个，就把它换成贪心选的——换完不但没变差，反而可能更好。最终最优解就变成了贪心解。

---

## 2. 区间调度（最高频）

### LeetCode 435 - 无重叠区间

> **题意**：给定一组区间，问至少移除多少个区间才能使剩余区间互不重叠。

**贪心策略**：按结束时间排序，优先选择最早结束的区间——这样能给后面的区间留出更多空间。

```python
# Python 实现
def eraseOverlapIntervals(intervals: list[list[int]]) -> int:
    if not intervals:
        return 0

    # 按结束时间升序排序
    intervals.sort(key=lambda x: x[1])

    count = 1          # 选择的区间数量
    end = intervals[0][1]  # 当前选中区间的结束时间

    for i in range(1, len(intervals)):
        # 如果当前区间的开始时间 >= 上一个选中区间的结束时间
        # 说明不重叠，可以选择
        if intervals[i][0] >= end:
            count += 1
            end = intervals[i][1]

    return len(intervals) - count  # 移除的最少数量 = 总数 - 选择的最多数

# 测试
intervals = [[1,2], [2,3], [3,4], [1,3]]
print(eraseOverlapIntervals(intervals))  # 输出: 1（移除 [1,3]）
```

```go
// Go 实现
func eraseOverlapIntervals(intervals [][]int) int {
    if len(intervals) == 0 {
        return 0
    }

    // 按结束时间升序排序
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][1] < intervals[j][1]
    })

    count := 1
    end := intervals[0][1]

    for i := 1; i < len(intervals); i++ {
        if intervals[i][0] >= end {
            count++
            end = intervals[i][1]
        }
    }

    return len(intervals) - count
}
```

**图解过程**：

```
输入区间: [1,2], [2,3], [3,4], [1,3]

按结束时间排序: [1,2], [2,3], [1,3], [3,4]
                ↑
              选择

Step 1: 选择 [1,2]，end=2
Step 2: [2,3] 开始=2>=2，不重叠，选择 [2,3]，end=3
Step 3: [1,3] 开始=1<3，重叠，跳过
Step 4: [3,4] 开始=3>=3，不重叠，选择 [3,4]

选择: [1,2], [2,3], [3,4]  → 移除: 1个
```

**复杂度分析**：

- 时间复杂度：O(N log N)（排序主导）
- 空间复杂度：O(1)（原地排序）或 O(N)（排序额外空间）

**正确性证明（交换论证）**：

1. 设最优解 O 选择了最多不重叠区间，排序后第一个选中区间结束时间为 `e`
2. 贪心选择的是结束时间最早 `g ≤ e` 的区间
3. 若 O 第一个选的也是 `g`，同；若不是，用 `g` 替换 O 的第一个区间
4. 由于 `g` 结束更早或相等，后续区间空间只增不减 → 替换不使解变差
5. 递归对剩余区间重复，最终最优解转化为贪心解

---

### LeetCode 56 - 合并区间

> **题意**：合并所有重叠的区间。

**贪心策略**：按开始时间排序，依次合并重叠区间。

```python
# Python 实现
def merge(intervals: list[list[int]]) -> list[list[int]]:
    if not intervals:
        return []

    intervals.sort(key=lambda x: x[0])  # 按开始时间排序

    merged = [intervals[0]]  # 初始化为第一个区间

    for i in range(1, len(intervals)):
        last = merged[-1]
        curr = intervals[i]

        if curr[0] <= last[1]:  # 有重叠，合并
            last[1] = max(last[1], curr[1])  # 取结束时间的较大值
        else:
            merged.append(curr)  # 无重叠，直接加入

    return merged

# 测试
intervals = [[1,3],[2,6],[8,10],[15,18]]
print(merge(intervals))  # 输出: [[1,6], [8,10], [15,18]]
```

```go
// Go 实现
func merge(intervals [][]int) [][]int {
    if len(intervals) == 0 {
        return [][]int{}
    }

    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })

    merged := [][]int{intervals[0]}

    for i := 1; i < len(intervals); i++ {
        last := merged[len(merged)-1]
        curr := intervals[i]

        if curr[0] <= last[1] {
            // 合并：结束时间取较大值
            if curr[1] > last[1] {
                last[1] = curr[1]
            }
        } else {
            merged = append(merged, curr)
        }
    }

    return merged
}
```

**图解过程**：

```
输入: [1,3], [2,6], [8,10], [15,18]
按开始时间排序: [1,3], [2,6], [8,10], [15,18]

Step 1: merged = [[1,3]]
Step 2: [2,6] 与 [1,3] 重叠 → 合并为 [1,6]
Step 3: [8,10] 不与 [1,6] 重叠 → 加入
Step 4: [15,18] 不与 [8,10] 重叠 → 加入

结果: [[1,6], [8,10], [15,18]]
```

**复杂度分析**：

- 时间复杂度：O(N log N)
- 空间复杂度：O(N)

---

### LeetCode 452 - 用最少数量的箭引爆气球

> **题意**：在二维平面上射箭，气球在区间 `[x_start, x_end]` 上，一支箭射在 x 位置可以引爆所有 x_start ≤ x ≤ x_end 的气球。求最少箭数。

**贪心策略**：按结束位置排序，射在每个不重叠气球的结束位置。

```python
# Python 实现
def findMinArrowShots(points: list[list[int]]) -> int:
    if not points:
        return 0

    points.sort(key=lambda x: x[1])  # 按结束位置排序

    arrows = 1
    arrow_pos = points[0][1]  # 第一支箭射在第一个气球的结束位置

    for i in range(1, len(points)):
        # 如果当前气球的开始位置 > 箭的位置，需要新射一支箭
        if points[i][0] > arrow_pos:
            arrows += 1
            arrow_pos = points[i][1]  # 新箭射在新气球结束位置

    return arrows

# 测试
points = [[10,16],[2,8],[1,6],[7,12]]
print(findMinArrowShots(points))  # 输出: 2
```

```go
// Go 实现
func findMinArrowShots(points [][]int) int {
    if len(points) == 0 {
        return 0
    }

    sort.Slice(points, func(i, j int) bool {
        return points[i][1] < points[j][1]
    })

    arrows := 1
    arrowPos := points[0][1]

    for i := 1; i < len(points); i++ {
        if points[i][0] > arrowPos {
            arrows++
            arrowPos = points[i][1]
        }
    }

    return arrows
}
```

**与 LeetCode 435 的区别**：本题气球边界 `[x1, x2]` 只需要箭的 x 在闭区间内，所以边界重叠也可以一支箭解决。排序方式相同，贪心策略相同，但判断条件略有不同。

**复杂度分析**：

- 时间复杂度：O(N log N)
- 空间复杂度：O(1)

---

## 3. 跳跃游戏

### LeetCode 55 - 跳跃游戏 I（判断能否到达）

> **题意**：给定一个非负整数数组 `nums`，`nums[i]` 表示从位置 `i` 出发可以跳跃的最大长度。判断能否到达最后一个位置。

**贪心策略**：维护当前能到达的最远位置 ` farthest`，遍历数组，如果当前索引 `i` 超过了 `farthest`，说明前面任何位置都到不了 `i`，直接返回 `False`。

```python
# Python 实现
def canJump(nums: list[int]) -> bool:
    farthest = 0  # 当前能到达的最远位置

    for i in range(len(nums)):
        if i > farthest:  # 当前位置不可达
            return False
        farthest = max(farthest, i + nums[i])  # 更新最远可达位置
        if farthest >= len(nums) - 1:  # 已经能到达终点
            return True

    return True

# 测试
print(canJump([2,3,1,1,4]))  # True
print(canJump([3,2,1,0,4]))  # False
```

```go
// Go 实现
func canJump(nums []int) bool {
    farthest := 0

    for i := 0; i < len(nums); i++ {
        if i > farthest {
            return false
        }
        if i+nums[i] > farthest {
            farthest = i + nums[i]
        }
        if farthest >= len(nums)-1 {
            return true
        }
    }

    return true
}
```

**图解**：

```
nums = [2,3,1,1,4]
索引:  0  1  2  3  4

Step 0: farthest=0,  i=0<=0, farthest=max(0,0+2)=2
Step 1: farthest=2,  i=1<=2, farthest=max(2,1+3)=4 → 4>=4, 到终点，返回True

nums = [3,2,1,0,4]
Step 0: farthest=0,  i=0<=0, farthest=max(0,0+3)=3
Step 1: farthest=3,  i=1<=3, farthest=max(3,1+2)=3
Step 2: farthest=3,  i=2<=3, farthest=max(3,2+1)=3
Step 3: farthest=3,  i=3<=3, farthest=max(3,3+0)=3
Step 4: farthest=3,  i=4>3  →  不可达！返回False
```

**复杂度分析**：

- 时间复杂度：O(N)
- 空间复杂度：O(1)

---

### LeetCode 45 - 跳跃游戏 II（最少跳数）

> **题意**：同 LeetCode 55，但现在假设一定能够到达终点，求最少跳跃次数。

**贪心策略**：在当前覆盖范围内（`curEnd`），贪心选择能跳到最远位置的下一步（`nextEnd`）。当到达 `curEnd` 时，触发一次跳跃。

```python
# Python 实现
def jump(nums: list[int]) -> int:
    jumps = 0         # 已跳次数
    curEnd = 0       # 当前一步能到达的边界
    nextEnd = 0      # 下一步能到达的最远边界

    for i in range(len(nums) - 1):  # 不需要遍历到最后一个元素
        nextEnd = max(nextEnd, i + nums[i])  # 更新下一步最远

        # 到达当前一步的边界，需要再跳一步
        if i == curEnd:
            jumps += 1
            curEnd = nextEnd

    return jumps

# 测试
print(jump([2,3,1,1,4]))  # 输出: 2 (0→1→4)
```

```go
// Go 实现
func jump(nums []int) int {
    jumps := 0
    curEnd := 0
    nextEnd := 0

    for i := 0; i < len(nums)-1; i++ {
        if i+nums[i] > nextEnd {
            nextEnd = i + nums[i]
        }

        if i == curEnd {
            jumps++
            curEnd = nextEnd
        }
    }

    return jumps
}
```

**图解**：

```
nums = [2,3,1,1,4]

i=0: nextEnd = max(0, 0+2)=2, i==curEnd(0), jumps=1, curEnd=2
     覆盖范围 [1,2]，下一步能跳到最远的是 i=1（i+nums[i]=4）
i=1: nextEnd = max(2, 1+3)=4, i!=curEnd
i=2: nextEnd = max(4, 2+1)=4, i==curEnd(2), jumps=2, curEnd=4
     已到达终点，返回2

最少跳跃次数: 2
```

**复杂度分析**：

- 时间复杂度：O(N)
- 空间复杂度：O(1)

**追问**：如果不知道一定能够到达怎么办？
→ 先用 LeetCode 55 的方法判断 `canJump`，如果 `False` 则返回 `-1`。

---

## 4. 分配问题

### LeetCode 135 - 分发糖果

> **题意**：一群孩子站成一排，每个孩子有一个评分 `rating`。给每个孩子发糖果，要求：
> - 相邻两个孩子中评分高的必须获得更多糖果
> - 求最少需要多少糖果

**贪心策略**：两次扫描
1. **左到右**：保证每个孩子的评分如果比左边高，就比左边多拿
2. **右到左**：保证每个孩子的评分如果比右边高，就比右边多拿
3. 取两次的最大值

```python
# Python 实现
def candy(ratings: list[int]) -> int:
    n = len(ratings)
    if n <= 1:
        return n

    # 第一遍：左到右，评分高比左边多
    left = [1] * n
    for i in range(1, n):
        if ratings[i] > ratings[i-1]:
            left[i] = left[i-1] + 1

    # 第二遍：右到左，评分高比右边多，同时累加
    right = 0
    total = 0
    for i in range(n-1, -1, -1):
        if i < n-1 and ratings[i] > ratings[i+1]:
            right += 1
        else:
            right = 1
        total += max(left[i], right)

    return total

# 测试
ratings = [1,0,2]
print(candy(ratings))  # 输出: 5
```

```go
// Go 实现
func candy(ratings []int) int {
    n := len(ratings)
    if n <= 1 {
        return n
    }

    left := make([]int, n)
    for i := 0; i < n; i++ {
        left[i] = 1
    }

    // 左到右
    for i := 1; i < n; i++ {
        if ratings[i] > ratings[i-1] {
            left[i] = left[i-1] + 1
        }
    }

    right := 0
    total := 0

    // 右到左
    for i := n - 1; i >= 0; i-- {
        if i < n-1 && ratings[i] > ratings[i+1] {
            right++
        } else {
            right = 1
        }
        if left[i] > right {
            total += left[i]
        } else {
            total += right
        }
    }

    return total
}
```

**图解**：

```
ratings:  [1,  0,  2]
          ↓
left:     [1,  1,  2]    # 左到右扫描
          ↓
right:    [2,  1,  1]    # 右到左扫描（当前值）
max:      [2,  1,  2]    # 取 max
糖果总数: 5

解释：
- 第一个孩子评分1，右边没有孩子，但比右边评分0高，需要2个糖果
- 中间孩子评分0，左右都低，1个糖果
- 最后一个孩子评分2，比左边评分0高，2个糖果
```

**复杂度分析**：

- 时间复杂度：O(N)（两遍扫描）
- 空间复杂度：O(N)

---

### LeetCode 134 - 加油站

> **题意**：一圈环形路上有 `n` 个加油站，第 `i` 个加油站有 `gas[i]` 升油，从第 `i` 个到第 `i+1` 个消耗 `cost[i]` 升油。问从哪个加油站出发能走完一圈。

**贪心策略**：如果从 `i` 出发到达 `j` 时油量变为负，说明 `[i, j]` 之间的任何点都无法到达 `j+1`（因为在 `j` 之前就已经油量不足了）。因此，下次从 `j+1` 开始尝试。

```python
# Python 实现
def gasStation(gas: list[int], cost: list[int]) -> int:
    total_tank = 0   # 全程总油量
    curr_tank = 0    # 当前油量
    start = 0        # 起始站点

    for i in range(len(gas)):
        total_tank += gas[i] - cost[i]
        curr_tank += gas[i] - cost[i]

        # 从 start 出发，在 i 位置油量不足
        if curr_tank < 0:
            start = i + 1      # 从下一个站点重新尝试
            curr_tank = 0      # 重置当前油量

    return start if total_tank >= 0 else -1

# 测试
gas  = [1,2,3,4,5]
cost = [3,4,5,1,2]
print(gasStation(gas, cost))  # 输出: 3
```

```go
// Go 实现
func canCompleteCircuit(gas []int, cost []int) int {
    totalTank := 0
    currTank := 0
    start := 0

    for i := 0; i < len(gas); i++ {
        totalTank += gas[i] - cost[i]
        currTank += gas[i] - cost[i]

        if currTank < 0 {
            start = i + 1
            currTank = 0
        }
    }

    if totalTank >= 0 {
        return start
    }
    return -1
}
```

**图解**：

```
gas  = [1, 2, 3, 4, 5]
cost = [3, 4, 5, 1, 2]
差值 = [-2,-2,-2, 3, 3]  (gas - cost)

累积: -2 → -4 → -6 → -3 → 0  (全程总和=0，有解)

从 start=3 出发:
位置3: 油=4,  到位置4消耗1, 剩余=4-1=3
位置4: 油=5,  到位置0消耗2, 剩余=3+5-2=6
位置0: 油=1,  到位置1消耗3, 剩余=6+1-3=4
位置1: 油=2,  到位置2消耗4, 剩余=4+2-4=2
位置2: 油=3,  到位置3消耗5, 剩余=2+3-5=0 → 成功绕一圈
```

**复杂度分析**：

- 时间复杂度：O(N)
- 空间复杂度：O(1)

**为什么贪心正确？**  
设从 `i` 出发到达 `j` 时油量变为负，那么对于任意 `k`（`i ≤ k ≤ j`），从 `k` 出发时，油量 ≤ 从 `i` 出发到达 `k` 时的油量，都不足以支撑到 `j+1`。因此 `j+1` 才是唯一可能的起点。

---

## 5. 其他高频题

### LeetCode 406 - 根据身高重建队列

> **题意**：有一群人按 `(h, k)` 形式排队，`h` 是身高，`k` 是前面身高 ≥ `h` 的人数。重建队列。

**贪心策略**：
1. 按身高 **降序** 排列（高个子先排，后面的人不影响高个子的 `k` 值）
2. 按 `k` 值插入到对应位置

```python
# Python 实现
def reconstructQueue(people: list[list[int]]) -> list[list[int]]:
    # 按身高降序，k 值升序排列
    people.sort(key=lambda x: (-x[0], x[1]))

    result = []
    for p in people:
        result.insert(p[1], p)  # 按 k 值插入到对应位置

    return result

# 测试
people = [[7,1],[4,4],[7,0],[5,0],[6,1],[5,2]]
print(reconstructQueue(people))
# 输出: [[5,0], [7,0], [5,2], [6,1], [4,4], [7,1]]
```

```go
// Go 实现
func reconstructQueue(people [][]int) [][]int {
    sort.Slice(people, func(i, j int) bool {
        if people[i][0] == people[j][0] {
            return people[i][1] < people[j][1]
        }
        return people[i][0] > people[j][0]
    })

    result := make([][]int, 0, len(people))
    for _, p := range people {
        idx := p[1]
        // 在 idx 位置插入
        result = append(result, []int{})
        copy(result[idx+1:], result[idx:])
        result[idx] = p
    }

    return result
}
```

**图解**：

```
输入: [[7,1], [4,4], [7,0], [5,0], [6,1], [5,2]]
排序后(身高降序, k升序): [[7,0], [7,1], [6,1], [5,0], [5,2], [4,4]]

插入过程:
[7,0] → k=0 → [[7,0]]
[7,1] → k=1 → [[7,0], [7,1]]
[6,1] → k=1 → [[7,0], [6,1], [7,1]]
[5,0] → k=0 → [[5,0], [7,0], [6,1], [7,1]]
[5,2] → k=2 → [[5,0], [7,0], [5,2], [6,1], [7,1]]
[4,4] → k=4 → [[5,0], [7,0], [5,2], [6,1], [4,4], [7,1]]
```

**复杂度分析**：

- 时间复杂度：O(N²)（插入操作 O(N)，共 N 次）
- 空间复杂度：O(N)

---

### LeetCode 968 - 监控二叉树（贪心 + DFS）

> **题意**：在二叉树上放置摄像头，每个摄像头可以监控父节点、本身和子节点。求监控所有节点所需的最小摄像头数量。

**贪心策略（从底向上 DFS）**：
- 贪心选择：叶节点的**父节点**优先放置摄像头
- 这样每个摄像头能覆盖最多节点（叶节点的父节点可以同时监控两个叶节点）

```python
# Python 实现
# 状态: 0=未被覆盖, 1=被摄像头覆盖, 2=需要摄像头
class TreeNode:
    def __init__(self, x):
        self.val = x
        self.left = None
        self.right = None

def minCameraCover(root: TreeNode) -> int:
    result = 0

    def dfs(node):
        nonlocal result
        if not node:
            return 1  # 空节点视为被覆盖（不需要摄像头）

        left = dfs(node.left)
        right = dfs(node.right)

        # 如果左右子树有一个未被覆盖，当前节点需要放摄像头
        if left == 0 or right == 0:
            result += 1
            return 1  # 当前节点放摄像头，变为被覆盖状态

        # 如果左右子树都被覆盖，且没有摄像头，当前节点需要摄像头
        if left == 1 and right == 1:
            return 0  # 当前节点未被覆盖，交给父节点处理

        # 如果左右子树有一个有摄像头，当前节点被覆盖
        if left == 2 or right == 2:
            return 1  # 当前节点被覆盖

        return 0

    # 根节点未被覆盖，需要在根放摄像头
    if dfs(root) == 0:
        result += 1

    return result

# 测试（手动构造树）
#       0
#     /   \
#    0     0
#  /\     /\
# 0  0   0  0
```

```go
// Go 实现
func minCameraCover(root *TreeNode) int {
    result := 0

    var dfs func(node *TreeNode) int
    dfs = func(node *TreeNode) int {
        if node == nil {
            return 2 // 视为被覆盖，不需要摄像头
        }

        left := dfs(node.Left)
        right := dfs(node.Right)

        // 左右子树有一个未覆盖，当前节点必须放摄像头
        if left == 0 || right == 0 {
            result++
            return 1 // 当前节点有摄像头，被覆盖
        }

        // 左右都被覆盖（且无摄像头），当前节点未覆盖
        if left == 1 && right == 1 {
            return 0
        }

        // 至少一个子树有摄像头，当前节点被覆盖
        if left == 2 || right == 2 {
            return 1
        }

        return 0
    }

    if dfs(root) == 0 {
        result++
    }

    return result
}

// TreeNode 定义
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}
```

**贪心原理**：把摄像头放在叶节点的父节点，可以同时覆盖两个叶节点（一举两得）。如果放在叶节点本身，只能覆盖一个节点和它的父节点（利用率低）。

**复杂度分析**：

- 时间复杂度：O(N)
- 空间复杂度：O(H)，H 为树高

---

## 6. 面试答题框架

当面试官问一道贪心题时，按以下结构回答：

### Step 1：识别贪心（30秒）

> "这道题可以尝试贪心，因为..."
> - 题目问的是"最优"（最少、最大、最短...）
> - 每一步的选择是独立的，不影响之前的选择
> - 存在一种"局部最优"的判断标准

### Step 2：描述贪心策略（1分钟）

> "我的策略是：每次选择 XXX，因为..."
> - 具体说明排序标准或选择标准
> - 解释为什么这个局部最优可能带来全局最优

### Step 3：证明正确性（1分钟）

> "我可以用交换论证法证明..."
> （参考上面的证明步骤）

### Step 4：写代码（5-10分钟）

> 边写边解释："我先按结束时间排序，然后遍历..."
> 注意边界条件处理

### Step 5：分析复杂度（30秒）

> "时间复杂度 O(N log N)，因为需要排序；空间复杂度 O(1)（如果原地排序）"

---

## 高频追问

### Q1：贪心算法一定能得到最优解吗？什么情况下贪心会失败？

**不一定**。贪心失效的典型场景：**存在"后效性"**——当前选择会影响未来的选项，且未来可能出现比当前更好的选择。

典型失败案例：**货币找零问题**  
> 假设人民币纸币面额为 [1, 3, 4]，要找 6 元。  
> 贪心（每次选最大）→ 4 + 1 + 1 = 3张  
> 最优解 → 3 + 3 = 2张  
> **失败原因**：选了 4 导致后面只能用 1 凑，余数变多。

**判断方法**：尝试反例，如果能轻易找到一个反例，说明这道题不适合贪心，应该用 DP。

---

### Q2：区间调度问题为什么按结束时间排序而不是开始时间？

**按开始时间排序可能选错**，因为早开始不一定早结束：

```
区间 A: [1, 100]  开始最早，结束最晚
区间 B: [2, 3]    开始晚，但结束早

按开始排序: [1,100], [2,3] → 只选1个
按结束排序: [2,3], [1,100] → 可选2个
```

**按结束时间排序的本质**：每次选最早结束的，就是给后面留最多空间。**结束时间才是资源竞争的关键节点**。

---

### Q3：什么时候用贪心，什么时候用 DP？

| 问题特征 | 倾向算法 |
|----------|----------|
| 选择只影响当前，不影响未来 | 贪心 |
| 需要"后悔"机制（可能推翻之前选择） | DP |
| 求最大值/最小值计数 | DP |
| 求存在性（能否/是否） | 贪心 或 二分 |
| 组合爆炸，无后效性 | DP |
| 单调最优子结构，选择唯一 | 贪心 |

> **一个经验法则**：如果一个问题你可以用一句话描述"每步选 XXX 最好"，且找不到明显反例 → 贪心优先。如果找不到简单贪心策略，或者能轻松举出反例 → DP。

---

## 关联阅读

- [📁 动态规划高频题](./08-dp.md)（对比学习）
- [📁 二分查找高频题](./04-binary-search.md)（区间问题的另一个角度）
