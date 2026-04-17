# 链表高频题

> 考察频率：★★★★☆  题型识别：指针操作、环检测、虚拟头节点

## 核心技巧

### 1. 虚拟头节点（Dummy Head）
当需要操作**头节点**时，创建一个 `dummy := &ListNode{0, head}`，最终返回 `dummy.Next`。避免单独处理 head 的边界情况。

### 2. 快慢指针
- **同步指针**（同速）：找链表中点
- **不同速**：环检测用 `fast=fast.Next.Next`，`slow=slow.Next`

### 3. 反转链表
记住三指针反转模板：`prev, curr, next`

---

## 题型一：反转链表（LeetCode 206）

### 代码

```go
// 反转整个链表
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head

    for curr != nil {
        next := curr.Next // 1. 先保存下一个
        curr.Next = prev  // 2. 反转指向
        prev = curr       // 3. prev 前移
        curr = next       // 4. curr 前移
    }
    return prev
}
```

### 追问：反转前 N 个节点

```go
func reverseN(head *ListNode, n int) *ListNode {
    var successor *ListNode // 第 n+1 个节点

    var dfs func(head *ListNode, n int) *ListNode
    dfs = func(head *ListNode, n int) *ListNode {
        if n == 1 {
            successor = head.Next
            return head
        }
        last := dfs(head.Next, n-1)
        head.Next.Next = head
        head.Next = successor
        return last
    }

    return dfs(head, n)
}
```

---

## 题型二：环形链表（LeetCode 141/142）

### 141 - 环形检测（返回 bool）

**快慢指针**：若存在环，fast 和 slow 必定相遇。

```go
func hasCycle(head *ListNode) bool {
    if head == nil || head.Next == nil {
        return false
    }
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            return true
        }
    }
    return false
}
```

### 142 - 环形链表 II（返回环入口节点）

**数学推导**：
- 设环前长度为 a，环入口到相遇点为 b，环剩余为 c
- slow 走：`a + b`，fast 走：`a + b + c + b = a + 2b + c`
- `2(a+b) = a+2b+c` → `a = c`
- 所以从 head 和相遇点同步走，相遇点就是环入口

```go
func detectCycle(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return nil
    }
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            // 有环，找入口
            p1, p2 := head, slow
            for p1 != p2 {
                p1 = p1.Next
                p2 = p2.Next
            }
            return p1
        }
    }
    return nil
}
```

---

## 题型三：合并两个有序链表（LeetCode 21）

```go
func mergeTwoLists(l1, l2 *ListNode) *ListNode {
    dummy := &ListNode{0, nil}
    curr := dummy

    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            curr.Next = l1
            l1 = l1.Next
        } else {
            curr.Next = l2
            l2 = l2.Next
        }
        curr = curr.Next
    }

    if l1 != nil {
        curr.Next = l1
    } else {
        curr.Next = l2
    }
    return dummy.Next
}
```

---

## 题型四：K 个一组翻转链表（LeetCode 25）

### 题目
每 k 个节点反转一次，不足 k 个不反转。

### 思路
分两步：1) 统计长度判断是否够 k 个；2) 反转 k 个节点；3) 递归处理后续。

```go
func reverseKGroup(head *ListNode, k int) *ListNode {
    // 统计够不够 k 个
    cnt := 0
    for cur := head; cur != nil; cur = cur.Next {
        cnt++
        if cnt >= k {
            break
        }
    }
    if cnt < k {
        return head
    }

    // 反转前 k 个
    var prev, curr, next *ListNode
    curr = head
    for i := 0; i < k; i++ {
        next = curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }

    // 此时 prev 是第 k 个节点（反转后的头），curr 是第 k+1 个
    // head 现在是第 k 个节点的 next，接到递归结果上
    head.Next = reverseKGroup(curr, k)

    return prev
}
```

---

## 题型五：LRU 缓存（LeetCode 146）

### 题目
实现 LRU 缓存，支持 O(1) 的 get 和 put。

### 数据结构
**哈希表 + 双向链表**：
- 哈希表：O(1) 查找
- 双向链表：O(1) 移动和删除节点

### Go 实现

