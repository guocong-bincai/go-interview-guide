# 34. 合并 K 个升序链表（Merge K Sorted Lists）

> 频率：★★★★☆  难度：困难  LeetCode 23

## 小白先理解
两个有序链表你会合并，K 个其实也一样，只是数量更多。
关键问题是：**每次怎么快速找到当前最小节点。**

想象 K 路归并的磁带，每路都有一个当前读头，要找到所有读头中最小的那个往下走。

---

## 面试为什么爱问
- 这是 **K 路归并** 的经典问题
- 同时考：最小堆（优先队列）+ 多路归并思维
- 能延伸到外部排序、合并 K 个有序文件

## 解法一：两两合并
- 不断把两个链表合并成一个
- 时间 `O(KN)`，K 个链表，每个 N 个节点

## 解法二：最小堆（推荐）

### 核心思路
把每个链表当前头节点放进最小堆。
每次弹出堆顶（全局最小），把它接到结果链表中，再把该链表的下一个节点入堆。

### 为什么最小堆快
- 原来需要 `O(K)` 比较找最小
- 堆只需要 `O(logK)`

### Go
```go
type ListNode struct {
    Val  int
    Next *ListNode
}

func mergeKLists(lists []*ListNode) *ListNode {
    if len(lists) == 0 {
        return nil
    }

    // 最小堆：按 Val 从小到大排
    hp := &Heap{}
    heap.Init(hp)

    // 所有链表的头节点入堆
    for _, list := range lists {
        if list != nil {
            heap.Push(hp, list)
        }
    }

    dummy := &ListNode{}
    cur := dummy

    for hp.Len() > 0 {
        // 弹出最小节点
        node := heap.Pop(hp).(*ListNode)
        cur.Next = node
        cur = cur.Next

        // 如果该链表还有下一个，继续入堆
        if node.Next != nil {
            heap.Push(hp, node.Next)
        }
    }
    return dummy.Next
}

// Heap 实现（Go 内置 container/heap）
type Heap []*ListNode

func (h Heap) Len() int            { return len(h) }
func (h Heap) Less(i, j int) bool { return h[i].Val < h[j].Val }
func (h Heap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *Heap) Push(x any) {
    *h = append(*h, x.(*ListNode))
}

func (h *Heap) Pop() any {
    old := *h
    n := len(old)
    node := old[n-1]
    *h = old[:n-1]
    return node
}
```

### Python
```python
import heapq

class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def mergeKLists(lists):
    heap = []
    for i, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, i, node))

    dummy = ListNode()
    cur = dummy
    while heap:
        val, i, node = heapq.heappop(heap)
        cur.next = node
        cur = cur.next
        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))

    return dummy.next
```

## 解法三：分治归并
- 每次合并两个链表，类比归并排序
- 时间 `O(N logK)`

### Go
```go
func mergeKLists(lists []*ListNode) *ListNode {
    if len(lists) == 0 {
        return nil
    }
    var mergeTwo func(*ListNode, *ListNode) *ListNode
    mergeTwo = func(l1, l2 *ListNode) *ListNode {
        if l1 == nil {
            return l2
        }
        if l2 == nil {
            return l1
        }
        if l1.Val <= l2.Val {
            l1.Next = mergeTwo(l1.Next, l2)
            return l1
        } else {
            l2.Next = mergeTwo(l1, l2.Next)
            return l2
        }
    }

    for len(lists) > 1 {
        merged := []int{}
        for i := 0; i < len(lists); i += 2 {
            l1 := lists[i]
            l2 := &ListNode{}
            if i+1 < len(lists) {
                l2 = lists[i+1]
            }
            merged = append(merged, mergeTwo(l1, l2).Val)
        }
    }
    return lists[0]
}
```

## 复杂度对比

| 解法 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| 两两合并 | O(KN) | O(1) |
| 最小堆 | O(N logK) | O(K) |
| 分治归并 | O(N logK) | O(logK) |

## 一句话记忆
**K 路归并问题，优先想最小堆。**