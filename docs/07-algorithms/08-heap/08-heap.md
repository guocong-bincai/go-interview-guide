# 堆 / 优先队列

> 考察频率：★★★☆☆  题型识别：TopK、第 K 个最值、合并有序列表

## Go 实现

Go 标准库 `container/heap` 是**小顶堆**。实现方式：定义 `Heap` 类型，实现 `Len() / Less() / Swap() / Push() / Pop()` 五个接口。

### 最小堆模板

```go
type IntHeap []int

func (h IntHeap) Len() int            { return len(h) }
func (h IntHeap) Less(i, j int) bool  { return h[i] < h[j] } // 小顶堆
func (h IntHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any)         { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}
```

### 最大堆
把 `Less` 改为 `return h[i] > h[j]`

---

## 题型一：Top K（LeetCode 703）

### 题目
数据流中不断流入数据，随时返回当前第 K 大的元素。

### 思路
维护大小为 K 的小顶堆。堆顶 = 第 K 大的元素。

```go
type KthLargest struct {
    k    int
    heap *IntHeap
}

func Constructor(k int, nums []int) KthLargest {
    h := IntHeap(nums)
    heap.Init(&h)
    // 保证堆大小为 k
    for h.Len() > k {
        heap.Pop(&h)
    }
    return KthLargest{k: k, heap: &h}
}

func (this *KthLargest) Add(val int) int {
    heap.Push(this.heap, val)
    if this.heap.Len() > this.k {
        heap.Pop(this.heap)
    }
    return (*this.heap)[0] // 堆顶即第 K 大
}
```

### 追问：求 Top K 大的元素（LeetCode 215）

```go
func findKthLargest(nums []int, k int) int {
    h := IntHeap(nums)
    heap.Init(&h)

    result := 0
    for i := 0; i < k; i++ {
        result = heap.Pop(&h).(int)
    }
    return result
}
```

### 优化：Quick Select（平均 O(n)）
```go
func findKthLargest(nums []int, k int) int {
    return quickSelect(nums, 0, len(nums)-1, len(nums)-k)
}

func quickSelect(nums []int, left, right, k int) int {
    if left == right {
        return nums[left]
    }
    pivot := partition(nums, left, right)
    if k == pivot {
        return nums[pivot]
    } else if k < pivot {
        return quickSelect(nums, left, pivot-1, k)
    } else {
        return quickSelect(nums, pivot+1, right, k)
    }
}

func partition(nums []int, left, right int) int {
    pivot := right
    cnt := left
    for i := left; i < right; i++ {
        if nums[i] < nums[pivot] {
            nums[i], nums[cnt] = nums[cnt], nums[i]
            cnt++
        }
    }
    nums[cnt], nums[pivot] = nums[pivot], nums[cnt]
    return cnt
}
```

---

## 题型二：合并 K 个有序链表（LeetCode 23）

### 思路
把所有链表头节点加入小顶堆，每次弹出最小节点，加入结果链表，弹出节点的下一个再入堆。

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

func mergeKLists(lists []*ListNode) *ListNode {
    dummy := &ListNode{}
    cur := dummy

    // 节点堆（按 Val 小顶堆）
    nodeHeap := &NodeHeap{}
    heap.Init(nodeHeap)

    for _, head := range lists {
        if head != nil {
            heap.Push(nodeHeap, head)
        }
    }

    for nodeHeap.Len() > 0 {
        node := heap.Pop(nodeHeap).(*ListNode)
        cur.Next = node
        cur = cur.Next
        if node.Next != nil {
            heap.Push(nodeHeap, node.Next)
        }
    }
    return dummy.Next
}

// NodeHeap 实现
type NodeHeap []*ListNode

func (h NodeHeap) Len() int            { return len(h) }
func (h NodeHeap) Less(i, j int) bool  { return h[i].Val < h[j].Val }
func (h NodeHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *NodeHeap) Push(x any)         { *h = append(*h, x.(*ListNode)) }
func (h *NodeHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}
```

### 复杂度
- 时间：O(N log K)，N 是总节点数，K 是链表数
- 空间：O(K) 堆大小

---

## 题型三：数据流中位数（LeetCode 295）

### 思路：两个堆
- `maxHeap`：存较小的一半，最大堆顶即左半边最大
- `minHeap`：存较大的一半，最小堆顶即右半边最小
- 维持 `maxHeap.Len() == minHeap.Len()` 或 `maxHeap 多一个`

### 代码

```go
type MedianFinder struct {
    maxHeap, minHeap *IntHeap // maxHeap 存负数实现大顶堆
}

