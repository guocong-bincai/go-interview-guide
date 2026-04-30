# 30. 柱状图中最大的矩形（Largest Rectangle in Histogram）

> 频率：★★★★☆  难度：困难  LeetCode 84

## 小白先理解
以每根柱子为“最低高度”，看它最多能向左右扩多宽。
面积 = 高度 × 宽度。

## 解法一：暴力扩展左右边界
- 以每个柱子为中心向两边扩
- 时间复杂度高

## 解法二：单调栈（推荐）
### 核心思路
找到每根柱子左边第一个更矮、右边第一个更矮的位置。
这样它能延伸的最大宽度就知道了。

### Go
```go
func largestRectangleArea(heights []int) int {
    stack := []int{}
    maxArea := 0
    heights = append([]int{0}, heights...)
    heights = append(heights, 0)

    for i := 0; i < len(heights); i++ {
        for len(stack) > 0 && heights[i] < heights[stack[len(stack)-1]] {
            h := heights[stack[len(stack)-1]]
            stack = stack[:len(stack)-1]
            w := i - stack[len(stack)-1] - 1
            area := h * w
            if area > maxArea {
                maxArea = area
            }
        }
        stack = append(stack, i)
    }
    return maxArea
}
```
### Python
```python
def largest_rectangle_area(heights):
    heights = [0] + heights + [0]
    stack = []
    max_area = 0
    for i, h in enumerate(heights):
        while stack and h < heights[stack[-1]]:
            height = heights[stack.pop()]
            width = i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)
    return max_area
```

## 一句话记忆
**柱状图最大矩形 = 以每根柱子为最低点，单调栈找边界。**
