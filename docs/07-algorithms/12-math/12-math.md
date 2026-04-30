# 数学类高频题（数论 + 模拟 + 随机）

> 考察频率：★★★☆☆  优先级：P1  Go + Python 双语
> 关键词：快速幂、质数筛、水塘抽样、模拟、概率

[🏠 首页](../../../README.md) · [📚 算法模块](../README.md)

---

## 面试官考察意图

数学类面试题看似"数学"，本质考的是**算法思维**：大整数处理、概率算法、矩阵快速幂。高级工程师要在 5 分钟内给出正确解法并分析复杂度。

---

## 核心答案（30 秒版）

| 题型 | 核心技巧 | 代表题 |
|------|---------|------|
| 快速幂 | `x^n = (x^{n/2})^2`，奇数多乘一次 x | LeetCode 50 |
| 质数判定 | 只需除到 √n | LeetCode 204 |
| 水塘抽样 | 等概率保证，每步更新概率 = 1/k | Random Pick |
| 矩阵幂 | 二分 + 矩阵乘法 | LeetCode 50 变种 |
| 随机算法 | 拒绝采样、Monte Carlo | 随机指针 |

---

## 1. 快速幂（必考！）

### LeetCode 50 - Pow(x, n)

**问题：** 实现 `pow(x, n)`，计算 x 的 n 次方，要求 log 时间。

**思路：** 将 n 视为二进制，逢 1 乘当前结果，逢 0 不乘。

```go
// Go 快速幂：O(log n)
func myPow(x float64, n int) float64 {
    if n < 0 {
        x = 1 / x
        n = -n // 注意：int 最小值 -n 会溢出，用 int64 中转
    }
    result := 1.0
    base := x
    for n > 0 {
        if n%2 == 1 {
            result *= base
        }
        base *= base
        n /= 2
    }
    return result
}
```

```python
# Python 快速幂
def myPow(self, x: float, n: int) -> float:
    if n < 0:
        x = 1 / x
        n = -n
    result = 1.0
    base = x
    while n > 0:
        if n % 2 == 1:
            result *= base
        base *= base
        n //= 2
    return result
```

**复杂度：** 时间 O(log n)，空间 O(1)

**追问：** n 是 int64 最小值（-2^63）时，- n 会溢出。解法：用 uint64 或先把 n 转为正数：

```go
func myPowSafe(x float64, n int64) float64 {
    if n < 0 {
        x = 1 / x
        n = -n
    }
    // 用无符号转，避免 int64 最小值溢出
    return powUint64(x, uint64(n))
}

func powUint64(x float64, n uint64) float64 {
    result := 1.0
    base := x
    for n > 0 {
        if n&1 == 1 { // 位运算比 % 更快
            result *= base
        }
        base *= base
        n >>= 1
    }
    return result
}
```

---

## 2. 质数判定与筛选

### LeetCode 204 - 计数质数

**问题：** 统计小于 n 的质数数量（n ≤ 5×10^6）。

**思路：** 埃拉托色尼筛法（Sieve of Eratosthenes），从 2 开始标记倍数为合数。

```go
// Go 埃拉托色尼筛法：O(n log log n)
func countPrimes(n int) int {
    if n <= 2 {
        return 0
    }
    // isPrime[i] 表示 i 是否为质数（0 和 1 特殊处理）
    isPrime := make([]bool, n)
    for i := 2; i < n; i++ {
        isPrime[i] = true
    }
    
    // 只需要筛选到 sqrt(n)
    limit := int(math.Sqrt(float64(n)))
    for i := 2; i <= limit; i++ {
        if isPrime[i] {
            // 从 i*i 开始标记（之前的是旧因子的倍数）
            for j := i * i; j < n; j += i {
                isPrime[j] = false
            }
        }
    }
    
    count := 0
    for i := 2; i < n; i++ {
        if isPrime[i] {
            count++
        }
    }
    return count
}
```

