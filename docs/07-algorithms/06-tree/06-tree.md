# 树高频题

> 考察频率：★★★★☆  题型识别：树结构遍历、递归思想

## 树节点定义

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}
```

---

## 遍历模板

### 前序遍历（根左右）
```go
func preorder(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    var result []int
    result = append(result, root.Val)
    result = append(result, preorder(root.Left)...)
    result = append(result, preorder(root.Right)...)
    return result
}
```

### 中序遍历（左根右）
```go
func inorder(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    var result []int
    result = append(result, inorder(root.Left)...)
    result = append(result, root.Val)
    result = append(result, inorder(root.Right)...)
    return result
}
```

### 后序遍历（左右根）
```go
func postorder(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    var result []int
    result = append(result, postorder(root.Left)...)
    result = append(result, postorder(root.Right)...)
    result = append(result, root.Val)
    return result
}
```

### 迭代版遍历（栈模拟）

```go
// 前序迭代
func preorderIter(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    var result []int
    stack := []*TreeNode{root}
    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, node.Val)
        if node.Right != nil {
            stack = append(stack, node.Right)
        }
        if node.Left != nil {
            stack = append(stack, node.Left)
        }
    }
    return result
}

// 中序迭代（顺着左子树一直入栈）
func inorderIter(root *TreeNode) []int {
    var result []int
    stack := []*TreeNode{}
    cur := root

    for cur != nil || len(stack) > 0 {
        for cur != nil {
            stack = append(stack, cur)
            cur = cur.Left
        }
        cur = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, cur.Val)
        cur = cur.Right
    }
    return result
}
```

---

## 题型一：二叉树最大深度（LeetCode 104）

### 代码（递归）

```go
func maxDepth(root *TreeNode) int {
    if root == nil {
        return 0
    }
    left := maxDepth(root.Left)
    right := maxDepth(root.Right)
    return max(left, right) + 1
}
```

### BFS 层序遍历

```go
func maxDepthBFS(root *TreeNode) int {
    if root == nil {
        return 0
    }
    depth := 0
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        depth++
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[i]
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        queue = queue[size:]
    }
    return depth
}
```

---

## 题型二：二叉树最近公共祖先（LeetCode 236）

### 题目
给定二叉树和两个节点 p、q，找出它们的最近公共祖先。

### 思路
后序遍历（左右根），利用递归返回值：
- 若当前节点是 p 或 q，返回当前节点
- 若左右子树各返回一个节点，当前节点就是 LCA
- 若只返回一边，说明 p、q 都在那一边
- 若返回 nil，说明都不在

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q {
        return root
    }

    left := lowestCommonAncestor(root.Left, p, q)
    right := lowestCommonAncestor(root.Right, p, q)

    if left != nil && right != nil {
        return root // p、q 分别在左右子树
    }
    if left != nil {
        return left // 都在左子树
    }
    if right != nil {
        return right // 都在右子树
    }
    return nil
}
```

### 追问：二叉搜索树的 LCA（LeetCode 235）
BST 有序，直接比较 val 即可：

```go
func lowestCommonAncestorBST(root, p, q *TreeNode) *TreeNode {
    for (root.Val-p.Val)*(root.Val-q.Val) > 0 {
        if root.Val > p.Val {
            root = root.Left
        } else {
            root = root.Right
        }
    }
    return root
}
```

---

## 题型三：层序遍历 / 二叉树右视图（LeetCode 199）

### 层序遍历（BFS）

```go
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    var result [][]int
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        size := len(queue)
        var level []int
        for i := 0; i < size; i++ {
            node := queue[i]
            level = append(level, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        result = append(result, level)
        queue = queue[size:]
    }
    return result
}
```

### 右视图：每层最后一个节点

```go
func rightSideView(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    var result []int
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[i]
            if i == size-1 {
                result = append(result, node.Val)
            }
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        queue = queue[size:]
    }
    return result
}
```

---

## 题型四：路径总和（LeetCode 112/113）

### 112：是否存在从根到叶子节点的路径，和等于 targetSum

