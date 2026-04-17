# 双指针高频题

> 考察频率：★★★★☆  难度：简单~中等  Go + Python 双语

---

## 题目导航

| # | 题目 | 难度 | 考察频率 |
|---|------|------|---------|
| 1 | [移除元素](#1-移除元素-leetcode-27) | 简单 | ★★★★☆ |
| 2 | [合并两个有序数组](#2-合并两个有序数组-leetcode-88) | 简单 | ★★★★★ |
| 3 | [盛最多水的容器](#3-盛最多水的容器-leetcode-11) | 中等 | ★★★★★ |
| 4 | [接雨水](#4-接雨水-leetcode-42) | 困难 | ★★★★☆ |

---

## 双指针的两种形态

```
对撞指针：左右向中间走，用于有序数组、回文、两数之和等
  left → → → → ← ← ← ← right

快慢指针：快指针先走，慢指针跟随，用于链表环检测、原地修改等
  slow → fast → →
```

---

## 1. 移除元素（LeetCode 27）

### 题意
原地移除数组中所有等于 `val` 的元素，返回新长度。

**例子：** `nums = [3,2,2,3]`，`val = 3` → 返回 `2`，nums 变为 `[2,2,_,_]`

### 思路讲解
**快慢指针**：
- `fast` 遍历所有元素
- 当 `nums[fast] != val` 时，把它复制到 `slow` 位置，`slow++`
- 最终 `slow` 就是新数组的长度

```
nums = [3, 2, 2, 3], val = 3

fast=0: nums[0]=3 == val, 跳过
fast=1: nums[1]=2 != val, nums[slow=0]=2, slow=1
fast=2: nums[2]=2 != val, nums[slow=1]=2, slow=2
fast=3: nums[3]=3 == val, 跳过

返回 slow=2，nums=[2,2,2,3]（前2个有效）
```

### Go 代码

```go
func removeElement(nums []int, val int) int {
    slow := 0
    for fast := 0; fast < len(nums); fast++ {
        if nums[fast] != val {
            nums[slow] = nums[fast]
            slow++
        }
    }
    return slow
}
```

### Python 代码

```python
def removeElement(nums: list[int], val: int) -> int:
    slow = 0
    for fast in range(len(nums)):
        if nums[fast] != val:
            nums[slow] = nums[fast]
            slow += 1
    return slow
```

---

## 2. 合并两个有序数组（LeetCode 88）

### 题意
将有序数组 `nums2` 合并入 `nums1`，`nums1` 末尾有足够空间（m+n 个位置）。

**例子：** `nums1=[1,2,3,0,0,0]`，`m=3`，`nums2=[2,5,6]`，`n=3`
→ `nums1=[1,2,2,3,5,6]`

### 思路讲解
**从后往前写**是关键！从前往后写会覆盖 nums1 还没用的数据。

用三个指针：
- `p1` 指向 nums1 有效部分末尾（位置 m-1）
- `p2` 指向 nums2 末尾（位置 n-1）
- `p` 指向 nums1 最末尾（位置 m+n-1），从后往前写

```
nums1 = [1, 2, 3, 0, 0, 0]  nums2 = [2, 5, 6]
               ↑p1                        ↑p2    ↑p（指向最后一个0）

比较 nums1[p1]=3 和 nums2[p2]=6: 6大 → nums1[p]=6, p2--, p--
比较 nums1[p1]=3 和 nums2[p2]=5: 5大 → nums1[p]=5, p2--, p--
比较 nums1[p1]=3 和 nums2[p2]=2: 3大 → nums1[p]=3, p1--, p--
比较 nums1[p1]=2 和 nums2[p2]=2: 相等 → nums1[p]=2, p2--, p--（任选其一）
...
```

### Go 代码

```go
func merge(nums1 []int, m int, nums2 []int, n int) {
    p1, p2, p := m-1, n-1, m+n-1
    for p1 >= 0 && p2 >= 0 {
        if nums1[p1] > nums2[p2] {
            nums1[p] = nums1[p1]
            p1--
        } else {
            nums1[p] = nums2[p2]
            p2--
        }
        p--
    }
    // nums2 剩余部分直接填入（nums1 剩余已在原位）
    for p2 >= 0 {
        nums1[p] = nums2[p2]
        p2--
        p--
    }
}
```

### Python 代码

```python
def merge(nums1: list[int], m: int, nums2: list[int], n: int) -> None:
    p1, p2, p = m - 1, n - 1, m + n - 1
    while p1 >= 0 and p2 >= 0:
        if nums1[p1] > nums2[p2]:
            nums1[p] = nums1[p1]
            p1 -= 1
        else:
            nums1[p] = nums2[p2]
            p2 -= 1
        p -= 1
    # nums2 剩余部分
    nums1[:p2 + 1] = nums2[:p2 + 1]
```

---

## 3. 盛最多水的容器（LeetCode 11）

### 题意
给定高度数组，找两根柱子使围成的容器面积最大。

**例子：** `height = [1,8,6,2,5,4,8,3,7]` → `49`（柱子 1 和柱子 8，高 min(8,7)=7，宽 8-1=7，面积=49）

### 思路讲解
**对撞双指针**：
容器面积 = `min(左高, 右高) × (右下标 - 左下标)`

直觉：左右指针分别从两端出发。每次移动**较矮的那根**柱子（因为移动较高的柱子，宽度减小，高度不会增加，面积只会更小）。

```
height = [1, 8, 6, 2, 5, 4, 8, 3, 7]
left=0(h=1), right=8(h=7)
面积 = min(1,7) * 8 = 8，left 更矮 → left++

left=1(h=8), right=8(h=7)
面积 = min(8,7) * 7 = 49，right 更矮 → right--

left=1(h=8), right=7(h=3)
面积 = min(8,3) * 6 = 18 < 49，right 更矮 → right--

... 继续直到 left >= right
最大面积 = 49
```

**为什么移动较矮的柱子是对的？**
假设 `h[left] < h[right]`，如果我们移动 `right`，面积 = `h[left] × (right-left-1)`，因为 `h[left]` 更小，所以面积上限已经被 `left` 限制了，新面积最多和现在持平，通常更小。所以移动 `left` 才有机会找到更大的面积。

### Go 代码

```go
func maxArea(height []int) int {
    left, right := 0, len(height)-1
    maxWater := 0
    for left < right {
        h := min(height[left], height[right])
        water := h * (right - left)
        if water > maxWater {
            maxWater = water
        }
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }
    return maxWater
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

### Python 代码

```python
def maxArea(height: list[int]) -> int:
    left, right = 0, len(height) - 1
    max_water = 0
    while left < right:
        water = min(height[left], height[right]) * (right - left)
        max_water = max(max_water, water)
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    return max_water
```

---

## 4. 接雨水（LeetCode 42）

### 题意
给定柱子高度数组，计算下雨后能接多少雨水。

**例子：** `height = [0,1,0,2,1,0,1,3,2,1,2,1]` → `6`

### 思路讲解
**每个位置能接的雨水** = `min(左边最高柱子, 右边最高柱子) - 当前高度`
（取左右最高中的较小值，减去当前高度，就是这个位置的水深）

**双指针优化**：不用预先计算所有位置的左右最大值，用两个变量实时维护。

```
维护 leftMax 和 rightMax
- 如果 leftMax < rightMax：左边是短板，处理 left 位置
  当前水量 = leftMax - height[left]（因为右边最高一定 >= rightMax >= leftMax）
  left++
- 否则：右边是短板，处理 right 位置
  当前水量 = rightMax - height[right]
  right--
```

```
height = [0,1,0,2,1,0,1,3,2,1,2,1]
         ↑left                    ↑right
leftMax=0, rightMax=0

left=0(h=0): leftMax=0 <= rightMax=0, 水量=0-0=0, leftMax=max(0,0)=0, left++
left=1(h=1): leftMax=0 < rightMax=0? 相等,处理left,水量=0-1<0取0,leftMax=1,left++
...
```

### Go 代码

```go
func trap(height []int) int {
    left, right := 0, len(height)-1
    leftMax, rightMax := 0, 0
    total := 0

    for left < right {
        if height[left] < height[right] {
            if height[left] >= leftMax {
                leftMax = height[left]
            } else {
                total += leftMax - height[left]
            }
            left++
        } else {
            if height[right] >= rightMax {
                rightMax = height[right]
            } else {
                total += rightMax - height[right]
            }
            right--
        }
    }
    return total
}
```

### Python 代码

```python
def trap(height: list[int]) -> int:
    left, right = 0, len(height) - 1
    left_max = right_max = 0
    total = 0

    while left < right:
        if height[left] < height[right]:
            if height[left] >= left_max:
                left_max = height[left]
            else:
                total += left_max - height[left]
            left += 1
        else:
            if height[right] >= right_max:
                right_max = height[right]
            else:
                total += right_max - height[right]
            right -= 1

    return total
```

---

## 高频追问

**Q：什么场景用快慢指针，什么场景用对撞指针？**
> - **对撞指针**：需要从两端向中间逼近，常见于有序数组、求最值、回文判断
> - **快慢指针**：需要原地修改数组（去重、移除）、或检测链表中的环

**Q：接雨水有哪几种解法？**
> 1. 暴力：每个位置分别向左、向右找最大值，O(n²)，直观但慢
> 2. 预处理数组：先计算 leftMax[i] 和 rightMax[i] 数组，O(n) 时间 O(n) 空间
> 3. 双指针：O(n) 时间 O(1) 空间，最优
> 4. 单调栈：也是 O(n)，适合理解"凹槽"形状的计算

**Q：盛水和接雨水有什么本质区别？**
> 盛水是在两根柱子之间装水，高度取 min，宽度是两端距离；接雨水是柱子之间的每个凹坑都能接水，要逐位置计算水深再求和。前者是一次求最大，后者是求总和。
