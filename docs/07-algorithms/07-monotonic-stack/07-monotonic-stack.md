# 单调栈

> 考察频率：★★★☆☆  题型识别：下一个更大/更小元素、柱状图最大矩形

## 核心思想

单调栈 = **栈中元素单调递增或递减**。用于在 O(n) 时间内解决"每个元素下一个更大/更小元素"的问题。

### 什么时候用
- 数组中每个元素需要找"左边/右边第一个比它大/小的元素"
- 柱状图最大矩形面积
- 每日温度（下一个更高温度的天数）

---

## 单调栈模板

### 模板一：下一个更大元素（单调递减栈）

```go
func nextGreaterElement(nums []int) []int {
    result := make([]int, len(nums))
    for i := range result {
        result[i] = -1
    }

    stack := []int{} // 存索引，栈内对应元素单调递减

    for i := 0; i < len(nums); i++ {
        // 当前元素比栈顶大 → 栈顶的"下一个更大"就是当前元素
        for len(stack) > 0 && nums[i] > nums[stack[len(stack)-1]] {
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[top] = nums[i]
        }
        stack = append(stack, i)
    }
    return result
}
```

### 模板二：下一个更小元素（单调递增栈）

```go
func nextSmallerElement(nums []int) []int {
    result := make([]int, len(nums))
    for i := range result {
        result[i] = -1
    }

    stack := []int{} // 栈内元素单调递增

    for i := 0; i < len(nums); i++ {
        for len(stack) > 0 && nums[i] < nums[stack[len(stack)-1]] {
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[top] = nums[i]
        }
        stack = append(stack, i)
    }
    return result
}
```

---

## 题型一：每日温度（LeetCode 739）

### 题目
给定温度数组，找到每个温度需要等多少天才能等到更高的温度。

### 代码

```go
func dailyTemperatures(temperatures []int) []int {
    result := make([]int, len(temperatures))
    stack := []int{} // 存索引，单调递增（温度从栈底到栈顶递增）

    for i, t := range temperatures {
        for len(stack) > 0 && temperatures[stack[len(stack)-1]] < t {
            prev := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[prev] = i - prev
        }
        stack = append(stack, i)
    }
    return result
}
```

### 追问
如果要求"左边第一个更高温度"？
→ 从右往左遍历，维护单调递减栈。

---

## 题型二：柱状图最大矩形（LeetCode 84）

### 题目
给定柱状图每根柱子的高度，求最大矩形面积。

### 核心思路
枚举每根柱子，以它的高度为矩形高，向左向右扩展直到遇到更低的柱子。

用**单调递增栈**：栈中索引对应的高度是单调递增的。当遇到更低的柱子时，栈顶出栈，出栈的柱子高度为 `h`，它的左边界是栈中下一个元素，右边界是当前更低的柱子位置。

### 代码

```go
func largestRectangleArea(heights []int) int {
    if len(heights) == 0 {
        return 0
    }

    // 扩展数组：首尾加 0，确保所有柱子都能出栈
    ext := make([]int, len(heights)+2)
    ext[0] = 0
    ext[len(ext)-1] = 0
    copy(ext[1:], heights)

    stack := []int{0} // 栈中存索引，高度单调递增
    maxArea := 0

    for i := 1; i < len(ext); i++ {
        for ext[i] < ext[stack[len(stack)-1]] {
            h := ext[stack[len(stack)-1]]
            stack = stack[:len(stack)-1]
            width := i - stack[len(stack)-1] - 1
            maxArea = max(maxArea, h*width)
        }
        stack = append(stack, i)
    }
    return maxArea
}
```

**为什么首尾加 0？**
- 头部加 0：确保第一个元素能入栈
- 尾部加 0：强制栈中所有元素出栈，计算以它们为最高柱的矩形

### 复杂度
- 时间：O(n)，每个元素最多入栈、出栈各一次
- 空间：O(n)

---

## 题型三：接雨水（LeetCode 42）

### 题目
给定高度图，计算能接多少雨水。

### 思路：单调栈（横向按层计算）

```go
func trap(height []int) int {
    if len(height) == 0 {
        return 0
    }

    stack := []int{} // 存索引，单调递减
    water := 0

    for i := 0; i < len(height); i++ {
        for len(stack) > 0 && height[i] > height[stack[len(stack)-1]] {
            bottomIdx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]

            if len(stack) == 0 {
                break
            }

            // 此时 height[stack[-1]] 是左边界，height[i] 是右边界
            leftIdx := stack[len(stack)-1]
            w := i - leftIdx - 1
            h := min(height[leftIdx], height[i]) - height[bottomIdx]
            water += w * h
        }
        stack = append(stack, i)
    }
    return water
}
```

### 追问：双指针 O(1) 空间解法
```go
func trapTwoPointers(height []int) int {
    left, right := 0, len(height)-1
    leftMax, rightMax := 0, 0
    water := 0

    for left < right {
        if height[left] < height[right] {
            if height[left] >= leftMax {
                leftMax = height[left]
            } else {
                water += leftMax - height[left]
            }
            left++
        } else {
            if height[right] >= rightMax {
                rightMax = height[right]
            } else {
                water += rightMax - height[right]
            }
            right--
        }
    }
    return water
}
```

---

## 题型四：字符串解码（LeetCode 394）—— 单调栈应用

### 题目
如 `3[a2[c]]` 解码为 `accaccacc`。

### 思路
用**两个栈**：数字栈和字符串栈，遍历字符串：
- 遇到数字：收集完整数字
- 遇到 `[`：把当前字符串和数字入栈，重置
- 遇到 `]`：弹栈，拼接

```go
func decodeString(s string) string {
    numStack := []int{}
    strStack := []string{}
    curStr := ""
    num := 0

    for _, c := range s {
        if c >= '0' && c <= '9' {
            num = num*10 + int(c-'0')
        } else if c == '[' {
            numStack = append(numStack, num)
            strStack = append(strStack, curStr)
            curStr = ""
            num = 0
        } else if c == ']' {
            repeatTimes := numStack[len(numStack)-1]
            numStack = numStack[:len(numStack)-1]
            prevStr := strStack[len(strStack)-1]
            strStack = strStack[:len(strStack)-1]

            // 拼接
            decoded := repeatStr(curStr, repeatTimes)
            curStr = prevStr + decoded
        } else {
            curStr += string(c)
        }
    }
    return curStr
}

func repeatStr(s string, times int) string {
    result := ""
    for i := 0; i < times; i++ {
        result += s
    }
    return result
}
```

---

## 单调栈题型总结

| 题目 | 栈类型 | 核心操作 |
|------|--------|---------|
| 下一个更大元素 | 单调递减栈 | 栈顶出栈 → 得到答案 |
| 每日温度 | 单调递减栈 | 同上 |
| 柱状图最大矩形 | 单调递增栈 | 出栈时计算宽度 |
| 接雨水 | 单调递减栈 | 形成凹槽时计算 |
| 字符串解码 | 两个栈 | `[` 入栈，`]` 弹栈拼接 |

### 模板记忆
```
遍历数组：
  while 栈顶不满足单调性：
      栈顶出栈，计算答案
  当前元素入栈
```

**关键**：栈中存的是**索引**，通过索引能找到值。单调性决定了是找更大还是更小。
