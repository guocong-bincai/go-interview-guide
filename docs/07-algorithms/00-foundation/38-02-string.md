# 字符串高频题

> 考察频率：★★★★☆  难度：简单~中等  Go + Python 双语

---

## 题目导航

| # | 题目 | 难度 | 考察频率 |
|---|------|------|---------|
| 1 | [反转字符串](#1-反转字符串-leetcode-344) | 简单 | ★★★★★ |
| 2 | [有效回文串](#2-有效回文串-leetcode-125) | 简单 | ★★★★★ |
| 3 | [最长公共前缀](#3-最长公共前缀-leetcode-14) | 简单 | ★★★★☆ |
| 4 | [字符串转整数（atoi）](#4-字符串转整数-leetcode-8) | 中等 | ★★★★☆ |
| 5 | [最长回文子串](#5-最长回文子串-leetcode-5) | 中等 | ★★★★☆ |

---

## 1. 反转字符串（LeetCode 344）

### 题意
原地反转字符数组。

**例子：** `['h','e','l','l','o']` → `['o','l','l','e','h']`

### 思路讲解
**双指针对撞**：左指针从头，右指针从尾，交换后同时向中间走，直到相遇。

```
[h, e, l, l, o]
 ↑              ↑
left          right

第1次：swap(h, o) → [o, e, l, l, h]，left++, right--
第2次：swap(e, l) → [o, l, l, e, h]，left++, right--
left >= right，结束
```

**时间复杂度：O(n)**，空间复杂度：O(1)

### Go 代码

```go
func reverseString(s []byte) {
    left, right := 0, len(s)-1
    for left < right {
        s[left], s[right] = s[right], s[left]
        left++
        right--
    }
}
```

### Python 代码

```python
def reverseString(s: list[str]) -> None:
    left, right = 0, len(s) - 1
    while left < right:
        s[left], s[right] = s[right], s[left]
        left += 1
        right -= 1
```

---

## 2. 有效回文串（LeetCode 125）

### 题意
给定字符串，只保留字母和数字，忽略大小写，判断是否是回文串。

**例子：** `"A man, a plan, a canal: Panama"` → `true`

### 思路讲解
**过滤 + 双指针**：
左右指针各自跳过非字母/数字字符，比较有效字符是否相同。

```
"A man, a plan, a canal: Panama"

left=0 'A', right=29 'a' → 忽略大小写，A==a ✓
left=1 ' ' → 跳过，left=2 'm'
right=28 'm' → m==m ✓
... 继续直到 left >= right
```

### Go 代码

```go
func isPalindrome(s string) bool {
    left, right := 0, len(s)-1
    for left < right {
        // 跳过非字母数字字符
        for left < right && !isAlphanumeric(s[left]) {
            left++
        }
        for left < right && !isAlphanumeric(s[right]) {
            right--
        }
        // 忽略大小写比较
        if toLower(s[left]) != toLower(s[right]) {
            return false
        }
        left++
        right--
    }
    return true
}

func isAlphanumeric(c byte) bool {
    return (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9')
}

func toLower(c byte) byte {
    if c >= 'A' && c <= 'Z' {
        return c + 32
    }
    return c
}
```

### Python 代码

```python
def isPalindrome(s: str) -> bool:
    # Python 一行版（面试时可以这么写）
    filtered = [c.lower() for c in s if c.isalnum()]
    return filtered == filtered[::-1]
```

```python
# 双指针版（展示思路更清晰）
def isPalindrome(s: str) -> bool:
    left, right = 0, len(s) - 1
    while left < right:
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1
        if s[left].lower() != s[right].lower():
            return False
        left += 1
        right -= 1
    return True
```

---

## 3. 最长公共前缀（LeetCode 14）

### 题意
找出字符串数组中所有字符串的最长公共前缀。

**例子：** `["flower","flow","flight"]` → `"fl"`

### 思路讲解
**纵向扫描**：以第一个字符串为基准，逐列检查所有字符串在同一位置的字符是否相同。

```
  f l o w e r
  f l o w
  f l i g h t

列0: f,f,f → 全同 ✓
列1: l,l,l → 全同 ✓
列2: o,o,i → 不同 ✗ → 返回前两列 "fl"
```

**时间复杂度：O(n·m)**，n 为字符串数量，m 为最短字符串长度

### Go 代码

```go
func longestCommonPrefix(strs []string) string {
    if len(strs) == 0 {
        return ""
    }
    prefix := strs[0]
    for _, s := range strs[1:] {
        // 当 s 不以 prefix 开头时，缩短 prefix
        for len(prefix) > 0 && (len(s) < len(prefix) || s[:len(prefix)] != prefix) {
            prefix = prefix[:len(prefix)-1]
        }
        if prefix == "" {
            return ""
        }
    }
    return prefix
}
```

### Python 代码

```python
def longestCommonPrefix(strs: list[str]) -> str:
    if not strs:
        return ""
    prefix = strs[0]
    for s in strs[1:]:
        # 缩短 prefix 直到 s 以 prefix 开头
        while not s.startswith(prefix):
            prefix = prefix[:-1]
            if not prefix:
                return ""
    return prefix
```

---

## 4. 字符串转整数（LeetCode 8）

### 题意
实现 `atoi` 函数，将字符串转为整数，需处理空格、符号、溢出。

**例子：** `"  -42"` → `-42`，`"4193 with words"` → `4193`，`"91283472332"` → `2147483647`（溢出截断）

### 思路讲解
**状态机思想**，按顺序处理：
1. 跳过前导空格
2. 处理符号（`+`/`-`）
3. 逐位读数字，边读边检查是否溢出
4. 遇到非数字字符停止

**溢出判断**：在乘以 10 之前先判断，防止 int64 溢出也是常见考点。

```
"  -42abc"

步骤1: 跳过空格 → "-42abc"
步骤2: 遇到 '-' → sign = -1
步骤3: '4' → result = 4
       '2' → result = 42
       'a' → 停止
步骤4: 返回 -1 * 42 = -42
```

### Go 代码

```go
import "math"

func myAtoi(s string) int {
    i, n := 0, len(s)

    // 1. 跳过前导空格
    for i < n && s[i] == ' ' {
        i++
    }

    // 2. 处理符号
    sign := 1
    if i < n && (s[i] == '+' || s[i] == '-') {
        if s[i] == '-' {
            sign = -1
        }
        i++
    }

    // 3. 读数字
    result := 0
    for i < n && s[i] >= '0' && s[i] <= '9' {
        digit := int(s[i] - '0')
        // 溢出检查：乘 10 前判断
        if result > (math.MaxInt32-digit)/10 {
            if sign == 1 {
                return math.MaxInt32
            }
            return math.MinInt32
        }
        result = result*10 + digit
        i++
    }

    return sign * result
}
```

### Python 代码

```python
def myAtoi(s: str) -> int:
    INT_MAX, INT_MIN = 2**31 - 1, -(2**31)
    s = s.lstrip()  # 去前导空格
    if not s:
        return 0

    sign = 1
    i = 0
    if s[0] in ('+', '-'):
        sign = -1 if s[0] == '-' else 1
        i = 1

    result = 0
    while i < len(s) and s[i].isdigit():
        result = result * 10 + int(s[i])
        i += 1

    result *= sign
    return max(INT_MIN, min(INT_MAX, result))  # 截断溢出
```

---

## 5. 最长回文子串（LeetCode 5）

### 题意
给定字符串，返回其中最长的回文子串。

**例子：** `"babad"` → `"bab"` 或 `"aba"`，`"cbbd"` → `"bb"`

### 思路讲解
**中心扩展法**：回文串是从中心向两边扩展的，枚举每个可能的中心，向两边扩展直到不对称。

中心有两种情况：
- 奇数长度：以单个字符为中心（如 `"aba"` 以 `b` 为中心）
- 偶数长度：以两个字符之间为中心（如 `"abba"` 以 `bb` 之间为中心）

```
s = "cbbd"

以 c 为中心（奇数）: 只有 "c"
以 cb 之间（偶数）:  c≠b，不是回文
以 b 为中心（奇数）: 只有 "b"（因为 c≠b）
以 bb 之间（偶数）:  b==b → "bb"，再向外 c≠d，停止 → "bb"
以 b 为中心（奇数）: b≠d，只有 "b"
以 bd 之间（偶数）:  b≠d，不是回文
以 d 为中心（奇数）: 只有 "d"

最长: "bb"，长度 2
```

**时间复杂度：O(n²)**，空间复杂度：O(1)

### Go 代码

```go
func longestPalindrome(s string) string {
    if len(s) < 2 {
        return s
    }
    start, maxLen := 0, 1

    expand := func(left, right int) {
        for left >= 0 && right < len(s) && s[left] == s[right] {
            if right-left+1 > maxLen {
                start = left
                maxLen = right - left + 1
            }
            left--
            right++
        }
    }

    for i := 0; i < len(s); i++ {
        expand(i, i)   // 奇数长度，以 i 为中心
        expand(i, i+1) // 偶数长度，以 i 和 i+1 之间为中心
    }
    return s[start : start+maxLen]
}
```

### Python 代码

```python
def longestPalindrome(s: str) -> str:
    res = ""

    def expand(left: int, right: int) -> str:
        while left >= 0 and right < len(s) and s[left] == s[right]:
            left -= 1
            right += 1
        return s[left + 1:right]  # 注意：退出时 left/right 已越界一步

    for i in range(len(s)):
        odd = expand(i, i)       # 奇数长度
        even = expand(i, i + 1)  # 偶数长度
        if len(odd) > len(res):
            res = odd
        if len(even) > len(res):
            res = even

    return res
```

---

## 高频追问

**Q：回文判断有哪几种方法？各自复杂度？**
> 1. 双指针：O(n) 时间，O(1) 空间，适合判断整个串是否回文
> 2. 中心扩展：O(n²) 时间，O(1) 空间，适合找最长回文子串
> 3. 动态规划：O(n²) 时间，O(n²) 空间，代码清晰但空间浪费
> 4. Manacher 算法：O(n) 时间，O(n) 空间，最优解但面试中极少考

**Q：atoi 容易踩的坑有哪些？**
> 1. 溢出：要在乘 10 之前判断，不能等结果出来再比较（否则已经溢出了）
> 2. 前导空格：只跳过空格，不跳过其他字符
> 3. 符号只能出现一次，且必须在数字前面
> 4. `"+-12"` 这种情况返回 0
