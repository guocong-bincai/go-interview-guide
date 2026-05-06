# 46. 对称二叉树（Symmetric Tree）

> 频率：★★★☆☆  难度：简单  LeetCode 101

## 题目描述
给定一个二叉树，检查它是否是镜像对称的。

## 面试为什么爱问
- 这是二叉树高频题，考察"对称性"思维
- 同时练递归和迭代两种写法
- 面试官会追问：和"两棵树是否相同"的区别是什么

## 核心考点
- [ ] 二叉树对称性判断
- [ ] 递归：镜像比较左右子树
- [ ] 迭代：BFS 队列配对比较

## 小白先理解
对称，就是左子树和右子树像"照镜子"。
比如这棵树：
```
     1
    / \
   2   2
  / \ / \
 3  4 4  3
```
根的左子树是 `2→3,4`，右子树是 `2→4,3`，结构互为镜像。

**怎么判断两棵树互为镜像？**
- 两棵树根值相同
- 树 A 的左子树和树 B 的右子树互为镜像（递归）
- 树 A 的右子树和树 B 的左子树互为镜像（递归）

---

## 解法一：递归镜像比较（推荐）

### 核心思路
定义一个函数 `isMirror(a, b)` 判断两棵树是否互为镜像。

### Go
```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val   int
 *     Left  *TreeNode
 *     Right *TreeNode
 * }
 */
func isSymmetric(root *TreeNode) bool {
    if root == nil {
        return true
    }
    var isMirror func(*TreeNode, *TreeNode) bool
    isMirror = func(a, b *TreeNode) bool {
        if a == nil && b == nil {
            return true // 两个空节点，互为镜像
        }
        if a == nil || b == nil {
            return false // 一个空一个非空，不对称
        }
        // 根值相等，且左右子树互为镜像
        return a.Val == b.Val && isMirror(a.Left, b.Right) && isMirror(a.Right, b.Left)
    }
    return isMirror(root.Left, root.Right)
}
```

### Python
```python
def is_symmetric(root):
    def is_mirror(a, b):
        if not a and not b:
            return True
        if not a or not b:
            return False
        return (a.val == b.val and
                is_mirror(a.left, b.right) and
                is_mirror(a.right, b.left))
    return is_mirror(root, root)
```

### 复杂度
- 时间：O(n)，每个节点访问一次
- 空间：O(h)，递归栈深度为树高

---

## 解法二：迭代（BFS 队列配对）

### 核心思路
每次从队列取出两个节点进行比较，然后将它们的左右子节点按对称顺序入队。

### Go
```go
func isSymmetricIterative(root *TreeNode) bool {
    if root == nil {
        return true
    }
    queue := []*TreeNode{root, root}
    for len(queue) > 0 {
        // 每次取出两个节点
        left := queue[0]
        queue = queue[1:]
        right := queue[0]
        queue = queue[1:]

        if left == nil && right == nil {
            continue
        }
        if left == nil || right == nil {
            return false
        }
        if left.Val != right.Val {
            return false
        }

        // 注意顺序：左的左 = 右的右，左的右 = 右的左
        queue = append(queue, left.Left, right.Right, left.Right, right.Left)
    }
    return true
}
```

### Python
```python
def is_symmetric_iterative(root):
    from collections import deque
    queue = deque([root, root])
    while queue:
        left = queue.popleft()
        right = queue.popleft()
        if not left and not right:
            continue
        if not left or not right:
            return False
        if left.val != right.val:
            return False
        queue.extend([left.left, right.right, left.right, right.left])
    return True
```

---

## 易错点
| 错误 | 问题 |
|------|------|
| 只比较左右子树值，没递归传递 | 对称性不传递，需要递归检查子树 |
| 递归终止条件漏判 `a == nil && b == nil` | 会漏掉两侧同时为空的情况 |
| 迭代时子节点顺序放错 | 必须按「镜像顺序」入队：左左、右右、左右、右左 |

---

## 面试追问

**Q：对称二叉树和两棵树是否相等有什么区别？**
相等是「顺序一致」，对称是「左右互换」。相等要求 `left.left == right.left`，对称要求 `left.left == right.right`。

**Q：如何在判断对称的同时记录层号？**
可以在递归时传入层号参数，每层用一个 map 记录该层所有值，然后判断每层是否对称回文。

**Q：这题和"另一棵树是否是另一棵树的子树"有什么联系？**
没有直接联系。对称是同一棵树内部的左右子树互为镜像；子树是判断一棵树是否是另一棵的子结构。

---

## 一句话记忆
**对称二叉树 = 判断左右子树是否互为镜像：递归比较 left.left vs right.right 且 left.right vs right.left。**