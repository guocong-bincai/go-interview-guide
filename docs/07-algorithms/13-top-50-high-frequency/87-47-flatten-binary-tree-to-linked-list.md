# 47. 二叉树展开为链表（Flatten Binary Tree to Linked List）

> 频率：★★★☆☆  难度：中等  LeetCode 114

## 小白先理解
把二叉树原地展开成链表，顺序要和前序遍历一样：
根 → 左 → 右

---

## 面试为什么爱问
- 这道题考的是 **前序遍历 + 链表重连** 的综合能力
- 原地操作，不允许用额外数组存遍历结果
- 考验对递归和指针操作的深度理解

## 解法一：先前序遍历存下来再重连
- 先前序遍历把节点顺序存到数组
- 再遍历数组逐个重连
- 好理解，但 `O(N)` 额外空间

## 解法二：递归原地展开（推荐）

### 核心思路
把左子树先展开成链表，再把右子树接到左子树链表的尾部。

```
展开前：          展开后：
    1            1
   / \            \
  2   5    →       2
   \    \           \
    3    6           3
     \               \
      4               4
                       \
                        5
                         \
                          6
```

关键发现：**前序遍历的顺序是 1→2→3→4→5→6，展开后每个节点的右指针指向下一个节点。**

### Go
```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func flatten(root *TreeNode) {
    if root == nil {
        return
    }

    // 先把左右子树展开
    flatten(root.Left)
    flatten(root.Right)

    // 此时左右子树都已经是链表形式
    // 保存原来的右子树（展开后会成为左子树链表的尾部）
    right := root.Right

    // 左子树链表接到右边
    root.Right = root.Left
    root.Left = nil

    // 找到左子树链表的最后一个节点
    curr := root
    for curr.Right != nil {
        curr = curr.Right
    }

    // 把原来的右子树接到左子树链表尾部
    curr.Right = right
}
```

### Python
```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def flatten(root):
    if not root:
        return

    # 先递归展开左右子树
    flatten(root.left)
    flatten(root.right)

    # 保存原来的右子树
    right = root.right

    # 左子树链表接到右边
    root.right = root.left
    root.left = None

    # 找到左子树链表尾部
    curr = root
    while curr.right:
        curr = curr.right

    # 接上原来的右子树
    curr.right = right
```

## 解法三：前序遍历 + 栈（显式栈）
```go
func flatten(root *TreeNode) {
    if root == nil {
        return
    }

    stack := []*TreeNode{}
    prev := &TreeNode{}

    // 前序遍历：根→左→右
    curr := root
    for curr != nil || len(stack) > 0 {
        // 先遍历左子树
        for curr != nil {
            stack = append(stack, curr)
            prev.Right = curr      // 前一个节点的右指针指向当前
            prev.Left = nil        // 左指针置空
            prev = curr
            curr = curr.Left
        }

        // 弹出栈顶，开始右子树
        if len(stack) > 0 {
            curr = stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            curr = curr.Right
        }
    }
}
```

## 复杂度对比

| 解法 | 时间 | 空间 |
|------|------|------|
| 前序遍历存数组 | O(N) | O(N) |
| 递归原地展开 | O(N) | O(H)，递归栈 |
| 前序+栈 | O(N) | O(H) |

## 一句话记忆
**展开二叉树 = 前序遍历顺序重连指针。**