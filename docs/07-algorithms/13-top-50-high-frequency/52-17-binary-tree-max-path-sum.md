# 17. 二叉树最大路径和（Binary Tree Maximum Path Sum）

> 频率：★★★★☆  难度：困难  LeetCode 124

## 题目描述
给你一个二叉树的根节点 `root`，返回任意路径的最大路径和。

路径不一定经过根节点，但路径上的节点不能重复。

## 小白先理解
这题不是只算“从根到叶子”的路径。
它允许：
- 从左子树某点开始
- 经过一个节点
- 再走到右子树某点结束

所以某个节点有可能成为“拐点”。

---

## 解法一：暴力想法
枚举所有路径，求最大和。

### 问题
几乎不可行，路径太多。

---

## 解法二：树形 DP（推荐）

### 核心思路
对每个节点，考虑两件事：

1. **往上返回给父节点的最大贡献值**
   - 只能走一边
   - 因为一条路径不能在父节点那里再分叉

2. **把当前节点当拐点时的路径和**
   - 左贡献 + 当前值 + 右贡献
   - 这可能更新全局最大值

### Go
```go
import "math"

func maxPathSum(root *TreeNode) int {
    ans := math.MinInt32

    var dfs func(*TreeNode) int
    dfs = func(node *TreeNode) int {
        if node == nil {
            return 0
        }

        left := max(dfs(node.Left), 0)
        right := max(dfs(node.Right), 0)

        ans = max(ans, node.Val+left+right)

        return node.Val + max(left, right)
    }

    dfs(root)
    return ans
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

### Python
```python
def max_path_sum(root):
    ans = float('-inf')

    def dfs(node):
        nonlocal ans
        if not node:
            return 0

        left = max(dfs(node.left), 0)
        right = max(dfs(node.right), 0)

        ans = max(ans, node.val + left + right)

        return node.val + max(left, right)

    dfs(root)
    return ans
```

## 一句话记忆
**树形 DP：返回单边贡献，全局更新双边路径。**
