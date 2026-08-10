# 42. K 个一组翻转链表（Reverse Nodes in k-Group）

> 频率：★★★★☆  难度：困难  LeetCode 25

## 小白先理解
把链表每 k 个节点分成一组，每组内部反转。
如果最后剩下不足 k 个，就保持原样。

---

## 面试为什么爱问
- 这是 **链表操作 + 区间翻转** 的综合题
- 考察指针操作的精细度（容易出错）
- 同时考察：如何先判断再行动（先计数再翻转）

## 解法一：借助数组分组反转
- 把节点收集到数组，再按组翻转
- 好理解，但用了额外空间

## 解法二：链表原地反转（推荐）

### 核心思路
1. 先用 dummy 节点简化头节点操作
2. 每次先判断剩余节点数是否够 k 个（不够直接结束）
3. 用两个指针：prev 指向待翻转区间前一个节点，start 指向区间第一个节点
4. 翻转区间后重新接上

### Go
```go
type ListNode struct {
    Val  int
    Next *ListNode
}

func reverseKGroup(head *ListNode, k int) *ListNode {
    dummy := &ListNode{Next: head}
    prev := dummy

    for {
        // 1. 检查剩余节点数是否够 k 个
        count := 0
        curr := prev
        for curr != nil && count < k {
            curr = curr.Next
            count++
        }
        if count < k {
            break // 剩余不够，直接结束
        }

        // 2. 开始翻转：start 是待翻转区间第一个节点
        start := prev.Next
        curr = start
        var prevNode *ListNode

        // 3. 翻转 k 个节点
        for i := 0; i < k; i++ {
            next := curr.Next
            curr.Next = prevNode
            prevNode = curr
            curr = next
        }

        // 4. 翻转完成后重新连接
        // 此时 prevNode 是翻转后的新头（即原区间最后一个节点）
        // start 变成翻转后的尾，curr 指向下一组开头
        prev.Next = prevNode // prev 指向翻转后新头
        start.Next = curr    // 原头（现在在翻转后位置）指向下一组
        prev = start         // prev 移动到下一组的前一个位置
    }

    return dummy.Next
}
```

### Python
```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def reverse_k_group(head, k):
    dummy = ListNode(0, head)
    prev = dummy

    while True:
        # 检查剩余节点数是否够 k 个
        count = 0
        curr = prev
        while curr and count < k:
            curr = curr.next
            count += 1
        if count < k:
            break

        # 开始翻转
        start = prev.next
        curr = start
        prev_node = None

        for _ in range(k):
            next_node = curr.next
            curr.next = prev_node
            prev_node = curr
            curr = next_node

        # 重新连接
        prev.next = prev_node
        start.next = curr
        prev = start

    return dummy.next
```

## 复杂度
- 时间：`O(N)`，每个节点最多访问常数次
- 空间：`O(1)`，原地翻转

## 一句话记忆
**K 个一组翻转 = 分组 + 局部反转 + 接回去。**