# 13. 无重叠区间（Non-overlapping Intervals）

> 频率：★★★★★  难度：中等  LeetCode 435

## 题目描述
给定一个区间集合 `intervals`，返回需要移除区间的最小数量，使剩余区间互不重叠。

## 小白先理解
题目不是问你“保留哪些区间”，而是问：
**最少删几个，才能让区间之间不打架。**

这类题通常反过来想更简单：
- 与其想删谁
- 不如想“最多能保留多少个不重叠区间”

---

## 解法一：DP

### 思路
定义 `dp[i]` 表示以第 i 个区间结尾时，最多能保留多少个不重叠区间。

### 问题
能做，但不是最优。

---

## 解法二：贪心（推荐）

### 核心思路
按区间结束位置排序。

为什么按结束位置？
因为结束越早，越不影响后面留更多区间。

步骤：
1. 按结束位置排序
2. 先保留第一个
3. 遇到新区间时：
   - 如果和当前已保留区间不重叠，就保留
   - 否则跳过它
4. 最后总数减去保留数，就是最少删除数

### Go
```go
import "sort"

func eraseOverlapIntervals(intervals [][]int) int {
    if len(intervals) == 0 {
        return 0
    }

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

### Python
```python
def erase_overlap_intervals(intervals):
    if not intervals:
        return 0

    intervals.sort(key=lambda x: x[1])
    count = 1
    end = intervals[0][1]

    for i in range(1, len(intervals)):
        if intervals[i][0] >= end:
            count += 1
            end = intervals[i][1]

    return len(intervals) - count
```

## 一句话记忆
**区间不重叠问题，优先按结束位置排序做贪心。**
