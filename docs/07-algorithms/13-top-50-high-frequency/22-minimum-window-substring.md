# 22. 最小覆盖子串（Minimum Window Substring）

> 频率：★★★★☆  难度：困难  LeetCode 76

## 小白先理解
在字符串 `s` 里，找一个最短的连续子串，这个子串必须把 `t` 里的所有字符都包含进去。
这是滑动窗口经典题。

## 解法一：暴力枚举所有子串
- 枚举起点终点
- 判断是否包含 t
- 很慢

## 解法二：滑动窗口（推荐）
### Go
```go
func minWindow(s string, t string) string {
    need := map[byte]int{}
    for i := 0; i < len(t); i++ { need[t[i]]++ }
    window := map[byte]int{}
    left, valid, start, length := 0, 0, 0, 1<<30
    for right := 0; right < len(s); right++ {
        c := s[right]
        if _, ok := need[c]; ok {
            window[c]++
            if window[c] == need[c] { valid++ }
        }
        for valid == len(need) {
            if right-left+1 < length { start, length = left, right-left+1 }
            d := s[left]
            left++
            if _, ok := need[d]; ok {
                if window[d] == need[d] { valid-- }
                window[d]--
            }
        }
    }
    if length == 1<<30 { return "" }
    return s[start:start+length]
}
```
### Python
```python
def min_window(s, t):
    need = {}
    for ch in t:
        need[ch] = need.get(ch, 0) + 1
    window = {}
    left = valid = 0
    start = 0
    length = float('inf')
    for right, ch in enumerate(s):
        if ch in need:
            window[ch] = window.get(ch, 0) + 1
            if window[ch] == need[ch]:
                valid += 1
        while valid == len(need):
            if right - left + 1 < length:
                start, length = left, right - left + 1
            d = s[left]
            left += 1
            if d in need:
                if window[d] == need[d]:
                    valid -= 1
                window[d] -= 1
    return '' if length == float('inf') else s[start:start+length]
```

## 一句话记忆
**最小覆盖子串 = 滑动窗口先扩张满足条件，再收缩找最短。**
