# 23. 搜索旋转排序数组

> 频率：★★★★☆  难度：中等  LeetCode 33

## 小白先理解
原本有序的数组，在某个点被“旋转”了。
虽然整体看起来不完全有序，但至少一半一定有序，所以还能二分。

## 解法一：先找旋转点再查找
- 先找最小值位置
- 再决定去哪一半二分

## 解法二：直接二分（推荐）
### Go
```go
func search(nums []int, target int) int {
    left, right := 0, len(nums)-1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target { return mid }
        if nums[left] <= nums[mid] {
            if nums[left] <= target && target < nums[mid] {
                right = mid - 1
            } else {
                left = mid + 1
            }
        } else {
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
### Python
```python
def search(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = (left + right) // 2
        if nums[mid] == target:
            return mid
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        else:
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1
    return -1
```

## 一句话记忆
**旋转数组二分关键：每次至少有一半是有序的。**
