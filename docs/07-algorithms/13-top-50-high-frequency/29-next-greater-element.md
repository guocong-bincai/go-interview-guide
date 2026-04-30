# 29. 下一个更大元素（Next Greater Element）

> 频率：★★★★☆  难度：中等

## 小白先理解
对于每个元素，找它右边第一个比它大的数。
如果暴力往右找，每个都找一遍会很慢。

## 解法一：暴力枚举
- 每个元素都往右扫

## 解法二：单调栈（推荐）
### 核心思路
维护一个“从栈底到栈顶递减”的栈。
新元素一来，如果比栈顶大，就说明它是栈顶元素的下一个更大元素。

### Go
```go
func nextGreaterElement(nums []int) []int {
    res := make([]int, len(nums))
    for i := range res { res[i] = -1 }
    stack := []int{}
    for i := 0; i < len(nums); i++ {
        for len(stack) > 0 && nums[i] > nums[stack[len(stack)-1]] {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            res[idx] = nums[i]
        }
        stack = append(stack, i)
    }
    return res
}
```
### Python
```python
def next_greater_element(nums):
    res = [-1] * len(nums)
    stack = []
    for i, num in enumerate(nums):
        while stack and num > nums[stack[-1]]:
            idx = stack.pop()
            res[idx] = num
        stack.append(i)
    return res
```

## 一句话记忆
**找右边第一个更大，优先想单调栈。**
