# 45. 零钱兑换（Coin Change）

> 频率：★★★★☆  难度：中等  LeetCode 322

## 小白先理解
给你若干硬币面值，每种硬币数量无限，问最少需要多少枚硬币，才能凑出目标金额。

## 解法一：暴力递归
- 每种硬币都尝试
- 重复子问题很多

## 解法二：动态规划（推荐）
### 核心思路
`dp[i]` 表示凑出金额 i 的最少硬币数。

状态转移：
```text
dp[i] = min(dp[i], dp[i-coin] + 1)
```

### Go
```go
func coinChange(coins []int, amount int) int {
    dp := make([]int, amount+1)
    for i := 1; i <= amount; i++ { dp[i] = amount + 1 }
    for i := 1; i <= amount; i++ {
        for _, coin := range coins {
            if i >= coin && dp[i-coin]+1 < dp[i] {
                dp[i] = dp[i-coin] + 1
            }
        }
    }
    if dp[amount] > amount { return -1 }
    return dp[amount]
}
```
### Python
```python
def coin_change(coins, amount):
    dp = [amount + 1] * (amount + 1)
    dp[0] = 0
    for i in range(1, amount + 1):
        for coin in coins:
            if i >= coin:
                dp[i] = min(dp[i], dp[i - coin] + 1)
    return -1 if dp[amount] > amount else dp[amount]
```

## 一句话记忆
**零钱兑换 = 完全背包型 DP。**
