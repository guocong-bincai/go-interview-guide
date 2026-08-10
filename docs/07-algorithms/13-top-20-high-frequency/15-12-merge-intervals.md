# 12. 合并区间（Merge Intervals）

> 频率：★★★★★  难度：中等  LeetCode 56

## 题目描述
给出若干区间 `intervals`，请合并所有重叠的区间。

## 面试为什么爱问
- 区间合并是高频综合题，考察排序和双指针的综合运用
- 面试官想看候选人对 **边界条件** 和 **贪心思想** 的掌握
- 还能延伸追问：插入区间、会议室、最少箭矢射气球等问题

## 核心考点
- 排序：按起点排序是区间题的标准预处理
- 双指针：维护一个结果区间，和当前区间比较
- 边界条件：起点/终点重叠的判断、区间为空时返回 nil

## 小白先理解
比如：

```text
[[1,3],[2,6],[8,10],[15,18]]
```

`[1,3]` 和 `[2,6]` 重叠（因为 `2 <= 3`），所以可以合并成 `[1,6]`。
`[8,10]` 和 `[15,18]` 不重叠（`15 > 6`）。

答案是：

```text
[[1,6],[8,10],[15,18]]
```

重点：**先把所有区间按起点排好序**，然后从头扫一遍，看能不能和前一个区间合并。

---

## 解法一：暴力反复合并

### 思路
不断检查区间两两之间是否能合并，合并后更新，再继续检查，直到没有区间可以合并。

### 优点
- 最容易想到，模拟人工合并过程

### 缺点
- 需要反复扫描，时间复杂度高
- 实现麻烦，容易写错

### Go
```go
func merge暴力(intervals [][]int) [][]int {
    merged := intervals
    changed := true
    for changed {
        changed = false
        newMerged := [][]int{}
        i := 0
        for i < len(merged) {
            mergedStart, mergedEnd := merged[i][0], merged[i][1]
            j := i + 1
            for j < len(merged) && merged[j][0] <= mergedEnd {
                mergedEnd = max(mergedEnd, merged[j][1])
                j++
                changed = true
            }
            newMerged = append(newMerged, []int{mergedStart, mergedEnd})
            i = j
        }
        merged = newMerged
    }
    return merged
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

### Python
```python
def merge_bruteforce(intervals):
    merged = intervals
    changed = True
    while changed:
        changed = False
        new_merged = []
        i = 0
        while i < len(merged):
            start, end = merged[i]
            j = i + 1
            while j < len(merged) and merged[j][0] <= end:
                end = max(end, merged[j][1])
                j += 1
                changed = True
            new_merged.append([start, end])
            i = j
        merged = new_merged
    return merged
```

### 复杂度
- 时间：`O(n²)`（每次合并可能遍历所有区间，最坏情况需要 O(n²) 次合并）
- 空间：`O(n)`（每次合并创建新数组）

---

## 解法二：排序后线性合并（推荐）

### 核心思路
1. 先按区间起点排序
2. 排序后，依次遍历：
   - 如果当前区间的起点 ≤ 上一个合并区间的终点 → 重叠，合并（更新终点为 max）
   - 否则 → 不重叠，直接将当前区间加入结果

### 关键点
- 为什么按起点排序有效？因为排序后，所有可能和当前区间重叠的区间都在它**后面**，所以只需要和**上一个**合并区间比较就够了

### Go
```go
import "sort"

func merge(intervals [][]int) [][]int {
    if len(intervals) == 0 {
        return nil
    }

    // 1. 按起点升序排序
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })

    // 2. 线性扫描，依次合并
    res := [][]int{intervals[0]}
    for i := 1; i < len(intervals); i++ {
        last := res[len(res)-1] // 结果中最后一个区间
        cur := intervals[i]    // 当前待处理区间

        // 重叠：当前起点 ≤ 上一个的终点
        if cur[0] <= last[1] {
            // 合并：终点取较大值
            if cur[1] > last[1] {
                last[1] = cur[1]
            }
        } else {
            // 不重叠，直接加入
            res = append(res, cur)
        }
    }
    return res
}
```

### Python
```python
def merge(intervals):
    if not intervals:
        return []

    # 1. 按起点排序
    intervals.sort(key=lambda x: x[0])

    # 2. 线性合并
    res = [intervals[0]]
    for cur in intervals[1:]:
        last = res[-1]
        if cur[0] <= last[1]:       # 重叠
            last[1] = max(last[1], cur[1])
        else:                        # 不重叠
            res.append(cur)
    return res
```

### 复杂度
- 时间：`O(n log n)`（排序是主导复杂度）
- 空间：`O(1)`（原地排序 + 结果数组，不计结果空间）

---

## 两种解法对比
| 维度 | 解法一（暴力） | 解法二（排序贪心） |
|------|--------------|------------------|
| 核心思想 | 反复扫描直到稳定 | 先排序再贪心合并 |
| 时间复杂度 | `O(n²)` | `O(n log n)` |
| 空间复杂度 | `O(n)` | `O(1)`（不计结果） |
| 面试推荐程度 | 了解即可 | **强烈推荐** |

---

## 易错点
- **空输入**：`len(intervals) == 0` 要先返回 nil，否则 `[0][0]` 会 panic
- **区间只有一个**：`sort.Slice` 没问题，但 scan loop 要正确处理 `i < len(intervals)`
- **重叠判断**：`cur[0] <= last[1]` 是 `<=` 而不是 `<`，因为 `起点 == 终点` 也算重叠（如 `[1,1]` 和 `[1,2]`）
- **合并后端点**：取 `max(last[1], cur[1])` 而不是直接用 `cur[1]`，因为可能 cur 完全被 last 包含

---

## 面试追问
**Q1：如果要求返回合并后区间的个数（不返回具体区间）呢？**
> 一样思路，只是返回 `len(res)`，不返回 res 数组本身。空间可降到 `O(1)`。

**Q2：如果要插入一个新区间 `newInterval`，怎么处理？**
> 三步：1. 把 `newInterval` 加入 intervals；2. 排序；3. 用同样方法合并。但更优做法是直接遍历，维护三个列表：已合并区间、与之重叠的待合并区间、不重叠的区间。

**Q3：这个问题和"最少箭矢射气球（LeetCode 452）"有什么关系？**
> 本质相同。都是先按起点排序，然后判断重叠。区别在于气球题要求的是不重叠区间的个数（即箭数），但解法完全一样。

**Q4：如果有 1000 万个区间，无法全部加载到内存，怎么处理？**
> 外排序思路：分批读入、排好序写文件、归并合并。但面试中提一下外排序思路即可，不需要写出代码。

---

## 一句话记忆
**区间题先按起点排序，再看能不能和前一个合并。**
[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [⚙️ 高频面经](../../README.md)
