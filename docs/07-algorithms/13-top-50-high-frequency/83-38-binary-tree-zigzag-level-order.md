# 38. 二叉树锯齿形层序遍历

> 频率：★★★☆☆  难度：中等  LeetCode 103

## 小白先理解
和普通层序遍历类似，只不过：
- 第一层从左到右
- 第二层从右到左
- 第三层从左到右

一层一换方向。

---

## 面试为什么爱问
- BFS + 方向控制，是层序遍历的变种
- 考察对队列和标志位的理解
- 延伸可以问：之字形打印矩阵、螺旋遍历

## 解法一：普通 BFS 后再翻转偶数层
- 层序遍历先做出来
- 偶数层 reverse

## 解法二：BFS + 方向控制（推荐）

### 核心思路
- 普通 BFS 用队列
- 用一个 `reverse` 标志控制每层是左→右还是右→左

### Go
```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func zigzagLevelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }

    res := [][]int{}
    queue := []*TreeNode{root}
    reverse := false // false: 左→右, true: 右→左

    for len(queue) > 0 {
        size := len(queue)
        level := make([]int, size)

        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]

            // 根据方向决定填充位置
            if !reverse {
                level[i] = node.Val
            } else {
                level[size-1-i] = node.Val
            }

            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }

        res = append(res, level)
        reverse = !reverse
    }
    return res
}
```

### Python
```python
from collections import deque

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def zigzag_level_order(root):
    if not root:
        return []

    res = []
    queue = deque([root])
    reverse = False  # False: 左→右, True: 右→左

    while queue:
        size = len(queue)
        level = [0] * size

        for i in range(size):
            node = queue.popleft()

            if not reverse:
                level[i] = node.val
            else:
                level[size - 1 - i] = node.val

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        res.append(level)
        reverse = not reverse

    return res
```

## 解法三：双端队列
用双端队列，每层交替从队首/队尾写入。

## 复杂度
- 时间：`O(N)`，每个节点访问一次
- 空间：`O(W)`，W 是最大层宽度

## 一句话记忆
**锯齿层序 = 普通层序 + 每层控制方向。**