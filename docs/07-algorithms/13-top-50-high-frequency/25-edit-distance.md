# 25. 编辑距离（Edit Distance）

> 频率：★★★★☆  难度：困难  LeetCode 72

## 小白先理解
把单词 A 变成单词 B，最少需要几步。
允许操作：插入、删除、替换。

## 解法一：递归搜索
- 每一步尝试三种操作
- 会大量重复计算

## 解法二：动态规划（推荐）
### Go
```go
func minDistance(word1 string, word2 string) int {
    m, n := len(word1), len(word2)
    dp := make([][]int, m+1)
    for i := range dp { dp[i] = make([]int, n+1) }
    for i := 0; i <= m; i++ { dp[i][0] = i }
    for j := 0; j <= n; j++ { dp[0][j] = j }
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if word1[i-1] == word2[j-1] {
                dp[i][j] = dp[i-1][j-1]
            } else {
                dp[i][j] = min3(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]) + 1
            }
        }
    }
    return dp[m][n]
}
func min3(a,b,c int) int { if a>b { a=b }; if a>c { a=c }; return a }
```
### Python
```python
def min_distance(word1, word2):
    m, n = len(word1), len(word2)
    dp = [[0]*(n+1) for _ in range(m+1)]
    for i in range(m+1): dp[i][0] = i
    for j in range(n+1): dp[0][j] = j
    for i in range(1, m+1):
        for j in range(1, n+1):
            if word1[i-1] == word2[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]) + 1
    return dp[m][n]
```

## 一句话记忆
**编辑距离 = 三种操作的 DP。**
