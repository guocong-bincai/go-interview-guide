# 06. 环形链表（Linked List Cycle）

> 频率：★★★★★  难度：简单  LeetCode 141

## 题目描述
给你一个链表的头节点 `head`，判断链表中是否有环。

如果链表中某个节点可以通过不断往后走 다시回到自己之前见过的节点，那就说明有环。

## 小白先理解
正常链表：
```text
1 -> 2 -> 3 -> nil
```

有环链表：
```text
1 -> 2 -> 3 -> 4
     ^         |
     |_________|
```

也就是说，走着走着走不出去，会一直绕圈。

---

## 面试为什么爱问
- 快慢指针代表题
- 不仅简单高频，还能延伸到“找环入口”
- 很适合区分会不会真正理解链表技巧

## 解法一：哈希表

### 思路
边走边把访问过的节点放进集合。
如果某个节点之前见过，说明有环。

### Go
```go
func hasCycleWithMap(head *ListNode) bool {
    seen := map[*ListNode]bool{}
    for head != nil {
        if seen[head] {
            return true
        }
        seen[head] = true
        head = head.Next
    }
    return false
}
```

### Python
```python
def has_cycle_with_set(head):
    seen = set()
    while head:
        if head in seen:
            return True
        seen.add(head)
        head = head.next
    return False
```

### 复杂度
- 时间：`O(n)`
- 空间：`O(n)`

---

## 解法二：快慢指针（推荐）

### 核心思路
- 慢指针一次走 1 步
- 快指针一次走 2 步

如果没有环：
- 快指针会先走到 `nil`

如果有环：
- 快指针最终一定会追上慢指针

### 为什么一定会相遇
可以把它想成操场跑圈：
- 慢的人每次走 1 步
- 快的人每次走 2 步
- 只要一直在圈里跑，快的人迟早追上慢的人

### Go
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

### Python
```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False
```

### 复杂度
- 时间：`O(n)`
- 空间：`O(1)`

---

## 易错点
- while 条件必须写成 `fast != nil && fast.Next != nil`
- 不能只判断 `fast != nil`
- 空链表和单节点链表都可能没环

## 一句话记忆
**判断链表有没有环，优先想快慢指针。**