```python
# Python 埃拉托色尼筛法
def countPrimes(self, n: int) -> int:
    if n <= 2:
        return 0
    isPrime = [True] * n
    isPrime[0] = isPrime[1] = False
    
    limit = int(n ** 0.5)
    for i in range(2, limit + 1):
        if isPrime[i]:
            for j in range(i * i, n, i):
                isPrime[j] = False
    
    return sum(isPrime)
```

**为什么从 i*i 开始标记：**
- i × (i-1) 已经在之前被标记过（因为 (i-1) < i）
- i × i 是第一个未被处理过的最小倍数

**复杂度：** 时间 O(n log log n)，空间 O(n)

---

## 3. 水塘抽样（Reservoir Sampling）

### LeetCode 382 - 链表随机节点

**问题：** 等概率随机返回链表中的一个节点，要求 O(1) 空间。

**思路：** 遍历链表，第 k 个节点以 1/k 概率替换结果。

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
type Solution struct {
    head *ListNode
    rnd  *rand.Rand
}

func Constructor(head *ListNode) Solution {
    return Solution{
        head: head,
        rnd:  rand.New(rand.NewSource(time.Now().UnixNano())),
    }
}

func (s *Solution) GetRandom() int {
    var result int
    count := 0
    cur := s.head
    
    for cur != nil {
        count++
        // 第 count 个节点以 1/count 概率替换当前结果
        if s.rnd.Intn(count) == 0 {
            result = cur.Val
        }
        cur = cur.Next
    }
    return result
}
```

```python
# Python 水塘抽样
class Solution:
    def __init__(self, head: Optional[ListNode]):
        self.head = head
        import random
        self.rng = random.Random(time.time())
    
    def getRandom(self) -> int:
        result = None
        count = 0
        cur = self.head
        
        while cur:
            count += 1
            if self.rng.randint(1, count) == 1:
                result = cur.val
            cur = cur.next
        return result
```

**正确性证明（数学归纳法）：**
- 第 1 个节点被选中概率 = 1/1 = 1 ✓
- 假设第 k 个节点被选中概率 = 1/k
- 第 k+1 个节点：以 1/(k+1) 概率替换 → 被选中概率 = 1/(k+1)
- 第 k 个节点：不被替换概率 = k/(k+1)，已有概率 1/k → 最终 = 1/(k+1) ✓

---

## 4. 阶乘后的零（LeetCode 172）

**问题：** 统计 n! 末尾有多少个零。

**思路：** 末尾的零来自因子 10 = 2 × 5，2 足够多，问题化为数 n! 包含多少个因子 5。

```go
// Go：统计因子 5 的个数
func trailingZeroes(n int) int {
    count := 0
    for n > 0 {
        n /= 5  // 5^1 + 5^2 + 5^3 + ...
        count += n
    }
    return count
}
```

```python
# Python
def trailingZeroes(self, n: int) -> int:
    count = 0
    while n > 0:
        n //= 5
        count += n
    return count
```

**为什么是除以 5：**
- n = 25 时，有因子 5 的数：5, 10, 15, 20, 25（5个），但 25 = 5×5，有两个 5
- n/5 + n/25 + n/125 + ... 恰好把所有 5 的幂次相加

---

## 5. 螺旋矩阵（LeetCode 54）

**问题：** 给定 m×n 矩阵，按顺时针螺旋顺序返回所有元素。

**思路：** 模拟遍历，按层剥离，四条边界交替收缩。

```go
// Go 螺旋矩阵
func spiralOrder(matrix [][]int) []int {
    if len(matrix) == 0 || len(matrix[0]) == 0 {
        return []int{}
    }
    m, n := len(matrix), len(matrix[0])
    result := make([]int, 0, m*n)
    
    top, bottom, left, right := 0, m-1, 0, n-1
    
    for left <= right && top <= bottom {
        // 左→右
        for col := left; col <= right; col++ {
            result = append(result, matrix[top][col])
        }
        top++
        
        // 上→下
        for row := top; row <= bottom; row++ {
            result = append(result, matrix[row][right])
        }
        right--
        
        if top <= bottom {
            // 右→左
            for col := right; col >= left; col-- {
                result = append(result, matrix[bottom][col])
            }
            bottom--
        }
        
        if left <= right {
            // 下→上
            for row := bottom; row >= top; row-- {
                result = append(result, matrix[row][left])
            }
            left++
        }
    }
    return result
}
```

```python
# Python 螺旋矩阵
def spiralOrder(self, matrix: List[List[int]]) -> List[int]:
    if not matrix:
        return []
    m, n = len(matrix), len(matrix[0])
    result = []
    top, bottom, left, right = 0, m-1, 0, n-1
    
    while left <= right and top <= bottom:
        # 左→右
        for col in range(left, right+1):
            result.append(matrix[top][col])
        top += 1
        
        # 上→下
        for row in range(top, bottom+1):
            result.append(matrix[row][right])
        right -= 1
        
        if top <= bottom:
            # 右→左
            for col in range(right, left-1, -1):
                result.append(matrix[bottom][col])
            bottom -= 1
        
        if left <= right:
            # 下→上
            for row in range(bottom, top-1, -1):
                result.append(matrix[row][left])
            left += 1
    
    return result
