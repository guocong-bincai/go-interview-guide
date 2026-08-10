# 28. 单词拆分（Word Break）

> 频率：★★★★☆  难度：中等  LeetCode 139

## 小白先理解
问的是：一个字符串能不能拆成若干个字典里的单词。
比如 `leetcode` 能拆成 `leet + code`。

## 解法一：递归暴力
- 从前往后试每个切分点
- 重复计算很多

## 解法二：动态规划（推荐）
### Go
```go
func wordBreak(s string, wordDict []string) bool {
    wordSet := map[string]bool{}
    for _, w := range wordDict { wordSet[w] = true }
    dp := make([]bool, len(s)+1)
    dp[0] = true
    for i := 1; i <= len(s); i++ {
        for j := 0; j < i; j++ {
            if dp[j] && wordSet[s[j:i]] {
                dp[i] = true
                break
            }
        }
    }
    return dp[len(s)]
}
```
### Python
```python
def word_break(s, wordDict):
    word_set = set(wordDict)
    dp = [False] * (len(s) + 1)
    dp[0] = True
    for i in range(1, len(s) + 1):
        for j in range(i):
            if dp[j] and s[j:i] in word_set:
                dp[i] = True
                break
    return dp[-1]
```

## 一句话记忆
**单词拆分 = 前缀 DP。**
