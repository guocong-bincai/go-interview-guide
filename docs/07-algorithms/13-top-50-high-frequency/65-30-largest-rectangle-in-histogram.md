# 30. 柱状图中最大的矩形（Largest Rectangle in Histogram）

> 频率：★★★★☆  难度：困难  LeetCode 84

## 题目描述
给定 `n` 个非负整数代表柱状图，每个柱子的宽度为 1，求其中能勾勒出的最大矩形面积。

## 面试为什么爱问
- 这是单调栈的**最难也最经典**的应用题
- 核心思想：对每根柱子，找它作为"最低高度"时能扩多宽
- 面试官会追问：和接雨水有什么区别？怎么优化空间？

## 核心考点
- [ ] 单调递增栈
- [ ] 哨兵技巧：首尾加 0 简化边界处理
- [ ] 面积计算公式：高度 × 宽度

## 小白先理解
**本质：每根柱子作为"短板"能围多大面积**

假设有柱子 `[2, 1, 5, 6, 2, 3]`：
- 以高度 1 为短板：能扩到整个宽度 `[0, 5]`，面积 = 1 × 6 = 6
- 以高度 2 为短板：能扩到 `[0, 1)`，面积 = 2 × 2 = 4；或者 `[4, 6)`，面积 = 2 × 2 = 4
- 以高度 5 为短板：只能自己，宽度 1，面积 5

**关键洞察：找每根柱子左边第一个比它矮的、右边第一个比它矮的位置，中间就是它能扩到的最大宽度。**

单调栈就是找"最近的小于关系"的神器。

---

## 解法一：暴力扩展（会超时，了解即可）

### 思路
以每根柱子为基准，分别向左向右扩展，直到遇到更矮的。
时间复杂度 O(n²)，空间 O(1)。

### Go
```go
func largestRectangleArea_brutal(heights []int) int {
    maxArea := 0
    for i := 0; i < len(heights); i++ {
        h := heights[i]
        // 向左扩
        left := i
        for left > 0 && heights[left-1] >= h {
            left--
        }
        // 向右扩
        right := i
        for right < len(heights)-1 && heights[right+1] >= h {
            right++
        }
        area := h * (right - left + 1)
        if area > maxArea {
            maxArea = area
        }
    }
    return maxArea
}
```

---

## 解法二：单调栈（推荐）

### 核心思路
1. 在数组首尾各加一个 0（哨兵），处理边界不用特判
2. 维护一个**单调递增**的栈，存柱子索引
3. 当遇到比栈顶矮的柱子时，说明栈顶这个柱子找到了"右边第一个更矮"的位置 → 开始弹栈计算

弹栈时，栈顶元素 `popH` 是当前遇到的高度。`popH` 的"右边第一个更矮"就是当前索引 `i`，"左边第一个更矮"是弹栈后的新栈顶 `stack[len(stack)-1]`（如果有的话）。

### Go
```go
func largestRectangleArea(heights []int) int {
    maxArea := 0
    // 哨兵：首尾加 0，统一处理边界
    h := make([]int, 0, len(heights)+2)
    h = append(h, 0)
    h = append(h, heights...)
    h = append(h, 0)

    stack := []int{0} // 栈内存索引，初始化放入第一个哨兵

    for i := 1; i < len(h); i++ {
        // 当前柱比栈顶矮 → 栈顶柱子找到了右边界，开始结算
        for h[i] < h[stack[len(stack)-1]] {
            popIdx := stack[len(stack)-1]    // 要结算的柱子索引
            stack = stack[:len(stack)-1]     // 弹出
            popH := h[popIdx]                // 高度
            leftIdx := stack[len(stack)-1]   // 左边界（栈顶）
            width := i - leftIdx - 1         // 中间宽度
            area := popH * width
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
    max_area = 0
    h = [0] + heights + [0]  # 哨兵
    stack = [0]

    for i in range(1, len(h)):
        while h[i] < h[stack[-1]]:
            pop_idx = stack.pop()
            pop_h = h[pop_idx]
            left_idx = stack[-1]
            width = i - left_idx - 1
            max_area = max(max_area, pop_h * width)
        stack.append(i)

    return max_area
```

### 复杂度
- 时间：O(n)，每个柱子最多入栈出栈各一次
- 空间：O(n)，栈最多存 n+2 个元素

---

## 两种解法对比
| 维度 | 暴力 | 单调栈 |
|------|------|--------|
| 时间复杂度 | O(n²) | O(n) |
| 空间复杂度 | O(1) | O(n) |
| 面试推荐 | 了解思路即可 | **必须掌握** |

---

## 易错点
| 错误 | 问题 |
|------|------|
| 不加哨兵 | 首尾边界要特判，容易漏 |
| 栈里存高度而非索引 | 宽度计算需要索引差值 |
| `while` 写成 `if` | 弹栈要弹干净，直到栈顶比当前矮 |

---

## 面试追问

**Q：和 Trapping Rain Water（接雨水）有什么区别？**
接雨水是找"两边高中间低"凹陷；本题是找"每根柱子能扩多宽"。接雨水用双指针，本题用单调栈。

**Q：空间能否优化到 O(1)？**
不能比 O(n) 更低，因为最坏情况（递增序列）栈就要存 n 个元素。

**Q：如果柱子是乱序的怎么办？**
单调栈方法对乱序完全适用，因为每次是找"最近的小于"，和顺序无关。

---

## 一句话记忆
**柱状图最大矩形 = 以每根柱子为最低高度，单调栈找左右最近更矮边界，哨兵简化边界处理。**