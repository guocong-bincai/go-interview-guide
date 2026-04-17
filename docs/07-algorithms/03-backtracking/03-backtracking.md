# 回溯算法

> 考察频率：★★★★☆  题型识别：排列/组合/子集 + 暴力枚举所有解

## 核心模板

回溯 = 暴力枚举 + **剪枝**。DFS 遍历决策树，走到叶子收集解，过程中通过剪枝避免无效搜索。

### 通用框架

```go
func backtrack(路径, 当前选择列表, 路径记录) {
    if 满足结束条件 {
        记录解（copy一份避免污染）
        return
    }

    for _, 选择 := range 当前选择列表 {
        // 剪枝：跳过不合法选择
        if !isValid(选择) {
            continue
        }

        // 做选择
        路径 = append(路径, 选择)
        used[选择] = true

        // 递归
        backtrack(路径, 剩余选择, 记录)

        // 撤销选择（回溯）
        路径 = 路径[:len(路径)-1]
        used[选择] = false
    }
}
```

**核心要点**：
1. `used` 数组或 `start` 索引控制可选范围
2. 收集结果时 `append(tmp, 路径...)` 避免引用同一个 slice
3. 排序 + 剪枝可以大幅减少搜索量

---

## 题型一：子集（LeetCode 78）

### 题目
给定数组 `nums`，返回所有可能的子集（幂集）。

### 思路
每个元素有"选"和"不选"两种选择，生成 $2^n$ 个子集。按索引顺序 DFS，保证不重不漏。

### 代码

```go
func subsets(nums []int) [][]int {
    var result [][]int
    var path []int

    var dfs func(start int)
    dfs = func(start int) {
        // 收集当前路径（每个节点都是解）
        tmp := make([]int, len(path))
        copy(tmp, path)
        result = append(result, tmp)

        // 遍历选择列表
        for i := start; i < len(nums); i++ {
            path = append(path, nums[i])
            dfs(i + 1) // 注意是 i+1，不是 start+1（避免重复）
            path = path[:len(path)-1]
        }
    }

    dfs(0)
    return result
}
```

**输出示例**（nums=[1,2,3]）：
```
[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]
```

### 复杂度
- 时间：O(n * 2^n)，每个子集 O(n) 复制
- 空间：O(n) 递归栈

---

## 题型二：子集 II（LeetCode 90）—— 含重复元素

### 题目
nums 可能包含重复元素，要求子集不能重复。

### 核心思路
**排序后 + 同一层跳过相同元素**

```go
func subsetsWithDup(nums []int) [][]int {
    sort.Ints(nums) // 排序是去重的关键！
    var result [][]int
    var path []int

    var dfs func(start int)
    dfs = func(start int) {
        tmp := make([]int, len(path))
        copy(tmp, path)
        result = append(result, tmp)

        for i := start; i < len(nums); i++ {
            // 剪枝：同一层跳过相同的数
            if i > start && nums[i] == nums[i-1] {
                continue
            }
            path = append(path, nums[i])
            dfs(i + 1)
            path = path[:len(path)-1]
        }
    }

    dfs(0)
    return result
}
```

---

## 题型三：全排列（LeetCode 46）

### 题目
给定不含重复数字的数组，返回所有排列。

### 思路
用 `used` 数组标记哪些元素已被使用，DFS 收集所有排列。

```go
func permute(nums []int) [][]int {
    var result [][]int
    path := make([]int, 0, len(nums))
    used := make([]bool, len(nums))

    var dfs func()
    dfs = func() {
        if len(path) == len(nums) {
            tmp := make([]int, len(path))
            copy(tmp, path)
            result = append(result, tmp)
            return
        }

        for i := 0; i < len(nums); i++ {
            if used[i] {
                continue
            }
            path = append(path, nums[i])
            used[i] = true
            dfs()
            path = path[:len(path)-1]
            used[i] = false
        }
    }

    dfs()
    return result
}
```

### 追问：含重复元素的全排列（LeetCode 47）
→ 排序后，同一层跳过相同元素（类似子集 II 的剪枝）。

---

## 题型四：组合（LeetCode 77）

### 题目
给定 n 和 k，返回 `[1..n]` 中所有 k 个数的组合。

### 代码

```go
func combine(n int, k int) [][]int {
    var result [][]int
    path := make([]int, 0, k)

    var dfs func(start int)
    dfs = func(start int) {
        if len(path) == k {
            tmp := make([]int, k)
            copy(tmp, path)
            result = append(result, tmp)
            return
        }

        // 剪枝：剩余元素不够
        for i := start; i <= n; i++ {
            // 可选数量不够，直接剪枝
            if n-i+1 < k-len(path) {
                break
            }
            path = append(path, i)
            dfs(i + 1)
            path = path[:len(path)-1]
        }
    }

    dfs(1)
    return result
}
```

### 关键剪枝
```
剩余可选数量: n - i + 1
还需数量: k - len(path)
若 n - i + 1 < k - len(path) → 剪枝
```

---

## 题型五：N 皇后（LeetCode 51）

