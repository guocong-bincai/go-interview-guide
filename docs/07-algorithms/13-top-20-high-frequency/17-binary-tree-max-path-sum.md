# 二叉树最大路径和

> 频率：★★★★☆  难度：困难  LeetCode 124

## 题目描述

给定一个**非空**二叉树，每个节点有一个整数值（可正可负可为零）。找到一条路径上所有节点值之和最大的那条路径。**路径不一定经过根节点，也不一定经过叶子节点**——只要是树中的连续路径即可。

**连续路径**：从任意节点出发，向下走到子节点（或不走），构成一条路径。

**示例**：

```
Input:
       -10
      /   \
     9     20
          /  \
         15   7

路径：9 + (-10) + 20 + 15 = 34（不一定从根开始）
Output: 34
```

更复杂的例子：

```
Input:
        1
       / \
      2   3
     /
    4

最大路径和：4 + 2 + 1 + 3 = 10
（路径是 4→2→1→3，不经过根节点也不经过叶子节点）
```

## 面试为什么爱问

这道题是**树形 DP 的经典题**，考察两个核心能力：
1. **递归思维**：如何把大问题拆成子问题，如何定义返回值
2. **全局 vs 局部**：需要同时维护"包含当前节点的最优路径"和"全局最优路径"

面试官通常会追问：为什么不能直接返回 max(left, right) + root.Val？——因为路径可以不往上传。

## 核心考点

- [x] 二叉树后序遍历（左右根）
- [x] DFS 递归函数设计
- [x] 后序位置处理（关键！）
- [x] 全局变量 vs 返回值
- [x] 负数处理

## 小白先理解

### 什么是一条"路径"？

在二叉树里，路径是从某个节点出发，向下走（只能往子节点走，不能往回走父节点）的一条线段。

```
      A
     / \
    B   C
   / \   \
  D   E   F

哪些是合法路径？
✅ A→B→D（A往下走，一直往下）
✅ E→B→A→C（不行！A是E的父节点，往上走了）
✅ B→A→C（A 是转折点，可以！A 同时连接 B 和 C）
✅ D→B→E（可以！都往下走）
```

**关键**：路径可以"拐弯"（经过一个节点往下走两个分支），但不能"回头"。

### 为什么不能只返回 max(left, right)？

```
      -10
      /  \
     9    20
          / \
         15  7

最大路径是 15+20+(-10)+9 = 34
这条路径经过了根节点 -10

如果用 max(left, right) + root.Val = max(9, 20+15+7=32) + (-10) = 22
算出来的只是"以 -10 为拐点往下走的路径和"，而不是整棵树中的最优路径
```

所以递归函数需要**同时**返回两个东西：
1. **往上传的最大路径**（只能从当前节点的左或右选一条，往上继续）
2. **全局最优路径**（可以是任意子树中的任意路径）

## 解法一：后序遍历 + 全局变量（推荐）

### 核心思路

用**后序遍历**处理每个节点。在后序位置（左右子树都处理完了），计算经过当前节点的"拐弯路径"的最大值，和"往上走的路径"的最大值。

**经过当前节点的路径**（可能拐弯）：
```
     左子树提供    根节点    右子树提供
       ↓          ↓          ↓
    ┌──────┐   ┌──────┐   ┌──────┐
    │ left │ + │ root │ + │ right │
    └──────┘   └──────┘   └──────┘
```

**往上走的路径**（只能选一条分支）：
```
max(left, right, 0) + root.Val
（如果左/右子树提供负数，不如不走，所以要和 0 比）
```

### Go
```go
package main

import "math"

type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func maxPathSum(root *TreeNode) int {
    if root == nil {
        return 0
    }
    maxSum := math.MinInt // 全局最优

    // 返回值：经过 root，往上能提供的最大路径和
    var dfs func(*TreeNode) int
    dfs = func(node *TreeNode) int {
        if node == nil {
            return 0
        }

        // 后序位置：左右子树都处理完了
        // 左/右子树能提供的最大路径（如果负数就不要）
        leftGain := max(dfs(node.Left), 0)
        rightGain := max(dfs(node.Right), 0)

        // 经过 node 的完整路径（node 是拐点，左右都走）
        // 这是一种"局部路径"：从左子树一路走到右子树
        pathThroughNode := node.Val + leftGain + rightGain

        // 更新全局最优
        maxSum = max(maxSum, pathThroughNode)

        // 返回：经过 node，往上走的最大路径（只能选一条分支）
        // 因为往上传的时候不能同时走两条路
        return node.Val + max(leftGain, rightGain)
    }

    dfs(root)
    return maxSum
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
import math

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def maxPathSum(root: TreeNode) -> int:
    max_sum = -math.inf

    def dfs(node: TreeNode) -> int:
        nonlocal max_sum
        if not node:
            return 0

        # 后序遍历：左右根
        left_gain = max(dfs(node.left), 0)
        right_gain = max(dfs(node.right), 0)

        # 经过 node 的完整路径（node 是拐点）
        path_through_node = node.val + left_gain + right_gain
        max_sum = max(max_sum, path_through_node)

        # 返回往上走的最大路径（只能选一条分支）
        return node.val + max(left_gain, right_gain)

    dfs(root)
    return max_sum
```

### 复杂度
- 时间：`O(N)`（每个节点访问一次）
- 空间：`O(H)`（递归栈深度，H 是树高；最坏 O(N)，平衡 O(logN)）

### 易错点
- 初始值要用 `-math.inf`（或 `math.MinInt`），因为树中可能有负数节点
- `max(leftGain, rightGain, 0)` —— 如果子树贡献负数，宁可不走这一步

## 解法对比

| 维度 | 递归（推荐） |
|------|------------|
| 核心思想 | 后序遍历，每个节点计算"拐弯路径"和"向上路径" |
| 时间复杂度 | O(N) |
| 空间复杂度 | O(H) |
| 面试推荐程度 | **高**（最符合直觉） |

## 易错点

- **下标越界 / 空输入**：树非空，但需处理极端情况（全负数）
- **全局变量**：`maxSum` 必须是全局/闭包变量，单靠返回值无法拿到"不经过根"的全局最优
- **子树贡献负数**：要和 0 比，不比就会选负数分支把全局最优拉低

## 面试追问

### Q: 路径可以只包含一个节点吗？
> 可以。如果所有路径和都是负数，最大路径和就是最大的那个负数节点。代码中 `max(leftGain, rightGain, 0)` 已经处理了这个情况。

### Q: 如果要求返回最大路径和以及对应的路径节点？
> 在递归中额外记录"选择哪个分支"的信息，沿着记录回溯即可。复杂度会更高，面试中能讲清思路即可。

### Q: 如果是 N 叉树怎么做？
> 同样的思路：遍历所有子节点，计算每个子树的增益，按大小排序后选前两个（因为路径只能拐一次弯）。时间变成 O(N * M)，M 是子节点平均数。

### Q: 如果限制路径不能拐弯（只能从上往下）呢？
> 那就简单了：把递归返回值改成 `max(left, right) + root.Val`，不需要 `maxSum` 全局变量，直接返回 `max(left, right, root.Val)`。

## 一句话记忆

> 最大路径和 = 后序遍历 + 全局变量存最优 + 返回值只选一条分支（拐弯只能拐一次）。
