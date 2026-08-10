# 26. 全排列（Permutations）

> 频率：★★★★☆  难度：中等  LeetCode 46

## 小白先理解
从一堆数字里，每次选一个放进路径里，直到所有数字都选完。
这就是典型回溯。

## 解法一：递归 + 新数组
- 每次传剩余元素
- 直观，但拷贝多

## 解法二：回溯 + used 数组（推荐）
### Go
```go
func permute(nums []int) [][]int {
    res := [][]int{}
    path := []int{}
    used := make([]bool, len(nums))
    var backtrack func()
    backtrack = func() {
        if len(path) == len(nums) {
            tmp := append([]int{}, path...)
            res = append(res, tmp)
            return
        }
        for i := 0; i < len(nums); i++ {
            if used[i] { continue }
            used[i] = true
            path = append(path, nums[i])
            backtrack()
            path = path[:len(path)-1]
            used[i] = false
        }
    }
    backtrack()
    return res
}
```
### Python
```python
def permute(nums):
    res = []
    path = []
    used = [False] * len(nums)
    def backtrack():
        if len(path) == len(nums):
            res.append(path[:])
            return
        for i in range(len(nums)):
            if used[i]:
                continue
            used[i] = True
            path.append(nums[i])
            backtrack()
            path.pop()
            used[i] = False
    backtrack()
    return res
```

## 一句话记忆
**全排列 = for 循环枚举选择 + 回溯撤销选择。**
