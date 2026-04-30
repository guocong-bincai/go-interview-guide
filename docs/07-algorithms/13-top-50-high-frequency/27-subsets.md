# 27. 子集（Subsets）

> 频率：★★★★☆  难度：中等  LeetCode 78

## 小白先理解
每个元素都只有两种选择：选 or 不选。
所以子集问题天然适合回溯。

## 解法一：递归二叉树选择
- 每个数都分“选”和“不选”

## 解法二：回溯枚举起点（推荐）
### Go
```go
func subsets(nums []int) [][]int {
    res := [][]int{}
    path := []int{}
    var backtrack func(int)
    backtrack = func(start int) {
        tmp := append([]int{}, path...)
        res = append(res, tmp)
        for i := start; i < len(nums); i++ {
            path = append(path, nums[i])
            backtrack(i + 1)
            path = path[:len(path)-1]
        }
    }
    backtrack(0)
    return res
}
```
### Python
```python
def subsets(nums):
    res = []
    path = []
    def backtrack(start):
        res.append(path[:])
        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1)
            path.pop()
    backtrack(0)
    return res
```

## 一句话记忆
**子集问题 = 每层决定后面还能选谁。**
