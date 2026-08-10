# 48. 组合总和（Combination Sum）

> 频率：★★★☆☆  难度：中等  LeetCode 39

## 小白先理解
给你一组数字，可以重复选，问有多少种组合能凑成 target。

## 解法一：暴力枚举
- 所有情况都试

## 解法二：回溯（推荐）
### 核心思路
每次从当前位置开始选：
- 选当前数，目标变小
- 还能继续选当前数（因为可以重复）
- 如果目标变成 0，记录答案

### Go
```go
func combinationSum(candidates []int, target int) [][]int {
    res := [][]int{}
    path := []int{}
    var backtrack func(int, int)
    backtrack = func(start, remain int) {
        if remain == 0 {
            tmp := append([]int{}, path...)
            res = append(res, tmp)
            return
        }
        if remain < 0 { return }
        for i := start; i < len(candidates); i++ {
            path = append(path, candidates[i])
            backtrack(i, remain-candidates[i])
            path = path[:len(path)-1]
        }
    }
    backtrack(0, target)
    return res
}
```
### Python
```python
def combination_sum(candidates, target):
    res = []
    path = []
    def backtrack(start, remain):
        if remain == 0:
            res.append(path[:])
            return
        if remain < 0:
            return
        for i in range(start, len(candidates)):
            path.append(candidates[i])
            backtrack(i, remain - candidates[i])
            path.pop()
    backtrack(0, target)
    return res
```

## 一句话记忆
**组合总和 = 回溯，且可以重复选当前数。**
