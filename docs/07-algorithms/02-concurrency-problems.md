# 并发相关算法题

> 考察频率：★★★☆☆  题型识别：生产者消费者、交替打印、并发安全

## 1. 生产者-消费者

### 题目描述
一个线程生产数据，另一个线程消费数据，要求线程安全、避免忙等待。

### 解法：Channel 实现

```go
func producer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)
}

func consumer(ch <-chan int) {
    for v := range ch {
        fmt.Println("consumed:", v)
    }
}

func main() {
    ch := make(chan int, 3) // 带缓冲 channel
    go producer(ch)
    consumer(ch)
}
```

### 解法：WaitGroup + Mutex

```go
func producerConsumer() {
    var wg sync.WaitGroup
    mu := sync.Mutex{}
    queue := []int{}
    cond := sync.NewCond(&mu)

    // 生产者
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < 5; i++ {
            mu.Lock()
            queue = append(queue, i)
            cond.Signal() // 通知消费者
            mu.Unlock()
            time.Sleep(100 * time.Millisecond)
        }
        mu.Lock()
        cond.Signal() // 通知结束
        mu.Unlock()
    }()

    // 消费者
    wg.Add(1)
    go func() {
        defer wg.Done()
        for {
            mu.Lock()
            for len(queue) == 0 {
                cond.Wait() // 等待通知，自动解锁
            }
            v := queue[0]
            queue = queue[1:]
            mu.Unlock()
            fmt.Println("consumed:", v)
            if v == 4 { // 最后一个
                return
            }
        }
    }()

    wg.Wait()
}
```

---

## 2. 交替打印（两个 goroutine）

### 题目描述
两个 goroutine 交替打印奇偶，输出 1,2,3,4,5,...

### 解法：Channel 信号

```go
func printAlternately() {
    ch1, ch2 := make(chan bool), make(chan bool)

    // 打印奇数
    go func() {
        for i := 1; i <= 10; i += 2 {
            <-ch1 // 等待信号
            fmt.Println(i)
            ch2 <- true // 通知偶数 goroutine
        }
    }()

    // 打印偶数
    go func() {
        for i := 2; i <= 10; i += 2 {
            <-ch2 // 等待信号
            fmt.Println(i)
            ch1 <- true // 通知奇数 goroutine
        }
    }()

    ch1 <- true // 启动
    time.Sleep(time.Second)
}
```

### 解法：Mutex

```go
func printAlternatelyMutex() {
    var mu sync.Mutex
    turn := 0 // 0: 奇数, 1: 偶数
    var wg sync.WaitGroup
    wg.Add(2)

    // 奇数
    go func() {
        defer wg.Done()
        for i := 1; i <= 10; i += 2 {
            mu.Lock()
            for turn != 0 {
                mu.Unlock()
                mu.Lock()
            }
            fmt.Println(i)
            turn = 1
            mu.Unlock()
        }
    }()

    // 偶数
    go func() {
        defer wg.Done()
        for i := 2; i <= 10; i += 2 {
            mu.Lock()
            for turn != 1 {
                mu.Unlock()
                mu.Lock()
            }
            fmt.Println(i)
            turn = 0
            mu.Unlock()
        }
    }()

    wg.Wait()
}
```

---

## 3. 三线程交替打印 A/B/C

### 解法

```go
func printABC() {
    ch1 := make(chan bool)
    ch2 := make(chan bool)
    ch3 := make(chan bool)

    printChar := func(chIn <-chan bool, chOut chan<- bool, char string) {
        for i := 0; i < 5; i++ {
            <-chIn
            fmt.Print(char)
            chOut <- true
        }
    }

    go printChar(ch1, ch2, "A")
    go printChar(ch2, ch3, "B")
    go printChar(ch3, ch1, "C")

    ch1 <- true
    time.Sleep(time.Second)
}
```

---

## 4. 并发安全计数器

### 题目描述
N 个 goroutine 同时给计数器 +1，最终结果必须为 N。

### 解法 1：Mutex

```go
type Counter struct {
    v   int
    mu  sync.Mutex
}

func (c *Counter) Inc() {
    c.mu.Lock()
    c.v++
    c.mu.Unlock()
}

func runCounter(n int) int {
    c := &Counter{}
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            c.Inc()
        }()
    }
    wg.Wait()
    return c.v
}
```

### 解法 2：Atomic

