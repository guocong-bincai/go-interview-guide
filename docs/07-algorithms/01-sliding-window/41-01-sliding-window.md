# 滑动窗口

> 考察频率：★★★★☆  题型识别：子串/子数组 + 最值/统计

## 核心模板

滑动窗口本质是**双指针**的特例——左右指针始终维持一个"窗口"，右指针扩张窗口，左指针收缩窗口。适用于"连续子数组/子串"问题，时间复杂度 O(n)。

### 固定窗口 vs 可变窗口

| 类型 | 窗口大小 | 适用场景 |
|------|---------|---------|
| 固定窗口 | `right - left == k` 时才计算 | 窗口平均值、最大和 |
| 可变窗口 | 根据条件自动扩张/收缩 | 无重复字符、长度最小子串 |

### Go 模板

```go
// 可变窗口模板（通用）
func slidingWindow(s string) int {
    left, right := 0, 0
    freq := make(map[byte]int) // 或 map[rune]int
    result := 0

    for right < len(s) {
        // 1. 右指针纳入窗口
        freq[s[right]]++
        right++

        // 2. 尝试收缩左指针（关键！）
        for left < right && validConditionViolated() {
            freq[s[left]]--
            left++
        }

        // 3. 更新答案
        result = max(result, right-left)
    }
    return result
}
```

---

## 题型一：无重复字符的最长子串（LeetCode 3）

### 题目
给定字符串 `s`，找出其中不含有重复字符的最长子串的长度。

### 思路
- 用 `freq` 记录窗口内每个字符出现次数
- 扩张右指针，当遇到重复字符（`freq[c] > 1`）时，收缩左指针直到重复字符被移除
- 每轮更新最大长度

### 代码

```go
func lengthOfLongestSubstring(s string) int {
    freq := make(map[byte]int)
    left, result := 0, 0

    for right := 0; right < len(s); right++ {
        freq[s[right]]++
        // 收缩左指针直到没有重复
        for left < right && freq[s[right]] > 1 {
            freq[s[left]]--
            left++
        }
        result = max(result, right-left+1)
    }
    return result
}
```

**变种**：如果允许最多 k 个重复字符怎么做？
→ 把 `freq[s[right]] > 1` 改为 `freq[s[right]] > k`，逻辑相同。

### 复杂度
- 时间：O(n)，每个字符最多进入窗口一次、离开一次
- 空间：O(min(m, n))，m 是字符集大小（ASCII 为 128）

---

## 题型二：长度最小的子数组（LeetCode 209）

### 题目
给定数组 `nums` 和目标值 `s`，找出最短的连续子数组，使其和 ≥ s。

### 思路
- 扩张右指针累加和
- 当 sum ≥ s 时，收缩左指针尝试缩小窗口，同时更新最短长度

### 代码

```go
func minSubArrayLen(s int, nums []int) int {
    left, sum, result := 0, 0, math.MaxInt

    for right := 0; right < len(nums); right++ {
        sum += nums[right]

        for left <= right && sum >= s {
            result = min(result, right-left+1)
            sum -= nums[left]
            left++
        }
    }
    if result == math.MaxInt {
        return 0
    }
    return result
}
```

### 复杂度
- 时间：O(n)
- 空间：O(1)

---

## 题型三：最大滑动窗口（LeetCode 239）

### 题目
给定数组和窗口大小 k，返回每个滑动窗口的最大值。

### 思路：单调递减队列

维护一个存储**下标**的单调递减队列：
- 队头 = 当前窗口最大值的下标
- 入队：移除队列中所有 ≤ 当前元素的下标（它们永远不可能是最大值了）
- 出队：移除队列头超出窗口范围的下标

### 代码

```go
func maxSlidingWindow(nums []int, k int) []int {
    if len(nums) == 0 || k == 0 {
        return []int{}
    }

    var result []int
    queue := make([]int, 0, k) // 存下标，单调递减

    for right := 0; right < len(nums); right++ {
        // 1. 入队：移除队列中所有 ≤ nums[right] 的下标（它们没用了）
        for len(queue) > 0 && nums[queue[len(queue)-1]] <= nums[right] {
            queue = queue[:len(queue)-1]
        }
        queue = append(queue, right)

        // 2. 出队：移除队头超出窗口范围的下标
        if queue[0] <= right-k {
            queue = queue[1:]
        }

        // 3. 窗口满了才记录答案
        if right >= k-1 {
            result = append(result, nums[queue[0]])
        }
    }
    return result
}
```

### 复杂度
- 时间：O(n)，每个元素最多入队、出队各一次
- 空间：O(k)

### 追问
1. 如果要返回最小值？→ 改为单调递增队列（`>=` 而非 `<=`）
2. 如何返回最大值和最小值同时？→ 同时维护递增和递减两个队列