```go
type LRUCache struct {
    capacity int
    cache    map[int]*Node
    // 虚拟头尾，避免判空
    head, tail *Node
}

type Node struct {
    key, val   int
    prev, next *Node
}

func Constructor(capacity int) LRUCache {
    head := &Node{}
    tail := &Node{}
    head.next = tail
    tail.prev = head
    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*Node),
        head:     head,
        tail:     tail,
    }
}

func (l *LRUCache) Get(key int) int {
    node, ok := l.cache[key]
    if !ok {
        return -1
    }
    // 移到尾部（最近使用）
    l.moveToTail(node)
    return node.val
}

func (l *LRUCache) Put(key int, value int) {
    if node, ok := l.cache[key]; ok {
        node.val = value
        l.moveToTail(node)
        return
    }
    // 新建节点
    node := &Node{key: key, val: value}
    l.cache[key] = node
    l.addToTail(node)

    if len(l.cache) > l.capacity {
        // 删除最久未使用的（头部的下一个）
        removed := l.head.next
        l.removeNode(removed)
        delete(l.cache, removed.key)
    }
}

func (l *LRUCache) moveToTail(node *Node) {
    l.removeNode(node)
    l.addToTail(node)
}

func (l *LRUCache) removeNode(node *Node) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (l *LRUCache) addToTail(node *Node) {
    node.prev = l.tail.prev
    node.next = l.tail
    l.tail.prev.next = node
    l.tail.prev = node
}
```

### 追问
- **LFU 缓存**（LeetCode 460）：用两个哈希表 + 双向链表，按访问频率维护。

---

## 题型六：删除链表的倒数第 N个节点（LeetCode 19）

### 思路：一次遍历
先让快指针走 n 步，然后快慢指针同步走。当快指针到末尾时，慢指针正好在倒数第 n 个节点前一个。

```go
func removeNthFromEnd(head *ListNode, n int) *ListNode {
    dummy := &ListNode{0, head}
    fast, slow := dummy, dummy

    // fast 先走 n+1 步
    for i := 0; i <= n; i++ {
        fast = fast.Next
    }

    // 同步走，直到 fast 到末尾
    for fast != nil {
        slow = slow.Next
        fast = fast.Next
    }

    // slow.Next 现在是倒数第 n 个
    slow.Next = slow.Next.Next
    return dummy.Next
}
```

---

## 题型七：相交链表（LeetCode 160）

### 思路
A 走完自己再走 B，B 走完自己再走 A，若相交必相遇。

```go
func getIntersectionNode(headA, headB *ListNode) *ListNode {
    if headA == nil || headB == nil {
        return nil
    }
    p1, p2 := headA, headB
    for p1 != p2 {
        if p1 == nil {
            p1 = headB
        } else {
            p1 = p1.Next
        }
        if p2 == nil {
            p2 = headA
        } else {
            p2 = p2.Next
        }
    }
    return p1
}
```

---

## 链表技巧总结

| 技巧 | 适用场景 |
|------|---------|
| 虚拟头节点 | 需要操作 head，或删除节点 |
| 快慢指针 | 找中点、环检测、删除倒数第 N 个 |
| 反转三指针 | 反转链表、反转前 N 个 |
| 哨兵节点 | 合并链表、批量删除 |
| 双指针遍历 | 相交链表 |

### 链表题常见坑
1. **空指针**：遍历前检查 `head != nil`
2. **环**：用快慢指针验证
3. **边界**：head 本身、只有一个节点
4. **画图**：链表题一定要画图理解指针移动
5. **循环不变**：每次操作前理解"当前状态"

---

## Python 实现汇总

> Python 没有指针概念，用对象引用操作，逻辑相同。

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
```

### 反转链表

```python
def reverseList(head: ListNode) -> ListNode:
    prev, cur = None, head
    while cur:
        nxt = cur.next
        cur.next = prev
        prev = cur
        cur = nxt
    return prev
```

### 环形链表检测

```python
def hasCycle(head: ListNode) -> bool:
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False
```

### 合并两个有序链表

```python
def mergeTwoLists(l1: ListNode, l2: ListNode) -> ListNode:
    dummy = ListNode(0)
    cur = dummy
    while l1 and l2:
        if l1.val <= l2.val:
            cur.next = l1
            l1 = l1.next
        else:
            cur.next = l2
            l2 = l2.next
        cur = cur.next
    cur.next = l1 or l2
    return dummy.next
```

### 删除倒数第 N 个节点

```python
def removeNthFromEnd(head: ListNode, n: int) -> ListNode:
    dummy = ListNode(0, head)
    fast = slow = dummy
    for _ in range(n + 1):
        fast = fast.next
    while fast:
        slow = slow.next
        fast = fast.next
    slow.next = slow.next.next
    return dummy.next
```

### LRU 缓存

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.cache = OrderedDict()  # 有序字典，记录使用顺序

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)  # 标记为最近使用
        return self.cache[key]

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.cap:
            self.cache.popitem(last=False)  # 移除最久未使用
```
