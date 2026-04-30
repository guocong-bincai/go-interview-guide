# 04. LRU 缓存（LRU Cache）

> 频率：★★★★★  难度：中等  LeetCode 146

## 题目描述
设计一个满足 LRU（最近最少使用）缓存机制的数据结构。

它需要支持：
- `get(key)`：如果 key 存在，返回 value，否则返回 -1
- `put(key, value)`：插入或更新 key
- 当容量满时，删除 **最久没用过** 的那个 key

## 小白先理解
LRU 的意思是：
**谁最久没被用到，谁就先被淘汰。**

比如容量是 2：
- 放入 `(1,1)`
- 放入 `(2,2)`
- 访问 `1`
- 再放入 `(3,3)`

这时候最久没用的是 `2`，所以删除 `2`。

---

## 面试为什么爱问
- 这是数据结构设计题代表题
- 会同时考：哈希表 + 双向链表
- 能看出你是否理解“查找快 + 更新顺序快”

## 解法一：数组/列表模拟

### 思路
用列表记录访问顺序。
每次访问一个元素，就把它移动到“最新”位置。

### 问题
- 查找慢
- 删除中间元素也慢

### 复杂度
- 时间：最坏 `O(n)`
- 空间：`O(n)`

---

## 解法二：哈希表 + 双向链表（标准答案）

### 核心思路
- 哈希表：根据 key 快速找到节点，`O(1)`
- 双向链表：快速删除、插入节点，维护“最近使用顺序”
- 链表头部：最近使用
- 链表尾部：最久未使用

### Go
```go
type Node struct {
    key, value int
    prev, next *Node
}

type LRUCache struct {
    capacity int
    cache    map[int]*Node
    head     *Node
    tail     *Node
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

func (l *LRUCache) remove(node *Node) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (l *LRUCache) addToHead(node *Node) {
    node.next = l.head.next
    node.prev = l.head
    l.head.next.prev = node
    l.head.next = node
}

func (l *LRUCache) moveToHead(node *Node) {
    l.remove(node)
    l.addToHead(node)
}

func (l *LRUCache) removeTail() *Node {
    node := l.tail.prev
    l.remove(node)
    return node
}

func (l *LRUCache) Get(key int) int {
    if node, ok := l.cache[key]; ok {
        l.moveToHead(node)
        return node.value
    }
    return -1
}

func (l *LRUCache) Put(key int, value int) {
    if node, ok := l.cache[key]; ok {
        node.value = value
        l.moveToHead(node)
        return
    }

    node := &Node{key: key, value: value}
    l.cache[key] = node
    l.addToHead(node)

    if len(l.cache) > l.capacity {
        tail := l.removeTail()
        delete(l.cache, tail.key)
    }
}
```

### Python
```python
class Node:
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None


class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head

    def remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def add_to_head(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def move_to_head(self, node):
        self.remove(node)
        self.add_to_head(node)

    def remove_tail(self):
        node = self.tail.prev
        self.remove(node)
        return node

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self.move_to_head(node)
        return node.value

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            node = self.cache[key]
            node.value = value
            self.move_to_head(node)
            return

        node = Node(key, value)
        self.cache[key] = node
        self.add_to_head(node)

        if len(self.cache) > self.capacity:
            tail = self.remove_tail()
            del self.cache[tail.key]
```

### 复杂度
- 时间：`O(1)`
- 空间：`O(capacity)`

---

## 一句话记忆
**LRU = 哈希表负责快查，双向链表负责快删快插。**
