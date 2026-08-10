# 20. 最长递增子序列（Longest Increasing Subsequence）

> 频率：★★★★☆  难度：中等  LeetCode 300

## 题目描述
给你一个整数数组 `nums`，找到其中最长严格递增子序列的长度。

## 小白先理解
注意：
- 这里是 **子序列**，不要求连续
- 只要求前后顺序不变

比如：

```text
[10,9,2,5,3,7,101,18]
```

最长递增子序列是：

```text
[2,3,7,101]
```

长度是 `4`。

---

## 解法一：DP

### 核心思路
定义：
`dp[i]` 表示以 `nums[i]` 结尾的最长递增子序列长度。

转移：
- 看前面所有 `j < i`
- 如果 `nums[j] < nums[i]`，可以接在后面

### Go
```go
func lengthOfLISDP(nums []int) int {
    n := len(nums)
    dp := make([]int, n)
    ans := 0

    for i := 0; i < n; i++ {
        dp[i] = 1
        for j := 0; j < i; j++ {
            if nums[j] < nums[i] && dp[j]+1 > dp[i] {
                dp[i] = dp[j] + 1
            }
        }
        if dp[i] > ans {
            ans = dp[i]
        }
    }
    return ans
}
```

### Python
```python
def length_of_lis_dp(nums):
    n = len(nums)
    dp = [1] * n
    ans = 0

    for i in range(n):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)
        ans = max(ans, dp[i])
    return ans
```

### 复杂度
- 时间：`O(n^2)`

---

## 解法二：贪心 + 二分（推荐）

### 核心思路
维护一个数组 `tails`：
- `tails[i]` 表示长度为 `i+1` 的递增子序列中，结尾最小是多少

遍历每个数：
- 用二分找到它应该放进 `tails` 的位置
- 替换或追加

`tails` 长度就是答案。

### 为什么这样对
结尾越小，后面越容易接更大的数。
所以我们总想让同样长度的递增子序列，结尾尽量小。

### Go
```go
import "sort"

func lengthOfLIS(nums []int) int {
    tails := []int{}
    for _, num := range nums {
        i := sort.SearchInts(tails, num)
        if i == len(tails) {
            tails = append(tails, num)
        } else {
            tails[i] = num
        }
    }
    return len(tails)
}
```

### Python
```python
import bisect

def length_of_lis(nums):
    tails = []
    for num in nums:
        idx = bisect.bisect_left(tails, num)
        if idx == len(tails):
            tails.append(num)
        else:
            tails[idx] = num
    return len(tails)
```

## 一句话记忆
**LIS 两种思路：朴素 DP，进阶是贪心 + 二分。**
