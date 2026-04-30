# 36. 盛最多水的容器（Container With Most Water）

> 频率：★★★★☆  难度：中等  LeetCode 11

## 小白先理解
两根柱子和地面可以围出一个容器。
面积 = 宽度 × 较短柱子的高度。

## 解法一：暴力枚举所有两根柱子
- 每对都算一次面积

## 解法二：双指针（推荐）
### 核心思路
左右各一个指针。
每次移动较短的那根，因为面积受短板限制。

### Go
```go
func maxArea(height []int) int {
    left, right := 0, len(height)-1
    ans := 0
    for left < right {
        h := height[left]
        if height[right] < h { h = height[right] }
        area := h * (right - left)
        if area > ans { ans = area }
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }
    return ans
}
```
### Python
```python
def max_area(height):
    left, right = 0, len(height) - 1
    ans = 0
    while left < right:
        area = min(height[left], height[right]) * (right - left)
        ans = max(ans, area)
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    return ans
```

## 一句话记忆
**盛水容器 = 双指针，每次移动短板。**
