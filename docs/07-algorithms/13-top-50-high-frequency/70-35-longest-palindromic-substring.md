# 35. 最长回文子串（Longest Palindromic Substring）

> 频率：★★★★☆  难度：中等  LeetCode 5

## 小白先理解
回文就是正着读和反着读一样。

比如：
- `aba`
- `abba`

注意奇数长度和偶数长度都可能是回文。

## 解法一：动态规划
- 判断 `s[i:j]` 是否为回文
- 能做，但写起来稍复杂

## 解法二：中心扩展（推荐）
### 核心思路
每个回文都有一个“中心”。
- 奇数回文：中心是一个字符
- 偶数回文：中心是两个字符之间

从中心往两边扩，看能扩多远。

### Go
```go
func longestPalindrome(s string) string {
    if len(s) < 2 { return s }
    start, end := 0, 0
    var expand func(int, int) (int, int)
    expand = func(l, r int) (int, int) {
        for l >= 0 && r < len(s) && s[l] == s[r] {
            l--
            r++
        }
        return l + 1, r - 1
    }
    for i := 0; i < len(s); i++ {
        l1, r1 := expand(i, i)
        l2, r2 := expand(i, i+1)
        if r1-l1 > end-start { start, end = l1, r1 }
        if r2-l2 > end-start { start, end = l2, r2 }
    }
    return s[start:end+1]
}
```
### Python
```python
def longest_palindrome(s):
    if len(s) < 2:
        return s
    start = end = 0
    def expand(l, r):
        while l >= 0 and r < len(s) and s[l] == s[r]:
            l -= 1
            r += 1
        return l + 1, r - 1
    for i in range(len(s)):
        l1, r1 = expand(i, i)
        l2, r2 = expand(i, i + 1)
        if r1 - l1 > end - start:
            start, end = l1, r1
        if r2 - l2 > end - start:
            start, end = l2, r2
    return s[start:end+1]
```

## 一句话记忆
**最长回文子串 = 以每个位置为中心向两边扩。**
