# 二分查找

> 考察频率：★★★★★  题型识别：有序数组/单调性 + O(log n) 要求

## 核心模板

二分查找的本质是**利用单调性，每次排除一半元素**。关键在于边界处理——推荐使用**左闭右开** `[left, right)` 模板，代码更干净。

### Go 标准模板（左闭右开）

```go
func binarySearch(nums []int, target int) int {
    left, right := 0, len(nums) // [left, right)

    for left < right { // 终止条件：left == right，区间为空
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        } else if nums[mid] < target {
            left = mid + 1  // 目标在右半部分，mid 本身已排除
        } else {
            right = mid     // 目标在左半部分，mid 已排除，right=mid（不开）
        }
    }
    return -1 // 未找到
}
```

**注意**：
- `mid := left + (right-left)/2` 防止 (left+right) 溢出
- `right = mid` 而不是 `right = mid-1`（因为右开，不包含 mid）

---

## 变种一：查找左边界（第一个 >= target 的位置）

```go
func lowerBound(nums []int, target int) int {
    left, right := 0, len(nums) // [left, right)

    for left < right {
        mid := left + (right-left)/2
        if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid // nums[mid] >= target，左边界在左边（含mid）
        }
    }
    return left // left == right
}
```

**验证**：
```
nums = [1, 3, 5, 7, 9], target = 5
→ 返回 2 (nums[2]=5)
nums = [1, 3, 5, 7, 9], target = 6
→ 返回 3 (第一个 >= 6 的位置)
```

---

## 变种二：查找右边界（最后一个 <= target 的位置）

```go
func upperBound(nums []int, target int) int {
    left, right := 0, len(nums) // [left, right)

    for left < right {
        mid := left + (right-left)/2
        if nums[mid] <= target {
            left = mid + 1 // 右边界在右边（mid 可能就是）
        } else {
            right = mid
        }
    }
    return left - 1 // 最后一个小于等于 target 的位置
}
```

**验证**：
```
nums = [1, 3, 5, 7, 9], target = 5
→ 返回 2 (nums[2]=5)
nums = [1, 3, 5, 7, 9], target = 6
→ 返回 2 (最后一个小于等于 6 的位置)
```

---

## 题型一：搜索旋转排序数组（LeetCode 33）

### 题目
数组原本升序，可以看成 `[0,1,2,3,4,5,6,7]` 旋转后变成 `[4,5,6,7,0,1,2]`，给定 target，找出是否存在。

### 核心思路
二分后，比较 `nums[mid]` 与 `nums[left]` 确定哪半边有序：
- 若 `nums[left] <= nums[mid]`，左半边 `[left..mid]` 有序
  - 若 target 在左半边范围：`right = mid`
  - 否则：`left = mid + 1`
- 否则右半边有序，同理

### 代码

```go
func search(nums []int, target int) int {
    if len(nums) == 0 {
        return -1
    }

    left, right := 0, len(nums)-1

    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        }

        // 左半边有序 [left..mid]
        if nums[left] <= nums[mid] {
            if nums[left] <= target && target < nums[mid] {
                right = mid - 1
            } else {
                left = mid + 1
            }
        } else {
            // 右半边有序 [mid..right]
            if nums[mid] < target && target <= nums[right] {
                left = mid + 1
            } else {
                right = mid - 1
            }
        }
    }
    return -1
}
```

### 复杂度
- 时间：O(log n)
- 空间：O(1)

### 追问
- **有重复数字的版本**（LeetCode 81）：当 `nums[left] == nums[mid]` 时，左指针右移 `left++`，跳过这个坑。

---

## 题型二：寻找峰值（LeetCode 162）

### 题目
找出数组中任意一个峰值（比相邻元素都大），要求 O(log n)。

### 思路
利用二分：比较 `nums[mid]` 与 `nums[mid+1]`：
- 若 `nums[mid] < nums[mid+1]`，峰值在**右边**，因为数组往右是上升的
- 否则，`nums[mid]` 可能是峰值（或左边有峰值），收缩到 `[left..mid]`

### 代码

```go
func findPeakElement(nums []int) int {
    left, right := 0, len(nums)-1

    for left < right {
        mid := left + (right-left)/2
        if nums[mid] < nums[mid+1] {
            left = mid + 1 // 峰值在右边
        } else {
            right = mid // mid 可能是峰值，或左边有峰值
        }
    }
    return left
}
```

---

## 题型三：搜索二维矩阵（LeetCode 74）

### 题目
矩阵每行升序，每行第一个元素大于上一行最后一个元素。判断 target 是否存在。

### 思路
把二维转一维：行 = idx / n，列 = idx % n，直接二分。或者从右上角开始 BST 查找。

```go
func searchMatrix(matrix [][]int, target int) bool {
    if len(matrix) == 0 || len(matrix[0]) == 0 {
        return false
    }

    m, n := len(matrix), len(matrix[0])
    left, right := 0, m*n-1

    for left <= right {
        mid := left + (right-left)/2
        val := matrix[mid/n][mid%n]

        if val == target {
            return true
        } else if val < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return false
}
```

---

## 题型四：爱吃香蕉的珂珂（LeetCode 875）

### 题目
珂珂以速度 k 吃香蕉，h 小时内吃完。求最小速度 k。

### 思路
求最小满足条件的速度 → 二分答案。单调性：速度越快，时间越短。

```go
func minEatingSpeed(piles []int, h int) int {
    // 二分查找速度
    left, right := 1, maxInSlice(piles)

    for left < right {
        mid := left + (right-left)/2
        // 计算 mid 速度需要多少小时
        hours := 0
        for _, p := range piles {
            hours += (p + mid - 1) / mid // 向上取整
        }

        if hours <= h {
            right = mid // mid 可行，尝试更小速度
        } else {
            left = mid + 1
        }
    }
    return left
}

func maxInSlice(nums []int) int {
    mx := nums[0]
    for _, v := range nums {
        if v > mx {
            mx = v
        }
    }
    return mx
}
```

---

## 二分答案模板

当问题满足：
1. 答案范围有界（可以用 left/right 表示）
2. 答案有单调性（check(mid) 是 true/false）

→ 直接二分答案

```go
func binarySearchAnswer(left, right int, check func(int) bool) int {
    for left < right {
        mid := left + (right-left)/2
        if check(mid) {
            right = mid // 答案在左半边
        } else {
            left = mid + 1
        }
    }
    return left
}
```

**典型应用**：
- 珂珂吃香蕉（速度）
- 分割数组的最大值（最大值）
- 至少需要 K 名工人的建筑外观（参数）
- 传送带最小速度（时间）

---

## 总结

| 要点 | 说明 |
|------|------|
| 边界处理 | 推荐左闭右开 `[left, right)`，统一 `right = mid` |
| 防溢出 | `mid = left + (right-left)/2` |
| 找左边界 | `right = mid`，返回 left |
| 找右边界 | `left = mid + 1`，返回 left-1 |
| 二分答案 | 单调性 + 范围可界 → 直接二分 |
| 旋转数组 | 先判断哪半边有序，再判断 target 位置 |