---

## 题型四：字符串的排列（LeetCode 567）

### 题目
判断 `s2` 是否包含 `s1` 的某个排列（字符种类和个数相同）。

### 思路
- 固定窗口大小 = len(s1)
- 用两个频率数组比较，或用一个 counter 记录当前窗口与目标的差异

### 代码

```go
func checkInclusion(s1 string, s2 string) bool {
    if len(s1) > len(s2) {
        return false
    }

    // 统计 s1 的字符频率
    need := [26]int{}
    for _, c := range s1 {
        need[c-'a']++
    }

    // 滑动窗口：维护 diff=还有多少字符未匹配
    diff := [26]int{}
    matched := 0

    for right := 0; right < len(s2); right++ {
        idx := s2[right] - 'a'
        if need[idx] > 0 { // 这个字符是 s1 需要的
            diff[idx]++
            if diff[idx] <= need[idx] {
                matched++
            }
        }

        // 收缩左指针保持窗口大小 = len(s1)
        if right >= len(s1) {
            leftIdx := s2[right-len(s1)] - 'a'
            if need[leftIdx] > 0 {
                if diff[leftIdx] <= need[leftIdx] {
                    matched--
                }
                diff[leftIdx]--
            }
        }

        if matched == len(s1) {
            return true
        }
    }
    return false
}
```

---

## 常见追问与变种

| 追问 | 思路 |
|------|------|
| 窗口内最大值 - 最小值 ≤ k | 同时维护递增和递减队列 |
| 最多替换 k 个字符的最长好子串 | 扩张时记录0的个数，超过k时收缩 |
| 和为 k 的倍数子数组 | 前缀和 % k，用 map 记录最早出现位置 |
| 环形滑动窗口最大值 | 把数组复制一倍，转化为普通滑动窗口 |

---

## 模板总结

```
滑动窗口三步曲：
1. 扩张 right，把元素纳入窗口
2. 收缩 left，直到窗口重新满足约束
3. 更新答案
```

**何时用单调队列**：需要 O(1) 获取窗口最值时，用单调递减队列（max）或单调递增队列（min）。

**何时用普通窗口**：只需要统计字符/计数时，用 freq map + 收缩条件判断。

---

## Python 实现汇总

### Python 模板

```python
def sliding_window(s: str) -> int:
    from collections import defaultdict
    freq = defaultdict(int)
    left = result = 0

    for right in range(len(s)):
        freq[s[right]] += 1          # 扩张
        while condition_violated():  # 收缩
            freq[s[left]] -= 1
            left += 1
        result = max(result, right - left + 1)  # 更新答案
    return result
```

### LeetCode 3 - 无重复字符的最长子串

```python
def lengthOfLongestSubstring(s: str) -> int:
    from collections import defaultdict
    freq = defaultdict(int)
    left = result = 0
    for right in range(len(s)):
        freq[s[right]] += 1
        while freq[s[right]] > 1:   # 有重复，收缩
            freq[s[left]] -= 1
            left += 1
        result = max(result, right - left + 1)
    return result
```

### LeetCode 209 - 长度最小的子数组

```python
def minSubArrayLen(target: int, nums: list[int]) -> int:
    left = cur_sum = 0
    result = float('inf')
    for right in range(len(nums)):
        cur_sum += nums[right]
        while cur_sum >= target:
            result = min(result, right - left + 1)
            cur_sum -= nums[left]
            left += 1
    return 0 if result == float('inf') else result
```

### LeetCode 239 - 滑动窗口最大值

```python
from collections import deque

def maxSlidingWindow(nums: list[int], k: int) -> list[int]:
    q = deque()  # 存下标，单调递减
    result = []
    for right in range(len(nums)):
        # 移除队列中所有 <= 当前值的下标
        while q and nums[q[-1]] <= nums[right]:
            q.pop()
        q.append(right)
        # 移除超出窗口范围的队头
        if q[0] <= right - k:
            q.popleft()
        # 窗口满了才记录
        if right >= k - 1:
            result.append(nums[q[0]])
    return result
```

### LeetCode 567 - 字符串的排列

```python
def checkInclusion(s1: str, s2: str) -> bool:
    from collections import Counter
    if len(s1) > len(s2):
        return False
    need = Counter(s1)
    window = Counter(s2[:len(s1)])
    if window == need:
        return True
    for i in range(len(s1), len(s2)):
        # 滑动：加入右端，移除左端
        window[s2[i]] += 1
        left_char = s2[i - len(s1)]
        window[left_char] -= 1
        if window[left_char] == 0:
            del window[left_char]
        if window == need:
            return True
    return False
```