```go
func runCounterAtomic(n int) int64 {
    var v int64
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&v, 1)
        }()
    }
    wg.Wait()
    return v
}
```

### 性能对比

| 方法 | QPS（万） | CPU |
|------|----------|-----|
| Mutex | ~50 | 高（自旋+阻塞）|
| Atomic | ~120 | 低 |

---

## 5. 并发安全的 LRUCache

### 题目描述
实现一个并发安全的 LRU 缓存。

### 解法

```go
type Node struct {
    key, value int
    prev, next *Node
}

type LRUCache struct {
    capacity int
    mu       sync.RWMutex
    cache    map[int]*Node
    head, tail *Node // 伪头尾节点
}

func NewLRU(capacity int) *LRUCache {
    c := &LRUCache{
        capacity: capacity,
        cache:    make(map[int]*Node),
        head:     &Node{},
        tail:     &Node{},
    }
    c.head.next = c.tail
    c.tail.prev = c.head
    return c
}

func (c *LRUCache) Get(key int) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    if node, ok := c.cache[key]; ok {
        c.moveToFront(node)
        return node.value
    }
    return -1
}

func (c *LRUCache) Put(key, value int) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if node, ok := c.cache[key]; ok {
        node.value = value
        c.moveToFront(node)
        return
    }
    if len(c.cache) >= c.capacity {
        // 驱逐最久未使用的（tail.prev）
        remove := c.tail.prev
        c.removeNode(remove)
        delete(c.cache, remove.key)
    }
    newNode := &Node{key: key, value: value}
    c.addToFront(newNode)
    c.cache[key] = newNode
}

func (c *LRUCache) moveToFront(node *Node) {
    c.removeNode(node)
    c.addToFront(node)
}

func (c *LRUCache) removeNode(node *Node) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (c *LRUCache) addToFront(node *Node) {
    node.next = c.head.next
    node.prev = c.head
    c.head.next.prev = node
    c.head.next = node
}
```

---

## 6. 哲学家就餐问题

### 题目描述
5 个哲学家围坐圆桌，思考和吃饭交替，需要左右两把叉子才能吃饭。

### 解法：Channel + 资源顺序

```go
type Fork struct{ mu sync.Mutex }

func philosopher(left, right *Fork, id int) {
    for i := 0; i < 10; i++ {
        // 防止死锁：按固定顺序获取叉子
        first, second := left, right
        if left == right {
            first, second = right, left
        }

        first.mu.Lock()
        second.mu.Lock()

        fmt.Printf("philosopher %d eating\n", id)

        second.mu.Unlock()
        first.mu.Unlock()

        time.Sleep(time.Millisecond)
    }
}
```

### 解法：WaitGroup 资源通道

```go
func philosopherChan(id int, forks chan int) {
    for i := 0; i < 10; i++ {
        left := id
        right := (id + 1) % 5
        // 按固定顺序获取
        if left > right {
            left, right = right, left
        }
        <-forks // 获取第一把
        forks <- 1 // 释放（实际需分开两把）
    }
}
```

---

## 7. 并发打印素数（筛法）

### 题目描述
使用 goroutine 并行筛选素数。

```go
func primes(n int) []int {
    ch := make(chan int, n)
    for i := 2; i <= n; i++ {
        ch <- i
    }
    close(ch)

    var wg sync.WaitGroup
    var primes []int

    for v := range ch {
        wg.Add(1)
        go func(prime int) {
            defer wg.Done()
            out := make(chan int, n)
            for v := range ch {
                if v%prime != 0 {
                    out <- v
                }
            }
            close(out)
            ch = out
        }(v)
        primes = append(primes, v)
    }

    wg.Wait()
    return primes
}
```

---

## 面试追问

1. **Channel vs Mutex 何时用？**
   - Channel：适合传递数据所有权（ Ownership）、生产者-消费者、信号机制
   - Mutex：适合保护共享状态

2. **如何避免死锁？**
   - 按固定顺序获取锁
   - 设置超时
   - 使用 Channel 代替互斥锁
   - 避免嵌套锁

3. **什么是 goroutine 泄漏？如何排查？**
   - 泄漏：goroutine 被永久阻塞（channel 无人读、无 buffer channel 满了）
   - 排查：`go tool pprof` 看 goroutine 数量，`runtime.NumGoroutine()` 监控