```go
func hasPathSum(root *TreeNode, targetSum int) bool {
    if root == nil {
        return false
    }
    if root.Left == nil && root.Right == nil {
        return root.Val == targetSum
    }
    return hasPathSum(root.Left, targetSum-root.Val) ||
           hasPathSum(root.Right, targetSum-root.Val)
}
```

### 113：返回所有路径

```go
func pathSum(root *TreeNode, targetSum int) [][]int {
    var result [][]int
    var path []int

    var dfs func(node *TreeNode, sum int)
    dfs = func(node *TreeNode, sum int) {
        if node == nil {
            return
        }
        path = append(path, node.Val)
        sum -= node.Val

        if node.Left == nil && node.Right == nil && sum == 0 {
            tmp := make([]int, len(path))
            copy(tmp, path)
            result = append(result, tmp)
        }
        dfs(node.Left, sum)
        dfs(node.Right, sum)
        path = path[:len(path)-1] // 回溯
    }

    dfs(root, targetSum)
    return result
}
```

---

## 题型五：重建二叉树（LeetCode 105）

### 题目
前序 + 中序重建二叉树。

### 思路
前序第一个是根，在中序中定位根，左边是左子树，右边是右子树，递归构建。

```go
func buildTree(preorder []int, inorder []int) *TreeNode {
    if len(preorder) == 0 {
        return nil
    }

    rootVal := preorder[0]
    root := &TreeNode{Val: rootVal}

    // 在中序中找根
    idx := 0
    for i, v := range inorder {
        if v == rootVal {
            idx = i
            break
        }
    }

    root.Left = buildTree(preorder[1:idx+1], inorder[:idx])
    root.Right = buildTree(preorder[idx+1:], inorder[idx+1:])
    return root
}
```

### 追问：迭代法 O(n)？
→ 利用前序遍历的特性，用栈模拟，略复杂，了解即可。

---

## 题型六：验证二叉搜索树（LeetCode 98）

### 思路：中序遍历必须升序

```go
func isValidBST(root *TreeNode) bool {
    var prev *TreeNode // 记录上一个访问的节点

    var validate func(node *TreeNode) bool
    validate = func(node *TreeNode) bool {
        if node == nil {
            return true
        }
        if !validate(node.Left) {
            return false
        }
        if prev != nil && node.Val <= prev.Val {
            return false
        }
        prev = node
        return validate(node.Right)
    }

    return validate(root)
}
```

---

## 题型七：二叉树展开为链表（LeetCode 114）

### 题目
将二叉树原地展开为"右子树链表"（先序遍历顺序）。

### 思路
使用后序遍历（右左根的顺序）收集节点，然后串起来。

```go
func flatten(root *TreeNode) {
    if root == nil {
        return
    }

    var prev *TreeNode

    var dfs func(node *TreeNode)
    dfs = func(node *TreeNode) {
        if node == nil {
            return
        }
        dfs(node.Right)
        dfs(node.Left)
        node.Right = prev
        node.Left = nil
        prev = node
    }

    dfs(root)
}
```

---

## 树形DP模板

树形 DP 核心：后序遍历 + 合并子问题结果。

```go
func dfs(node *TreeNode) [2]int {
    // [不选当前节点的收益, 选当前节点的收益]
    if node == nil {
        return [2]int{0, 0}
    }
    left := dfs(node.Left)
    right := dfs(node.Right)

    notChoose := max(left[0], left[1]) + max(right[0], right[1])
    choose := left[0] + right[0] + node.Val

    return [2]int{notChoose, choose}
}
```

---

## 总结

| 题型 | 关键方法 | 时间 |
|------|---------|------|
| 遍历（递归/迭代） | 前/中/后序，栈模拟 | O(n) |
| 层序遍历 | BFS + queue | O(n) |
| 最大深度 | 递归/BFS | O(n) |
| LCA（普通二叉树） | 后序遍历返回值 | O(n) |
| LCA（BST） | 直接比较 val | O(n) |
| 路径总和 | 递归/DFS | O(n) |
| 验证 BST | 中序遍历升序 | O(n) |
| 重建二叉树 | 前+中序递归 | O(n) |
| 展开链表 | 后序遍历（右键左） | O(n) |
