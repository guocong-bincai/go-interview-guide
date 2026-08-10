# 01. 两数之和（Two Sum）

> 频率：★★★★★  难度：简单  LeetCode 1

## 题目描述
给定一个整数数组 `nums` 和一个目标值 `target`，请你在数组中找出 **和为目标值** 的那两个整数下标。

## 面试为什么爱问
- 这是哈希表入门题，特别高频
- 面试官想看你会不会用 **空间换时间**
- 还能顺手看你是否能把暴力解法优化掉

## 小白先理解题意
比如：

```text
nums = [2, 7, 11, 15], target = 9
```

因为 `2 + 7 = 9`，所以答案是 `[0, 1]`。

重点不是把所有组合都试一遍，而是：
**当我看到一个数 x 时，我就想知道 target - x 之前有没有出现过。**

---

## 解法一：暴力枚举

### 思路
用两层循环，把所有两两组合都试一次。

### 步骤
1. 第一个数从前往后选
2. 第二个数从第一个数后面开始选
3. 如果两数之和等于 target，直接返回

### 优点
- 最容易想到
- 代码简单

### 缺点
- 太慢
- 时间复杂度高

### Go
```go
func twoSumBruteForce(nums []int, target int) []int {
    for i := 0; i < len(nums); i++ {
        for j := i + 1; j < len(nums); j++ {
            if nums[i]+nums[j] == target {
                return []int{i, j}
            }
        }
    }
    return nil
}
```

### Python
```python
def two_sum_bruteforce(nums, target):
    n = len(nums)
    for i in range(n):
        for j in range(i + 1, n):
            if nums[i] + nums[j] == target:
                return [i, j]
    return []
```

### 复杂度
- 时间：`O(n^2)`
- 空间：`O(1)`

---

## 解法二：哈希表（推荐，面试标准答案）

### 核心思路
当我们遍历到 `nums[i] = x` 时：
- 需要找的另一个数是 `target - x`
- 如果这个数之前出现过，就直接找到答案
- 所以用哈希表记录：`数值 -> 下标`

### 举例
```text
nums = [2, 7, 11, 15], target = 9
```

- 看到 2，想找 7，没找到，把 2 存起来
- 看到 7，想找 2，找到了
- 返回 `[0, 1]`

### Go
```go
func twoSum(nums []int, target int) []int {
    indexMap := make(map[int]int)
    for i, num := range nums {
        need := target - num
        if j, ok := indexMap[need]; ok {
            return []int{j, i}
        }
        indexMap[num] = i
    }
    return nil
}
```

### Python
```python
def two_sum(nums, target):
    index_map = {}
    for i, num in enumerate(nums):
        need = target - num
        if need in index_map:
            return [index_map[need], i]
        index_map[num] = i
    return []
```

### 复杂度
- 时间：`O(n)`
- 空间：`O(n)`

---

## 两种解法怎么选
- 如果只是先想到最基础方法：先说暴力
- 面试时一定要继续优化到哈希表
- 标准回答：**暴力能做，但最优是哈希表**

---

## 易错点
- 不要先把当前元素放进哈希表再查，否则会把自己和自己配对
- 题目返回的是 **下标**，不是数值
- 一般默认每种输入只有一个答案

---

## 面试追问
- 如果数组已经有序，怎么做？
- 如果要返回所有结果怎么办？
- 如果不允许额外空间怎么办？

---

## 一句话记忆
**看到“找两个数之和”，优先想哈希表。**
