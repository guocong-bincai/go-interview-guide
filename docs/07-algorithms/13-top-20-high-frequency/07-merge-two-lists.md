# 07. 合并两个有序链表（Merge Two Sorted Lists）

> 频率：★★★★★  难度：简单  LeetCode 21

## 题目描述
将两个升序链表合并为一个新的升序链表，并返回合并后的链表头节点。

## 小白先理解
比如：

```text
l1 = 1 -> 2 -> 4
l2 = 1 -> 3 -> 4
```

合并后：

```text
1 -> 1 -> 2 -> 3 -> 4 -> 4
```

你可以把它想成：
**每次从两个链表头部，挑一个更小的接到结果后面。**

---

## 解法一：递归

### 思路
每次比较两个头节点：
- 谁小，谁就做结果头
- 然后把剩余部分继续递归合并

### Go
```go
func mergeTwoListsRecursive(list1 *ListNode, list2 *ListNode) *ListNode {
    if list1 == nil {
        return list2
    }
    if list2 == nil {
        return list1
    }

    if list1.Val < list2.Val {
        list1.Next = mergeTwoListsRecursive(list1.Next, list2)
        return list1
    }
    list2.Next = mergeTwoListsRecursive(list1, list2.Next)
    return list2
}
```

### Python
```python
def merge_two_lists_recursive(list1, list2):
    if not list1:
        return list2
    if not list2:
        return list1

    if list1.val < list2.val:
        list1.next = merge_two_lists_recursive(list1.next, list2)
        return list1
    else:
        list2.next = merge_two_lists_recursive(list1, list2.next)
        return list2
```

### 复杂度
- 时间：`O(m+n)`
- 空间：`O(m+n)`（递归栈）

---

## 解法二：迭代 + 虚拟头节点（推荐）

### 核心思路
创建一个 `dummy` 虚拟头节点，这样不用反复处理“第一个节点怎么接”的特殊情况。

然后：
- 比较两个链表当前节点
- 把较小的接到结果链表末尾
- 对应链表往后走一步
- 最后把剩下的链表直接接上

### Go
```go
func mergeTwoLists(list1 *ListNode, list2 *ListNode) *ListNode {
    dummy := &ListNode{}
    cur := dummy

    for list1 != nil && list2 != nil {
        if list1.Val < list2.Val {
            cur.Next = list1
            list1 = list1.Next
        } else {
            cur.Next = list2
            list2 = list2.Next
        }
        cur = cur.Next
    }

    if list1 != nil {
        cur.Next = list1
    }
    if list2 != nil {
        cur.Next = list2
    }

    return dummy.Next
}
```

### Python
```python
def merge_two_lists(list1, list2):
    dummy = ListNode(0)
    cur = dummy

    while list1 and list2:
        if list1.val < list2.val:
            cur.next = list1
            list1 = list1.next
        else:
            cur.next = list2
            list2 = list2.next
        cur = cur.next

    cur.next = list1 if list1 else list2
    return dummy.next
```

### 复杂度
- 时间：`O(m+n)`
- 空间：`O(1)`

---

## 一句话记忆
**合并有序链表 = 每次挑更小的接上，dummy 节点能省很多事。**
