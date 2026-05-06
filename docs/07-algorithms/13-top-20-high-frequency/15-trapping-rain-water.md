# 15. 接雨水（Trapping Rain Water）

> 频率：★★★★★  难度：困难  LeetCode 42

## 题目描述
给定 `n` 个非负整数表示每个宽度为 1 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。

## 小白先理解
每个位置能装多少水，取决于：
- 左边最高柱子
- 右边最高柱子

当前位置能装的水量：

```text
min(左边最高, 右边最高) - 当前高度
```

如果结果小于等于 0，就装不了水。

---

## 解法一：前后缀最大值

### 思路
提前算好：
- `leftMax[i]`：i 左边最高柱子
- `rightMax[i]`：i 右边最高柱子

然后每个位置直接套公式。

### Go
```go
func trapPrefix(height []int) int {
    n := len(height)
    if n == 0 {
        return 0
    }

    leftMax := make([]int, n)
    rightMax := make([]int, n)

    leftMax[0] = height[0]
    for i := 1; i < n; i++ {
        if leftMax[i-1] > height[i] {
            leftMax[i] = leftMax[i-1]
        } else {
            leftMax[i] = height[i]
        }
    }

    rightMax[n-1] = height[n-1]
    for i := n - 2; i >= 0; i-- {
        if rightMax[i+1] > height[i] {
            rightMax[i] = rightMax[i+1]
        } else {
            rightMax[i] = height[i]
        }
    }

    ans := 0
    for i := 0; i < n; i++ {
        water := min(leftMax[i], rightMax[i]) - height[i]
        ans += water
    }
    return ans
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

### Python
```python
def trap_prefix(height):
    if not height:
        return 0

    n = len(height)
    left_max = [0] * n
    right_max = [0] * n

    left_max[0] = height[0]
    for i in range(1, n):
        left_max[i] = max(left_max[i - 1], height[i])

    right_max[-1] = height[-1]
    for i in range(n - 2, -1, -1):
        right_max[i] = max(right_max[i + 1], height[i])

    ans = 0
    for i in range(n):
        ans += min(left_max[i], right_max[i]) - height[i]
    return ans
```

---

## 解法二：双指针（推荐）

### 核心思路
左右两边各放一个指针。

如果左边较低：
- 当前能装多少水，只看左边最大值

如果右边较低：
- 当前能装多少水，只看右边最大值

这样就不用额外数组了。

### Go
```go
func trap(height []int) int {
    left, right := 0, len(height)-1
    leftMax, rightMax := 0, 0
    ans := 0

    for left < right {
        if height[left] < height[right] {
            if height[left] >= leftMax {
                leftMax = height[left]
            } else {
                ans += leftMax - height[left]
            }
            left++
        } else {
            if height[right] >= rightMax {
                rightMax = height[right]
            } else {
                ans += rightMax - height[right]
            }
            right--
        }
    }
    return ans
}
```

### Python
```python
def trap(height):
    left, right = 0, len(height) - 1
    left_max, right_max = 0, 0
    ans = 0

    while left < right:
        if height[left] < height[right]:
            if height[left] >= left_max:
                left_max = height[left]
            else:
                ans += left_max - height[left]
            left += 1
        else:
            if height[right] >= right_max:
                right_max = height[right]
            else:
                ans += right_max - height[right]
            right -= 1

    return ans
```

## 一句话记忆
**接雨水核心公式：当前位置水量 = min(左高, 右高) - 当前高。**
