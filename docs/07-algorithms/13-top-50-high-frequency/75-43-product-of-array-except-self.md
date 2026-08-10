# 43. 除自身以外数组的乘积（Product of Array Except Self）

> 频率：★★★★☆  难度：中等  LeetCode 238

## 小白先理解
对于每个位置 i，要得到“除了 nums[i] 自己以外，其他所有数字的乘积”。

不能直接对每个位置都重新乘一遍，那会太慢。

## 解法一：暴力枚举
- 每个位置都乘一遍其他元素

## 解法二：前后缀积（推荐）
### 核心思路
- `left[i]` 表示 i 左边所有数的乘积
- `right[i]` 表示 i 右边所有数的乘积
- 答案就是 `left[i] * right[i]`

### Go
```go
func productExceptSelf(nums []int) []int {
    n := len(nums)
    res := make([]int, n)
    res[0] = 1
    for i := 1; i < n; i++ {
        res[i] = res[i-1] * nums[i-1]
    }
    right := 1
    for i := n - 1; i >= 0; i-- {
        res[i] *= right
        right *= nums[i]
    }
    return res
}
```
### Python
```python
def product_except_self(nums):
    n = len(nums)
    res = [1] * n
    for i in range(1, n):
        res[i] = res[i - 1] * nums[i - 1]
    right = 1
    for i in range(n - 1, -1, -1):
        res[i] *= right
        right *= nums[i]
    return res
```

## 一句话记忆
**除自身外乘积 = 左乘积 × 右乘积。**
