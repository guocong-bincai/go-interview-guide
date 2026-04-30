# 12. 合并区间（Merge Intervals）

> 频率：★★★★★  难度：中等  LeetCode 56

## 题目描述
给出若干区间 `intervals`，请合并所有重叠的区间。

## 小白先理解
比如：

```text
[[1,3],[2,6],[8,10],[15,18]]
```

`[1,3]` 和 `[2,6]` 重叠，所以可以合并成 `[1,6]`。

答案是：

```text
[[1,6],[8,10],[15,18]]
```

---

## 解法一：暴力反复合并

### 思路
不断检查区间两两之间是否能合并，直到不能再合并。

### 问题
实现麻烦，而且效率低。

---

## 解法二：排序后线性合并（推荐）

### 核心思路
先按区间起点排序。

排序后：
- 如果当前区间和结果最后一个区间重叠，就合并
- 否则直接加入结果

### Go
```go
import "sort"

func merge(intervals [][]int) [][]int {
    if len(intervals) == 0 {
        return nil
    }

    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })

    res := [][]int{intervals[0]}
    for i := 1; i < len(intervals); i++ {
        last := res[len(res)-1]
        cur := intervals[i]
        if cur[0] <= last[1] {
            if cur[1] > last[1] {
                last[1] = cur[1]
            }
        } else {
            res = append(res, cur)
        }
    }
    return res
}
```

### Python
```python
def merge(intervals):
    intervals.sort(key=lambda x: x[0])
    res = [intervals[0]]

    for cur in intervals[1:]:
        last = res[-1]
        if cur[0] <= last[1]:
            last[1] = max(last[1], cur[1])
        else:
            res.append(cur)
    return res
```

## 一句话记忆
**区间题先排序，再看能不能和前一个合并。**
