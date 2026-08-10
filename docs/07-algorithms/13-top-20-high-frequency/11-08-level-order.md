# 08. 二叉树层序遍历（Binary Tree Level Order Traversal）

> 频率：★★★★★  难度：中等  LeetCode 102

## 题目描述
给你二叉树的根节点 `root`，返回其节点值的 **层序遍历**。

也就是：
- 第一层从左到右
- 第二层从左到右
- 第三层从左到右

## 小白先理解
比如这棵树：

```text
    3
   / \
  9  20
    /  \
   15   7
```

层序遍历结果是：

```text
[[3], [9,20], [15,7]]
```

它其实就是：
**一层一层往下扫。**

---

## 解法一：DFS 按层记录

### 思路
虽然层序遍历最常见是 BFS，但 DFS 也可以做。
递归时记录当前层数，把当前节点放到对应层的数组里。

### Go
```go
func levelOrderDFS(root *TreeNode) [][]int {
    var res [][]int
    var dfs func(node *TreeNode, depth int)
    dfs = func(node *TreeNode, depth int) {
        if node == nil {
            return
        }
        if depth == len(res) {
            res = append(res, []int{})
        }
        res[depth] = append(res[depth], node.Val)
        dfs(node.Left, depth+1)
        dfs(node.Right, depth+1)
    }
    dfs(root, 0)
    return res
}
```

### Python
```python
def level_order_dfs(root):
    res = []

    def dfs(node, depth):
        if not node:
            return
        if depth == len(res):
            res.append([])
        res[depth].append(node.val)
        dfs(node.left, depth + 1)
        dfs(node.right, depth + 1)

    dfs(root, 0)
    return res
```

---

## 解法二：队列 BFS（推荐）

### 核心思路
BFS 最适合“按层遍历”。
用队列保存当前层节点：
- 先记录当前队列长度 = 当前层节点数
- 这一轮只弹出这么多个
- 同时把下一层孩子节点加入队列

### Go
```go
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return [][]int{}
    }

    res := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        size := len(queue)
        level := []int{}
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        res = append(res, level)
    }

    return res
}
```

### Python
```python
from collections import deque

def level_order(root):
    if not root:
        return []

    res = []
    queue = deque([root])

    while queue:
        size = len(queue)
        level = []
        for _ in range(size):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        res.append(level)

    return res
```

### 复杂度
- 时间：`O(n)`
- 空间：`O(n)`

---

## 一句话记忆
**按层遍历二叉树，优先想 BFS 队列。**
