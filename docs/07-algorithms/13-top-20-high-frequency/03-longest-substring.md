# 03. 最长不重复子串（Longest Substring Without Repeating Characters）

> 频率：★★★★★  难度：中等  LeetCode 3

## 题目描述
给定一个字符串 `s`，请你找出其中 **不含有重复字符的最长子串** 的长度。

## 小白先理解题意
注意：这里说的是 **子串**，不是子序列。

子串要求：
- 必须连续

比如：
```text
s = "abcabcbb"
```

最长不重复子串是：`"abc"`
长度是 `3`。

---

## 面试为什么爱问
- 经典滑动窗口题
- 可以看出你会不会“边走边维护区间”
- 很适合追问优化思路

## 解法一：暴力枚举

### 思路
把所有子串都找出来，然后检查有没有重复字符。

### Go
```go
func lengthOfLongestSubstringBruteForce(s string) int {
    ans := 0
    for i := 0; i < len(s); i++ {
        seen := map[byte]bool{}
        for j := i; j < len(s); j++ {
            if seen[s[j]] {
                break
            }
            seen[s[j]] = true
            if j-i+1 > ans {
                ans = j - i + 1
            }
        }
    }
    return ans
}
```

### Python
```python
def length_of_longest_substring_bruteforce(s: str) -> int:
    ans = 0
    for i in range(len(s)):
        seen = set()
        for j in range(i, len(s)):
            if s[j] in seen:
                break
            seen.add(s[j])
            ans = max(ans, j - i + 1)
    return ans
```

### 复杂度
- 时间：`O(n^2)`
- 空间：`O(n)`

---

## 解法二：滑动窗口（推荐）

### 核心思路
用一个窗口 `[left, right]` 表示当前“没有重复字符”的区间。

- 右指针不断往右扩
- 如果发现重复字符，就移动左指针，直到窗口重新合法

### 例子
`s = "abba"`

- 先看 `a`，窗口 = `a`
- 再看 `b`，窗口 = `ab`
- 再看 `b`，重复了，就移动左指针，直到窗口不重复

### Go
```go
func lengthOfLongestSubstring(s string) int {
    window := map[byte]int{}
    left, ans := 0, 0

    for right := 0; right < len(s); right++ {
        ch := s[right]
        window[ch]++

        for window[ch] > 1 {
            window[s[left]]--
            left++
        }

        if right-left+1 > ans {
            ans = right - left + 1
        }
    }
    return ans
}
```

### Python
```python
def length_of_longest_substring(s: str) -> int:
    window = {}
    left = 0
    ans = 0

    for right, ch in enumerate(s):
        window[ch] = window.get(ch, 0) + 1
        while window[ch] > 1:
            window[s[left]] -= 1
            left += 1
        ans = max(ans, right - left + 1)

    return ans
```

### 复杂度
- 时间：`O(n)`
- 空间：`O(n)`

---

## 一句话记忆
**看到“连续 + 不重复”，优先想滑动窗口。**
