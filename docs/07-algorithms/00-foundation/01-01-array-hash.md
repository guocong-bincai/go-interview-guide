# 数组与哈希高频题

> 考察频率：★★★★★  难度：简单~中等  Go + Python 双语

---

## 题目导航

| # | 题目 | 难度 | 考察频率 |
|---|------|------|---------|
| 1 | [两数之和](#1-两数之和-leetcode-1) | 简单 | ★★★★★ |
| 2 | [三数之和](#2-三数之和-leetcode-15) | 中等 | ★★★★☆ |
| 3 | [字母异位词分组](#3-字母异位词分组-leetcode-49) | 中等 | ★★★★☆ |
| 4 | [最长连续序列](#4-最长连续序列-leetcode-128) | 中等 | ★★★★☆ |
| 5 | [有效的括号](#5-有效的括号-leetcode-20) | 简单 | ★★★★★ |

---

## 1. 两数之和（LeetCode 1）

### 题意
给定整数数组 `nums` 和目标值 `target`，找出两个数的下标，使它们相加等于 `target`。

**例子：** `nums = [2, 7, 11, 15]`，`target = 9` → `[0, 1]`（因为 `2 + 7 = 9`）

### 思路讲解

**暴力法**（不推荐）：两层 for 循环，枚举所有组合，O(n²)。

**哈希表法**（推荐）：
想象你在找"另一半"。对于每个数 `x`，你要找的是 `target - x`。
用一个字典记录"我见过哪些数，它们在哪个位置"，边走边找，一遍扫描搞定。

```
nums = [2, 7, 11, 15], target = 9

i=0, x=2, 需要找 9-2=7, 字典里没有 → 把 {2:0} 存入字典
i=1, x=7, 需要找 9-7=2, 字典里有 2 在位置 0 → 返回 [0, 1]
```

**时间复杂度：O(n)**，空间复杂度：O(n)

### Go 代码

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int) // value -> index
    for i, x := range nums {
        need := target - x
        if j, ok := seen[need]; ok {
            return []int{j, i}
        }
        seen[x] = i
    }
    return nil
}
```

### Python 代码

```python
def twoSum(nums: list[int], target: int) -> list[int]:
    seen = {}  # value -> index
    for i, x in enumerate(nums):
        need = target - x
        if need in seen:
            return [seen[need], i]
        seen[x] = i
    return []
```

---

## 2. 三数之和（LeetCode 15）

### 题意
找出数组中所有不重复的三元组，使得三数之和为 0。

**例子：** `nums = [-1, 0, 1, 2, -1, -4]` → `[[-1, -1, 2], [-1, 0, 1]]`

### 思路讲解

**哈希表法为什么不好用？** 三元组去重太麻烦，容易出 bug。

**排序 + 双指针**（推荐）：
1. 先排序，让相同的数相邻，方便去重
2. 固定第一个数 `nums[i]`，剩下两个数用双指针在 `[i+1, n-1]` 范围内找

```
排序后: [-4, -1, -1, 0, 1, 2]

固定 i=1, nums[i]=-1, 双指针 left=2, right=5
  -1 + (-1) + 2 = 0 ✓ 找到一组，left++, right--
  -1 + 0 + 1 = 0 ✓ 找到一组，left++, right--
  left >= right，结束本轮

固定 i=2 时，nums[2]==-1 == nums[1]，跳过（去重）
```

**时间复杂度：O(n²)**，空间复杂度：O(1)（不算输出）

### Go 代码

```go
import "sort"

func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    result := [][]int{}
    n := len(nums)

    for i := 0; i < n-2; i++ {
        // 去重：跳过重复的第一个数
        if i > 0 && nums[i] == nums[i-1] {
            continue
        }
        // 优化：最小的三个数之和已经 > 0，后面不可能有解
        if nums[i] > 0 {
            break
        }

        left, right := i+1, n-1
        for left < right {
            sum := nums[i] + nums[left] + nums[right]
            if sum == 0 {
                result = append(result, []int{nums[i], nums[left], nums[right]})
                // 去重：跳过重复的 left 和 right
                for left < right && nums[left] == nums[left+1] {
                    left++
                }
                for left < right && nums[right] == nums[right-1] {
                    right--
                }
                left++
                right--
            } else if sum < 0 {
                left++
            } else {
                right--
            }
        }
    }
    return result
}
```

### Python 代码

```python
def threeSum(nums: list[int]) -> list[list[int]]:
    nums.sort()
    result = []
    n = len(nums)

    for i in range(n - 2):
        if i > 0 and nums[i] == nums[i - 1]:  # 去重
            continue
        if nums[i] > 0:  # 最小的数都 > 0，不可能有解
            break

        left, right = i + 1, n - 1
        while left < right:
            s = nums[i] + nums[left] + nums[right]
            if s == 0:
                result.append([nums[i], nums[left], nums[right]])
                while left < right and nums[left] == nums[left + 1]:
                    left += 1
                while left < right and nums[right] == nums[right - 1]:
                    right -= 1
                left += 1
                right -= 1
            elif s < 0:
                left += 1
            else:
                right -= 1

    return result
```

---

## 3. 字母异位词分组（LeetCode 49）

### 题意
将字符串数组中互为字母异位词的字符串分到同一组。

**例子：** `["eat","tea","tan","ate","nat","bat"]` → `[["eat","tea","ate"], ["tan","nat"], ["bat"]]`

### 思路讲解

**什么是异位词？** 组成字母相同，顺序不同。比如 "eat" 和 "tea" 都由 e、a、t 组成。

**关键思路：找到异位词的"身份证"**
把每个单词排序后，异位词的排序结果一定相同。
- "eat" → 排序 → "aet"
- "tea" → 排序 → "aet"
- "ate" → 排序 → "aet"

用排序结果作为哈希表的 key，把原词归入同一组。

**时间复杂度：O(n · k·log k)**，k 为字符串平均长度

### Go 代码

```go
import "sort"

func groupAnagrams(strs []string) [][]string {
    groups := make(map[string][]string)

    for _, s := range strs {
        // 把字符串排序作为 key
        key := sortString(s)
        groups[key] = append(groups[key], s)
    }

    result := make([][]string, 0, len(groups))
    for _, v := range groups {
        result = append(result, v)
    }
    return result
}

func sortString(s string) string {
    b := []byte(s)
    sort.Slice(b, func(i, j int) bool { return b[i] < b[j] })
    return string(b)
}
```

### Python 代码

```python
from collections import defaultdict

def groupAnagrams(strs: list[str]) -> list[list[str]]:
    groups = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))  # 排序结果作为 key
        groups[key].append(s)
    return list(groups.values())
