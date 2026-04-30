# 09. 二叉树最近公共祖先（Lowest Common Ancestor of a Binary Tree）

> 频率：★★★★★  难度：中等  LeetCode 236

## 题目描述
给定一个二叉树，找到该树中两个指定节点 `p` 和 `q` 的最近公共祖先。

## 小白先理解
最近公共祖先，就是：
**离 p 和 q 最近的那个“共同祖先”节点。**

比如：
- `p` 在左子树
- `q` 在右子树
那么当前根节点就是它们最近公共祖先。

---

## 解法一：记录父节点

### 思路
先遍历整棵树，记录每个节点的父亲是谁。
然后：
- 从 p 一路往上走，存到集合里
- 再从 q 一路往上走，第一个出现在集合里的就是答案

### 复杂度
- 时间：`O(n)`
- 空间：`O(n)`

---

## 解法二：递归分治（推荐）

### 核心思路
对于当前节点 root：
- 如果 root 是空，返回空
- 如果 root 就是 p 或 q，直接返回 root
- 分别去左子树和右子树找 p/q

情况分三种：
1. 左边找到了，右边也找到了 → 当前 root 就是最近公共祖先
2. 只左边找到 → 返回左边结果
3. 只右边找到 → 返回右边结果

### Go
```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q {
        return root
    }

    left := lowestCommonAncestor(root.Left, p, q)
    right := lowestCommonAncestor(root.Right, p, q)

    if left != nil && right != nil {
        return root
    }
    if left != nil {
        return left
    }
    return right
}
```

### Python
```python
def lowest_common_ancestor(root, p, q):
    if not root or root == p or root == q:
        return root

    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)

    if left and right:
        return root
    return left if left else right
```

### 复杂度
- 时间：`O(n)`
- 空间：`O(h)`，h 是树高

---

## 一句话记忆
**LCA 递归核心：左右都找到，当前就是答案。**
