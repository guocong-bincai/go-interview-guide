# 02. 三数之和（3Sum）

> 频率：★★★★★  难度：中等  LeetCode 15

## 题目描述
给你一个整数数组 `nums`，判断是否存在三元组 `[nums[i], nums[j], nums[k]]` 满足：
- `i != j`
- `i != k`
- `j != k`
- `nums[i] + nums[j] + nums[k] == 0`

要求返回所有 **不重复** 的三元组。

## 面试为什么爱问
- 排序 + 双指针的经典题
- 会考去重细节
- 能看出你是不是只会背模板，还是真的理解

## 小白先理解
这题其实是：
**先固定一个数，再去剩下的部分里找两个数，让三数和为 0。**

比如固定 `-1`，那么剩下两个数之和就要等于 `1`。

---

## 解法一：暴力枚举

### 思路
三层循环，把所有三元组都试一遍。
再用集合去重。

### Go
```go
func threeSumBruteForce(nums []int) [][]int {
    res := [][]int{}
    seen := make(map[[3]int]bool)
    sort.Ints(nums)
    n := len(nums)
    for i := 0; i < n; i++ {
        for j := i + 1; j < n; j++ {
            for k := j + 1; k < n; k++ {
                if nums[i]+nums[j]+nums[k] == 0 {
                    key := [3]int{nums[i], nums[j], nums[k]}
                    if !seen[key] {
                        seen[key] = true
                        res = append(res, []int{nums[i], nums[j], nums[k]})
                    }
                }
            }
        }
    }
    return res
}
```

### Python
```python
def three_sum_bruteforce(nums):
    nums.sort()
    n = len(nums)
    res = []
    seen = set()
    for i in range(n):
        for j in range(i + 1, n):
            for k in range(j + 1, n):
                if nums[i] + nums[j] + nums[k] == 0:
                    item = (nums[i], nums[j], nums[k])
                    if item not in seen:
                        seen.add(item)
                        res.append([nums[i], nums[j], nums[k]])
    return res
```

### 复杂度
- 时间：`O(n^3)`
- 空间：`O(n)`（去重集合）

---

## 解法二：排序 + 双指针（推荐）

### 核心思路
1. 先排序
2. 枚举第一个数 `nums[i]`
3. 变成在右边区间里找两个数，使它们和等于 `-nums[i]`
4. 用左右指针逼近

### 为什么能用双指针
因为数组排序后：
- 和太小，左指针右移
- 和太大，右指针左移

### Go
```go
import "sort"

func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    res := [][]int{}
    n := len(nums)

    for i := 0; i < n-2; i++ {
        if i > 0 && nums[i] == nums[i-1] {
            continue
        }
        left, right := i+1, n-1
        for left < right {
            sum := nums[i] + nums[left] + nums[right]
            if sum == 0 {
                res = append(res, []int{nums[i], nums[left], nums[right]})
                left++
                right--
                for left < right && nums[left] == nums[left-1] {
                    left++
                }
                for left < right && nums[right] == nums[right+1] {
                    right--
                }
            } else if sum < 0 {
                left++
            } else {
                right--
            }
        }
    }
    return res
}
```

### Python
```python
def three_sum(nums):
    nums.sort()
    n = len(nums)
    res = []

    for i in range(n - 2):
        if i > 0 and nums[i] == nums[i - 1]:
            continue
        left, right = i + 1, n - 1
        while left < right:
            s = nums[i] + nums[left] + nums[right]
            if s == 0:
                res.append([nums[i], nums[left], nums[right]])
                left += 1
                right -= 1
                while left < right and nums[left] == nums[left - 1]:
                    left += 1
                while left < right and nums[right] == nums[right + 1]:
                    right -= 1
            elif s < 0:
                left += 1
            else:
                right -= 1
    return res
```

### 复杂度
- 时间：`O(n^2)`
- 空间：`O(1)`（不算结果）

---

## 易错点
- 一定要排序
- 一定要去重：`i` 去重、`left` 去重、`right` 去重
- 返回的是三元组，不是下标

## 一句话记忆
**三数之和 = 排序 + 固定一个数 + 双指针找另外两个数。**