```

---

## 4. 最长连续序列（LeetCode 128）

### 题意
给定未排序的整数数组，找到最长连续数字序列的长度，要求 O(n) 时间复杂度。

**例子：** `[100, 4, 200, 1, 3, 2]` → `4`（序列 `[1, 2, 3, 4]`）

### 思路讲解

**排序后找？** 可以但是 O(n log n)，不符合要求。

**哈希表 O(n) 做法**：
只从每段连续序列的**起点**开始计数，避免重复扫描。

怎么判断是起点？`x-1` 不在集合里，说明 `x` 就是这段序列的开头。

```
set = {100, 4, 200, 1, 3, 2}

x=100: 99 不在 set 里 → 是起点，往后数 101, 102... 只有 100，长度=1
x=4:   3 在 set 里 → 不是起点，跳过
x=200: 199 不在 set 里 → 是起点，201 不在，长度=1
x=1:   0 不在 set 里 → 是起点，2 在→3 在→4 在→5 不在，长度=4
x=3:   2 在 set 里 → 不是起点，跳过
x=2:   1 在 set 里 → 不是起点，跳过

最长 = 4
```

**时间复杂度：O(n)**，每个元素最多被访问两次

### Go 代码

```go
func longestConsecutive(nums []int) int {
    set := make(map[int]bool)
    for _, x := range nums {
        set[x] = true
    }

    best := 0
    for x := range set {
        // 只从序列起点开始数
        if !set[x-1] {
            cur := x
            length := 1
            for set[cur+1] {
                cur++
                length++
            }
            if length > best {
                best = length
            }
        }
    }
    return best
}
```

### Python 代码

```python
def longestConsecutive(nums: list[int]) -> int:
    num_set = set(nums)
    best = 0

    for x in num_set:
        if x - 1 not in num_set:  # 只从起点开始
            cur = x
            length = 1
            while cur + 1 in num_set:
                cur += 1
                length += 1
            best = max(best, length)

    return best
```

---

## 5. 有效的括号（LeetCode 20）

### 题意
给定只含括号的字符串，判断括号是否正确匹配。

**例子：** `"()[]{}"` → `true`，`"(]"` → `false`，`"([)]"` → `false`

### 思路讲解

**用栈模拟**，就像叠盘子：
- 遇到左括号 `(`、`[`、`{`，压入栈
- 遇到右括号，看栈顶是不是对应的左括号：
  - 是 → 弹出栈顶（匹配成功）
  - 不是 → 直接返回 false
- 最后栈为空才算全部匹配

```
s = "({[]})"

遇 ( → 栈: [(]
遇 { → 栈: [(, {]
遇 [ → 栈: [(, {, []
遇 ] → 栈顶是 [, 匹配! 栈: [(, {]
遇 } → 栈顶是 {, 匹配! 栈: [(]
遇 ) → 栈顶是 (, 匹配! 栈: []

栈为空 → true
```

**时间复杂度：O(n)**，空间复杂度：O(n)

### Go 代码

```go
func isValid(s string) bool {
    stack := []rune{}
    pairs := map[rune]rune{
        ')': '(',
        ']': '[',
        '}': '{',
    }

    for _, c := range s {
        if c == '(' || c == '[' || c == '{' {
            stack = append(stack, c)
        } else {
            // 右括号：检查栈顶
            if len(stack) == 0 || stack[len(stack)-1] != pairs[c] {
                return false
            }
            stack = stack[:len(stack)-1]
        }
    }
    return len(stack) == 0
}
```

### Python 代码

```python
def isValid(s: str) -> bool:
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}

    for c in s:
        if c in '([{':
            stack.append(c)
        else:
            if not stack or stack[-1] != pairs[c]:
                return False
            stack.pop()

    return len(stack) == 0
```

---

## 高频追问

**Q：哈希表和数组做"计数"有什么区别？**
> 字符范围固定（如只有小写字母）用 `[26]int` 数组，O(1) 空间、更快。字符范围不固定（如任意字符串）用哈希表。

**Q：三数之和为什么比两数之和难？**
> 两数之和用哈希表 O(n) 搞定；三数之和要去重，哈希表去重逻辑复杂，排序+双指针更直观稳定。

**Q：最长连续序列为什么不用排序？**
> 排序是 O(n log n)，题目要求 O(n)。哈希集合的核心技巧是"只从起点开始数"，避免重复计算。
