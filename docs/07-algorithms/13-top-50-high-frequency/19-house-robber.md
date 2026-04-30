# 19. 打家劫舍（House Robber）

> 频率：★★★★☆  难度：中等  LeetCode 198

## 题目描述
你是一个专业小偷，计划偷窃沿街的房屋。每间房内都有一定现金，
相邻的房屋装有相互连通的防盗系统，如果两间相邻的房屋在同一晚上被偷，系统会自动报警。

给定一个整数数组 `nums`，表示每间房屋存放的金额，计算你不触动警报装置的情况下，一夜之内能够偷窃到的最高金额。

## 小白先理解
每间房只有两个选择：
- 偷
- 不偷

但如果偷当前这间，就不能偷前一间。

所以这题本质是一个“选或不选”的动态规划。

---

## 解法一：递归搜索

### 思路
对于第 i 间房：
- 偷它：`nums[i] + dfs(i-2)`
- 不偷它：`dfs(i-1)`

取最大值。

### 问题
会重复计算很多次。

---

## 解法二：动态规划（推荐）

### 核心思路
定义：
`dp[i]` = 偷到第 i 间房时的最大金额

状态转移：
```text
dp[i] = max(dp[i-1], dp[i-2] + nums[i])
```

解释：
- 不偷第 i 间：`dp[i-1]`
- 偷第 i 间：`dp[i-2] + nums[i]`

### Go
```go
func rob(nums []int) int {
    n := len(nums)
    if n == 0 {
        return 0
    }
    if n == 1 {
        return nums[0]
    }

    dp := make([]int, n)
    dp[0] = nums[0]
    if nums[1] > nums[0] {
        dp[1] = nums[1]
    } else {
        dp[1] = nums[0]
    }

    for i := 2; i < n; i++ {
        if dp[i-1] > dp[i-2]+nums[i] {
            dp[i] = dp[i-1]
        } else {
            dp[i] = dp[i-2] + nums[i]
        }
    }

    return dp[n-1]
}
```

### Python
```python
def rob(nums):
    if not nums:
        return 0
    if len(nums) == 1:
        return nums[0]

    dp = [0] * len(nums)
    dp[0] = nums[0]
    dp[1] = max(nums[0], nums[1])

    for i in range(2, len(nums)):
        dp[i] = max(dp[i - 1], dp[i - 2] + nums[i])

    return dp[-1]
```

## 一句话记忆
**打家劫舍就是经典 DP：偷当前 or 不偷当前。**