func ConstructorMedian() MedianFinder {
    return MedianFinder{
        maxHeap: &IntHeap{},
        minHeap: &IntHeap{},
    }
}

func (f *MedianFinder) AddNum(num int) {
    // 先入大顶堆（较小的半部分）
    heap.Push(f.maxHeap, -num)

    // 平衡：大顶堆堆顶 <= 小顶堆堆顶
    if f.maxHeap.Len() > 0 && f.minHeap.Len() > 0 &&
        -(*f.maxHeap)[0] > (*f.minHeap)[0] {
        v := -heap.Pop(f.maxHeap).(int)
        heap.Push(f.minHeap, v)
    }

    // 保证 maxHeap 最多比 minHeap 多 1 个
    if f.maxHeap.Len() > f.minHeap.Len()+1 {
        v := -heap.Pop(f.maxHeap).(int)
        heap.Push(f.minHeap, v)
    }

    // 保证 maxHeap 和 minHeap 大小差 ≤ 1
    if f.minHeap.Len() > f.maxHeap.Len() {
        v := heap.Pop(f.minHeap).(int)
        heap.Push(f.maxHeap, -v)
    }
}

func (f *MedianFinder) FindMedian() float64 {
    if f.maxHeap.Len() == 0 {
        return 0
    }
    if f.maxHeap.Len() > f.minHeap.Len() {
        return float64(-(*f.maxHeap)[0])
    }
    return (float64(-(*f.maxHeap)[0]) + float64((*f.minHeap)[0])) / 2.0
}
```

---

## 题型四：最后一块石头的重量（LeetCode 1046）

### 题目
每次取两个最重的石头碰撞，重量相减，直到 ≤ 1 块石头。

```go
func lastStoneWeight(stones []int) int {
    h := IntHeap(stones)
    heap.Init(&h)

    for h.Len() > 1 {
        y := heap.Pop(&h).(int) // 最重
        x := heap.Pop(&h).(int) // 第二重
        if y > x {
            heap.Push(&h, y-x)
        }
    }
    if h.Len() == 0 {
        return 0
    }
    return heap.Pop(&h).(int)
}
```

---

## 题型五：滑动窗口最大值（堆版，补充单调队列版）

```go
func maxSlidingWindowHeap(nums []int, k int) []int {
    result := []int{}
    h := &MaxIntHeap{}
    heap.Init(h)

    for i := 0; i < len(nums); i++ {
        heap.Push(h, &Item{nums[i], i})

        // 移除超出窗口的元素
        for len(*h) > 0 && (*h)[0].Index <= i-k {
            heap.Pop(h)
        }

        // 窗口满了才记录
        if i >= k-1 {
            result = append(result, (*h)[0].Val)
        }
    }
    return result
}

type Item struct {
    Val  int
    Index int
}

type MaxIntHeap []*Item

func (h MaxIntHeap) Len() int            { return len(h) }
func (h MaxIntHeap) Less(i, j int) bool { return h[i].Val > h[j].Val } // 大顶堆
func (h MaxIntHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MaxIntHeap) Push(x any)         { *h = append(*h, x.(*Item)) }
func (h *MaxIntHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}
```

---

## 堆操作时间复杂度

| 操作 | 时间复杂度 |
|------|-----------|
| Push | O(log n) |
| Pop | O(log n) |
| 查看堆顶 | O(1) |
| 建堆（从数组） | O(n) |
| 排序 | O(n log n) |

### TopK 问题对比

| 方法 | 时间 | 空间 | 适用场景 |
|------|------|------|---------|
| 全部排序 | O(n log n) | O(1) | 无 |
| 堆 | O(n log k) | O(k) | 数据流、TopK |
| Quick Select | O(n) 平均 | O(1) | 一次性求第 K 大 |

---

## 面试高频追问

1. **堆和二叉搜索树的区别？**
   - BST：中序遍历有序，查找 O(log n)
   - 堆：只保证堆顶最大/最小，其他节点无序

2. **堆的底层实现？**
   - 完全二叉树，用数组存储（索引 i 的左子=2i+1，右子=2i+2，父=(i-1)/2）
   - 优点：数组存储无指针开销，内存连续，缓存友好

3. **如何手写一个大顶堆？**
   → 改 Less 函数；或者把元素取负数存入小顶堆