```

**复杂度：** 时间 O(m×n)，空间 O(1)

---

## 6. 随机指针链表复制（LeetCode 138）

**问题：** 链表每个节点有 random 指针指向随机节点，深度复制。

**思路：** 三步法：①复制节点接原节点后面 ②设置 random ③拆分。

```go
// Go 随机指针链表复制：三步法
func copyRandomList(head *Node) *Node {
    if head == nil {
        return nil
    }
    
    // 第一步：复制节点，插入原节点后面
    cur := head
    for cur != nil {
        copy := &Node{Val: cur.Val, Next: cur.Next}
        cur.Next = copy
        cur = copy.Next
    }
    
    // 第二步：设置 random 指针
    cur = head
    for cur != nil {
        if cur.Random != nil {
            cur.Next.Random = cur.Random.Next
        }
        cur = cur.Next.Next
    }
    
    // 第三步：拆分新旧链表
    cur = head
    copyHead := head.Next
    for cur != nil {
        copy := cur.Next
        cur.Next = copy.Next
        if copy.Next != nil {
            copy.Next = copy.Next.Next
        }
        cur = cur.Next
    }
    return copyHead
}
```

```python
# Python 随机指针链表复制：三步法
def copyRandomList(self, head: 'Optional[Node]') -> 'Optional[Node]':
    if not head:
        return None
    
    # 第一步：复制节点插入后面
    cur = head
    while cur:
        copy = Node(cur.val)
        copy.next = cur.next
        cur.next = copy
        cur = copy.next
    
    # 第二步：设置 random
    cur = head
    while cur:
        if cur.random:
            cur.next.random = cur.random.next
        cur = cur.next.next
    
    # 第三步：拆分
    cur = head
    copy_head = head.next
    while cur:
        copy = cur.next
        cur.next = copy.next
        copy.next = copy.next.next if copy.next else None
        cur = cur.next
    
    return copy_head
```

**复杂度：** 时间 O(n)，空间 O(1)

---

## 7. 数学类题复杂度速查

| 题型 | 时间复杂度 | 空间复杂度 | 关键技巧 |
|------|-----------|-----------|---------|
| 快速幂 | O(log n) | O(1) | 位运算替代 % |
| 质数埃氏筛 | O(n log log n) | O(n) | i*i 起始优化 |
| 水塘抽样 | O(n) | O(1) | 1/k 替换概率 |
| 阶乘末尾零 | O(log n) | O(1) | 除 5 的幂次和 |
| 螺旋矩阵 | O(m×n) | O(1) | 四边界交替 |
| 随机链表复制 | O(n) | O(1) | 三步原地复制 |

---

## 延伸阅读

- [LeetCode 50 - Pow(x, n)](https://leetcode.com/problems/powx-n/)
- [LeetCode 204 - 计数质数](https://leetcode.com/problems/count-primes/)
- [LeetCode 382 - 链表随机节点（水塘抽样）](https://leetcode.com/problems/linked-list-random-node/)
- 《算法导论》第 5 章：概率分析和随机算法（水塘抽样正确性证明）
