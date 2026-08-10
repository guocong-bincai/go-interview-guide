# 41. 滑动窗口最大值（Sliding Window Maximum）

> 频率：★★★★☆  难度：困难  LeetCode 239

## 小白先理解
给你一个窗口大小 `k`，窗口在数组上从左往右滑动。
每次滑动后，都要知道窗口里的最大值。

如果每次都重新遍历窗口找最大值，会很慢。

## 解法一：暴力枚举
- 每个窗口都扫一遍
- 时间复杂度高

## 解法二：单调队列（推荐）
### 核心思路
维护一个“单调递减”的队列，队首永远是当前窗口最大值。

### Go
```go
func maxSlidingWindow(nums []int, k int) []int {
    q := []int{}
    res := []int{}
    for i := 0; i < len(nums); i++ {
        if len(q) > 0 && q[0] <= i-k {
            q = q[1:]
        }
        for len(q) > 0 && nums[q[len(q)-1]] <= nums[i] {
            q = q[:len(q)-1]
        }
        q = append(q, i)
        if i >= k-1 {
            res = append(res, nums[q[0]])
        }
    }
    return res
}
```
### Python
```python
from collections import deque

def max_sliding_window(nums, k):
    q = deque()
    res = []
    for i, num in enumerate(nums):
        if q and q[0] <= i - k:
            q.popleft()
        while q and nums[q[-1]] <= num:
            q.pop()
        q.append(i)
        if i >= k - 1:
            res.append(nums[q[0]])
    return res
```

## 一句话记忆
**滑动窗口最大值 = 单调队列。**
