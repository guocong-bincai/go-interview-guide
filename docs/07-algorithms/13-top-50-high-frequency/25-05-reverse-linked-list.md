# 05. 反转链表（Reverse Linked List）

> 频率：★★★★★  难度：简单  LeetCode 206

## 题目描述
给你一个单链表的头节点 `head`，请你把链表反转，并返回反转后的头节点。

## 小白先理解题意
比如原链表是：

```text
1 -> 2 -> 3 -> nil
```

反转后要变成：

```text
3 -> 2 -> 1 -> nil
```

这题的关键不是新建一个链表，而是：
**把每个节点的 next 指针方向改过来。**

---

## 面试为什么爱问
- 链表题基础中的基础
- 能看出你是否真的理解指针
- 经常作为更多链表题的起点

## 解法一：借助数组/栈

### 思路
先把链表所有节点放进数组，再倒着重新连起来。

### 这个方法的价值
- 小白容易理解
- 能帮助你明白“反转”到底发生了什么

### Go
```go
func reverseListWithArray(head *ListNode) *ListNode {
    if head == nil {
        return nil
    }

    nodes := []*ListNode{}
    for cur := head; cur != nil; cur = cur.Next {
        nodes = append(nodes, cur)
    }

    for i := len(nodes) - 1; i > 0; i-- {
        nodes[i].Next = nodes[i-1]
    }
    nodes[0].Next = nil
    return nodes[len(nodes)-1]
}
```

### Python
```python
def reverse_list_with_array(head):
    if not head:
        return None

    nodes = []
    cur = head
    while cur:
        nodes.append(cur)
        cur = cur.next

    for i in range(len(nodes) - 1, 0, -1):
        nodes[i].next = nodes[i - 1]
    nodes[0].next = None
    return nodes[-1]
```

### 复杂度
- 时间：`O(n)`
- 空间：`O(n)`

---

## 解法二：迭代双指针（推荐）

### 核心思路
用三个变量：
- `prev`：当前节点反转后要指向谁
- `cur`：当前正在处理的节点
- `next`：先保存下一个节点，避免链断掉

每次做三件事：
1. 先保存 `cur.Next`
2. 把 `cur.Next` 指向 `prev`
3. 整体往后移动

### 例子
原来：
```text
prev=nil, cur=1 -> 2 -> 3
```

处理 1 后：
```text
nil <- 1    2 -> 3
```

再处理 2：
```text
nil <- 1 <- 2    3
```

再处理 3：
```text
nil <- 1 <- 2 <- 3
```

### Go
```go
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    cur := head

    for cur != nil {
        next := cur.Next
        cur.Next = prev
        prev = cur
        cur = next
    }

    return prev
}
```

### Python
```python
def reverse_list(head):
    prev = None
    cur = head

    while cur:
        nxt = cur.next
        cur.next = prev
        prev = cur
        cur = nxt

    return prev
```

### 复杂度
- 时间：`O(n)`
- 空间：`O(1)`

---

## 易错点
- 一定要先保存 `next`
- 最后返回的是 `prev`，不是原来的 `head`
- 空链表要处理

## 一句话记忆
**反转链表就是：保存下一个，反转当前指针，整体后移。**
