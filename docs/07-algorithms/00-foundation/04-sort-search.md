# 排序与查找高频题

> 考察频率：★★★★☆  难度：简单~中等  Go + Python 双语

---

## 题目导航

| # | 题目 | 难度 | 考察频率 |
|---|------|------|---------|
| 1 | [排序算法原理速查](#1-排序算法原理速查) | — | ★★★★★ |
| 2 | [搜索旋转排序数组](#2-搜索旋转排序数组-leetcode-33) | 中等 | ★★★★★ |
| 3 | [在排序数组中查找元素的第一个和最后一个位置](#3-查找第一个和最后一个位置-leetcode-34) | 中等 | ★★★★☆ |
| 4 | [寻找峰值](#4-寻找峰值-leetcode-162) | 中等 | ★★★☆☆ |

---

## 1. 排序算法原理速查

> 面试常问：讲一讲快排/归并/堆排序的原理，时间复杂度是多少？

### 常见排序对比

| 算法 | 平均时间 | 最坏时间 | 空间 | 稳定性 | 适用场景 |
|------|---------|---------|------|--------|---------|
| 快速排序 | O(n log n) | O(n²) | O(log n) | 不稳定 | 通用，实践最快 |
| 归并排序 | O(n log n) | O(n log n) | O(n) | 稳定 | 链表排序、外排序 |
| 堆排序 | O(n log n) | O(n log n) | O(1) | 不稳定 | 原地，不需要额外空间 |
| 插入排序 | O(n²) | O(n²) | O(1) | 稳定 | 小数组，几乎有序时最快 |

### 快速排序（核心思路）

**分治**：选一个"基准"值，把比它小的放左边，比它大的放右边，递归处理两边。

```
[3, 6, 8, 10, 1, 2, 1]
选 pivot=3

一趟分区后：[1, 1, 2, | 3 | 6, 8, 10]

递归排序左边 [1,1,2] 和右边 [6,8,10]
```

### Go 代码（快速排序）

```go
func quickSort(nums []int, left, right int) {
    if left >= right {
        return
    }
    pivot := partition(nums, left, right)
    quickSort(nums, left, pivot-1)
    quickSort(nums, pivot+1, right)
}

func partition(nums []int, left, right int) int {
    pivot := nums[right] // 选最右边为基准
    i := left - 1        // i 指向小于 pivot 的区域末尾
    for j := left; j < right; j++ {
        if nums[j] <= pivot {
            i++
            nums[i], nums[j] = nums[j], nums[i]
        }
    }
    nums[i+1], nums[right] = nums[right], nums[i+1]
    return i + 1
}
```

### Python 代码（快速排序）

```python
def quick_sort(nums: list[int], left: int, right: int) -> None:
    if left >= right:
        return
    pivot_idx = partition(nums, left, right)
    quick_sort(nums, left, pivot_idx - 1)
    quick_sort(nums, pivot_idx + 1, right)

def partition(nums: list[int], left: int, right: int) -> int:
    pivot = nums[right]
    i = left - 1
    for j in range(left, right):
        if nums[j] <= pivot:
            i += 1
            nums[i], nums[j] = nums[j], nums[i]
    nums[i + 1], nums[right] = nums[right], nums[i + 1]
    return i + 1
```

### 归并排序（核心思路）

**分治 + 合并**：把数组对半拆分，分别排好，再合并两个有序数组。

```go
func mergeSort(nums []int) []int {
    if len(nums) <= 1 {
        return nums
    }
    mid := len(nums) / 2
    left := mergeSort(nums[:mid])
    right := mergeSort(nums[mid:])
    return mergeSorted(left, right)
}

func mergeSorted(a, b []int) []int {
    result := make([]int, 0, len(a)+len(b))
    i, j := 0, 0
    for i < len(a) && j < len(b) {
        if a[i] <= b[j] {
            result = append(result, a[i])
            i++
        } else {
            result = append(result, b[j])
            j++
        }
    }
    result = append(result, a[i:]...)
    result = append(result, b[j:]...)
    return result
}
```

```python
def merge_sort(nums: list[int]) -> list[int]:
    if len(nums) <= 1:
        return nums
    mid = len(nums) // 2
    left = merge_sort(nums[:mid])
    right = merge_sort(nums[mid:])
    return merge_sorted(left, right)

def merge_sorted(a: list[int], b: list[int]) -> list[int]:
    result, i, j = [], 0, 0
    while i < len(a) and j < len(b):
        if a[i] <= b[j]:
            result.append(a[i]); i += 1
        else:
            result.append(b[j]); j += 1
    return result + a[i:] + b[j:]
```

---

## 2. 搜索旋转排序数组（LeetCode 33）

### 题意
有序数组被旋转（如 `[4,5,6,7,0,1,2]`），在其中查找目标值，要求 O(log n)。

**例子：** `nums = [4,5,6,7,0,1,2]`，`target = 0` → `4`

### 思路讲解
二分查找的变体。关键观察：**旋转后，mid 左边或右边至少有一侧是有序的。**

每次 mid 将数组分成两半，判断哪边有序，然后判断 target 是否在有序那边：

```
[4, 5, 6, 7, 0, 1, 2], target=0
 ↑left      ↑mid     ↑right

nums[mid]=7 > nums[left]=4 → 左半段 [4,5,6,7] 有序
target=0 不在 [4,7] 范围内 → 去右半段找
left = mid+1

[0, 1, 2], target=0
 ↑left ↑mid ↑right
nums[mid]=1 > nums[left]=0 → 左半段有序
target=0 在 [0,1] 范围内 → 去左半段找
right = mid-1

[0], target=0 → 找到，返回下标
```

### Go 代码

```go
func search(nums []int, target int) int {
    left, right := 0, len(nums)-1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        }
        // 左半段有序
        if nums[left] <= nums[mid] {
            if nums[left] <= target && target < nums[mid] {
                right = mid - 1 // target 在左半段
            } else {
                left = mid + 1 // target 在右半段
            }
        } else {
            // 右半段有序
            if nums[mid] < target && target <= nums[right] {
                left = mid + 1 // target 在右半段
            } else {
                right = mid - 1 // target 在左半段
            }
        }
    }
    return -1
}
```

### Python 代码

```python
def search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = (left + right) // 2
        if nums[mid] == target:
            return mid
        # 左半段有序
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        else:
            # 右半段有序
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1
    return -1
```

---

## 3. 查找第一个和最后一个位置（LeetCode 34）

### 题意
在有序数组中找到目标值的第一个和最后一个位置，没有则返回 `[-1, -1]`。

**例子：** `nums = [5,7,7,8,8,10]`，`target = 8` → `[3, 4]`

### 思路讲解
**两次二分**：分别找左边界和右边界。

找**左边界**：找到 target 时不停，继续往左缩（right = mid - 1），直到找到最左的位置。
找**右边界**：找到 target 时不停，继续往右扩（left = mid + 1），直到找到最右的位置。

### Go 代码

```go
func searchRange(nums []int, target int) []int {
    return []int{findLeft(nums, target), findRight(nums, target)}
}

func findLeft(nums []int, target int) int {
    left, right, result := 0, len(nums)-1, -1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            result = mid
            right = mid - 1 // 继续往左找
        } else if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return result
}

func findRight(nums []int, target int) int {
    left, right, result := 0, len(nums)-1, -1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            result = mid
            left = mid + 1 // 继续往右找
        } else if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return result
}
```

### Python 代码

```python
def searchRange(nums: list[int], target: int) -> list[int]:
    def find_left() -> int:
        left, right, res = 0, len(nums) - 1, -1
        while left <= right:
            mid = (left + right) // 2
            if nums[mid] == target:
                res = mid
                right = mid - 1  # 继续往左
            elif nums[mid] < target:
                left = mid + 1
            else:
                right = mid - 1
        return res

    def find_right() -> int:
        left, right, res = 0, len(nums) - 1, -1
        while left <= right:
            mid = (left + right) // 2
            if nums[mid] == target:
                res = mid
                left = mid + 1  # 继续往右
            elif nums[mid] < target:
                left = mid + 1
            else:
                right = mid - 1
        return res

    return [find_left(), find_right()]
```

---

## 4. 寻找峰值（LeetCode 162）

### 题意
峰值是严格大于相邻元素的元素，找到任意一个峰值的下标，要求 O(log n)。

**例子：** `nums = [1,2,3,1]` → `2`（下标，nums[2]=3 是峰值）

### 思路讲解
**二分思想**：若 `nums[mid] < nums[mid+1]`，说明右侧还在上坡，峰值在右边；
若 `nums[mid] > nums[mid+1]`，说明已经开始下坡，峰值在左边（含 mid）。

```
[1, 2, 3, 1]
      ↑mid=1, nums[1]=2 < nums[2]=3 → 往右找

[3, 1]
 ↑mid=2, nums[2]=3 > nums[3]=1 → 往左找（含mid）
left=right=2, 返回 2
```

### Go 代码

```go
func findPeakElement(nums []int) int {
    left, right := 0, len(nums)-1
    for left < right {
        mid := left + (right-left)/2
        if nums[mid] < nums[mid+1] {
            left = mid + 1 // 上坡，峰值在右
        } else {
            right = mid // 下坡，峰值在左（含mid）
        }
    }
    return left
}
```

### Python 代码

```python
def findPeakElement(nums: list[int]) -> int:
    left, right = 0, len(nums) - 1
    while left < right:
        mid = (left + right) // 2
        if nums[mid] < nums[mid + 1]:
            left = mid + 1  # 上坡，右边有峰
        else:
            right = mid     # 下坡，左边有峰（含mid）
    return left
```

---

## 高频追问

**Q：快排最坏情况是什么？怎么优化？**
> 最坏情况：数组已经有序或全部相同，每次 pivot 选到最大/最小，退化为 O(n²)。
> 优化：随机选 pivot（`rand.Intn`）、三数取中法（取首、中、尾的中位数作为 pivot）。

**Q：二分查找 `left + (right - left) / 2` 为什么不写 `(left + right) / 2`？**
> `left + right` 可能整数溢出（两个大 int 相加超过 MaxInt）。用 `left + (right-left)/2` 等价但安全。

**Q：什么时候用 `left < right` 结束，什么时候用 `left <= right`？**
> - `left <= right`：标准二分，找一个确定的值，范围缩到单个元素时必须再判断一次
> - `left < right`：找边界/峰值时，循环结束时 `left == right`，指向答案，不需要额外判断
