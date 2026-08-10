# 44. 颜色分类（Sort Colors）

> 频率：★★★★☆  难度：中等  LeetCode 75

## 小白先理解
数组中只有 0、1、2 三种数，要把它们原地排好序。

这题其实就是：
- 0 放左边
- 2 放右边
- 1 留中间

## 解法一：计数排序
- 先数 0、1、2 各多少个
- 再覆盖回去

## 解法二：三指针（推荐）
### 核心思路
荷兰国旗问题。
维护三个区域：
- 左边都是 0
- 中间都是 1
- 右边都是 2

### Go
```go
func sortColors(nums []int) {
    zero, i, two := 0, 0, len(nums)-1
    for i <= two {
        if nums[i] == 0 {
            nums[i], nums[zero] = nums[zero], nums[i]
            zero++
            i++
        } else if nums[i] == 2 {
            nums[i], nums[two] = nums[two], nums[i]
            two--
        } else {
            i++
        }
    }
}
```
### Python
```python
def sort_colors(nums):
    zero, i, two = 0, 0, len(nums) - 1
    while i <= two:
        if nums[i] == 0:
            nums[i], nums[zero] = nums[zero], nums[i]
            zero += 1
            i += 1
        elif nums[i] == 2:
            nums[i], nums[two] = nums[two], nums[i]
            two -= 1
        else:
            i += 1
```

## 一句话记忆
**颜色分类 = 荷兰国旗三指针。**
