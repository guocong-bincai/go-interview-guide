# 46. 对称二叉树（Symmetric Tree）

> 频率：★★★☆☆  难度：简单  LeetCode 101

## 小白先理解
一棵树如果左右两边像照镜子一样，就是对称的。

## 解法一：BFS 成对检查
- 队列里每次取两个节点比较

## 解法二：递归镜像比较（推荐）
### 核心思路
判断两棵树是否互为镜像：
- 根值相同
- 左的左 = 右的右
- 左的右 = 右的左

### Go
```go
func isSymmetric(root *TreeNode) bool {
    var check func(*TreeNode, *TreeNode) bool
    check = func(a, b *TreeNode) bool {
        if a == nil && b == nil { return true }
        if a == nil || b == nil || a.Val != b.Val { return false }
        return check(a.Left, b.Right) && check(a.Right, b.Left)
    }
    return check(root, root)
}
```
### Python
```python
def is_symmetric(root):
    def check(a, b):
        if not a and not b:
            return True
        if not a or not b or a.val != b.val:
            return False
        return check(a.left, b.right) and check(a.right, b.left)
    return check(root, root)
```

## 一句话记忆
**对称二叉树 = 判断左右子树是否互为镜像。**