### 题目
在 n×n 棋盘上放 n 个皇后，互相不攻击。返回所有解。

### 核心思路
按行 DFS，每行选一个列位置，检查是否与已放置的皇后冲突（列冲突 + 对角线冲突）。

### 代码

```go
func solveNQueens(n int) [][]string {
    var result [][]string
    board := make([][]byte, n)
    for i := range board {
        board[i] = make([]byte, n)
        for j := range board[i] {
            board[i][j] = '.'
        }
    }

    cols := make([]bool, n)       // 列是否已有皇后
    diags1 := make([]bool, 2*n)    // 对角线 ↘，下标 = row-col+n
    diags2 := make([]bool, 2*n)   // 对角线 ↙，下标 = row+col

    var dfs func(row int)
    dfs = func(row int) {
        if row == n {
            // 收集解
            sol := make([]string, n)
            for i := 0; i < n; i++ {
                sol[i] = string(board[i])
            }
            result = append(result, sol)
            return
        }

        for col := 0; col < n; col++ {
            d1 := row - col + n // row-col可能为负，偏移n
            d2 := row + col
            if cols[col] || diags1[d1] || diags2[d2] {
                continue
            }
            board[row][col] = 'Q'
            cols[col], diags1[d1], diags2[d2] = true, true, true
            dfs(row + 1)
            board[row][col] = '.'
            cols[col], diags1[d1], diags2[d2] = false, false, false
        }
    }

    dfs(0)
    return result
}
```

### 复杂度
- 时间：O(n!)，实际剪枝后远小于 n!
- 空间：O(n) 递归栈 + O(n²) 棋盘

---

## 题型六：括号生成（LeetCode 22）

### 题目
生成 n 对括号的所有合法组合。

### 思路
用 `left`、`right` 分别表示已使用的左/右括号数。合法条件：
- `left < right` → 不合法（右括号不能比左括号多）
- `left <= n`，`right <= n`

```go
func generateParenthesis(n int) []string {
    var result []string
    path := make([]byte, 0, 2*n)

    var dfs func(left, right int)
    dfs = func(left, right int) {
        if len(path) == 2*n {
            result = append(result, string(path))
            return
        }

        // 可以加左括号
        if left < n {
            path = append(path, '(')
            dfs(left+1, right)
            path = path[:len(path)-1]
        }

        // 可以加右括号（不能比左括号多）
        if right < left {
            path = append(path, ')')
            dfs(left, right+1)
            path = path[:len(path)-1]
        }
    }

    dfs(0, 0)
    return result
}
```

---

## 常见剪枝策略

| 剪枝策略 | 适用题型 | 实现方式 |
|---------|---------|---------|
| 排序 + 跳过同层相同元素 | 子集 II、全排列 II | `if i > start && nums[i] == nums[i-1]` |
| 数量不够剪枝 | 组合 | `if n-i+1 < k-len(path)` |
| 约束不满足 | N皇后、括号生成 | `if !isValid(选择)` |
| 上下界剪枝 | 组合求和 | `if sum+rest < target` |

---

## 总结

| 题型 | 关键参数 | 去重方式 |
|------|---------|---------|
| 子集 | `start` 索引 | 排序 + 同层跳过相同 |
| 全排列 | `used` 数组 | 排序 + 同层跳过相同 |
| 组合 | `start` 索引 | 不选 start 前面的（天然有序）|
| N皇后 | `cols/diags` 标记 | 三个集合检查冲突 |
| 括号生成 | `left/right` 计数 | `right < left` 不合法 |

---

## Python 实现汇总

### 全排列

```python
def permute(nums: list[int]) -> list[list[int]]:
    result, path, used = [], [], [False] * len(nums)

    def backtrack():
        if len(path) == len(nums):
            result.append(path[:])
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
    return result
```

### 子集

```python
def subsets(nums: list[int]) -> list[list[int]]:
    result, path = [], []

    def backtrack(start: int):
        result.append(path[:])    # 每次进入都是一个合法子集
        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1)
            path.pop()

    backtrack(0)
    return result
```

### 组合求和（元素可重复用）

```python
def combinationSum(candidates: list[int], target: int) -> list[list[int]]:
    result, path = [], []

    def backtrack(start: int, remain: int):
        if remain == 0:
            result.append(path[:])
            return
        for i in range(start, len(candidates)):
            if candidates[i] > remain:
                break
            path.append(candidates[i])
            backtrack(i, remain - candidates[i])  # i 不是 i+1，可重复选
            path.pop()

    candidates.sort()
    backtrack(0, target)
    return result
```

### 括号生成

```python
def generateParenthesis(n: int) -> list[str]:
    result = []

    def backtrack(s: str, left: int, right: int):
        if len(s) == 2 * n:
            result.append(s)
            return
        if left < n:
            backtrack(s + '(', left + 1, right)
        if right < left:     # 右括号数 < 左括号数才能加右括号
            backtrack(s + ')', left, right + 1)

    backtrack('', 0, 0)
    return result
```
