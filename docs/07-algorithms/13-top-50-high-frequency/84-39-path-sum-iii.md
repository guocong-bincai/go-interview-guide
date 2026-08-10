# 39. 路径总和 III（Path Sum III）

> 频率：★★★☆☆  难度：中等  LeetCode 437

## 小白先理解
问的是：二叉树里有多少条"向下走"的路径，它们的和等于 targetSum。
不要求从根开始，也不要求到叶子结束。

---

## 面试为什么爱问
- 这是 **前缀和思想在树上的应用**
- 比普通路径和多一个维度：起点不固定
- 经典套路：前缀和 + 哈希表记录路径和出现次数

## 解法一：枚举每个节点作为起点
- 对每个节点向下 DFS
- 时间 `O(N²)`，极端情况是每个节点作为起点都要遍历整棵树

## 解法二：前缀和 + DFS（推荐）

### 核心思路
和数组前缀和很像：
- 从根到当前节点的路径和称为 `currSum`
- 如果 `currSum - targetSum` 在之前的路径和中出现过，说明存在一条从某个祖先到当前节点的路径和为 target
- 用哈希表记录：从根到当前节点的路径中，"所有可能前缀和"出现的次数

### 为什么要用哈希表记录次数
因为从根到某节点的路径上，可能有多个节点到当前节点的和等于 targetSum，所以要计数。

### Go
```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func pathSum(root *TreeNode, targetSum int) int {
    // 前缀和 -> 出现次数
    prefix := map[int]int{0: 1}

    var dfs func(*TreeNode, int) int
    dfs = func(node *TreeNode, currSum int) int {
        if node == nil {
            return 0
        }

        currSum += node.Val
        // 看看有多少个起点到当前节点路径和 = targetSum
        // 即：currSum - targetSum 在路径中出现过多少次
        count := prefix[currSum-targetSum]

        // 把当前节点的前缀和加入记录
        prefix[currSum]++

        // 继续往下走
        count += dfs(node.Left, currSum)
        count += dfs(node.Right, currSum)

        // 回溯：离开当前节点，要从路径记录中移除
        prefix[currSum]--

        return count
    }

    return dfs(root, 0)
}
```

### Python
```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def path_sum(root, target_sum):
    prefix = {0: 1}

    def dfs(node, curr_sum):
        if not node:
            return 0

        curr_sum += node.val
        count = prefix.get(curr_sum - target_sum, 0)

        prefix[curr_sum] = prefix.get(curr_sum, 0) + 1

        count += dfs(node.left, curr_sum)
        count += dfs(node.right, curr_sum)

        prefix[curr_sum] -= 1

        return count

    return dfs(root, 0)
```

## 复杂度
- 时间：`O(N)`，每个节点访问一次
- 空间：`O(H)`，H 是树高（递归栈 + 哈希表）

## 一句话记忆
**路径总和 III = 树上的前缀和。**