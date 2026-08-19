# 手写代码面试（Live Coding）应对策略

> 考察频率：★★★★★（几乎所有技术一面必考）  难度：★★★★☆
> 适用：所有技术轮次，Go 工程师常考并发 + 数据结构

---

## 核心答案（30 秒版）

Live Coding 不只看你能不能写出正确代码，更看**你写代码的过程**。正确的流程：**读题→确认边界→想思路→边说边走→写代码→测试用例→分析复杂度**。Go 工程师高频考点：**Channel 用法、Goroutine 安全、Slice/Map 操作、排序搜索、贪心/动态规划**。

---

## 1. Go Live Coding 常见题型

### 【🔥🔥🔥🔥🔥】并发生成器 / 数据管道

```go
// 高频题：实现一个 fan-in 模式的数据管道
func MergeSortedChannels(channels []<-chan int) <-chan int {
    out := make(chan int)
    
    var wg sync.WaitGroup
    // 每个输入通道一个 goroutine
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    
    // 等所有输入都关闭后，关闭输出
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### 【🔥🔥🔥🔥🔥】LRU Cache

```go
// 高频题：实现线程安全的 LRU Cache（Go 高级工程师必会）
type LRUCache struct {
    capacity int
    cache    map[int]*Node
    head     *Node // dummy head
    tail     *Node // dummy tail
}

type Node struct {
    key, val int
    prev, next *Node
}

func (l *LRUCache) Get(key int) int { ... }
func (l *LRUCache) Put(key, val int) { ... }

// 面试官追问：
// Q: 如果容量很大（百万级），你的双向链表会不会成为瓶颈？
// A: 可以引入分片机制，按 hash(key) % shards 分散到多个小 LRU
```

### 【🔥🔥🔥🔥】goroutine 泄漏排查

```go
// 高频题：这段代码有 goroutine 泄漏吗？怎么修复？
func getData(ctx context.Context) []string {
    ch := make(chan string)
    
    go func() {
        items := fetchFromDB()  // 可能很慢
        for _, item := range items {
            ch <- item  // ← 如果有人没读，goroutine 就会阻塞
        }
        close(ch)
    }()
    
    result := make([]string, 0, 10)
    for i := 0; i < 10; i++ {
        result = append(result, <-ch)
    }
    return result
}

// 问题：如果 fetchFromDB 返回超过 10 个元素，goroutine 会在第 11 个写入时永久阻塞！
// 修复：用 select + ctx.Done()
```

### 【🔥🔥🔥🔥】字符串处理 / Trie

```go
// 高频题：前缀树实现（单词查找表）
type TrieNode struct {
    children [26]*TrieNode
    isEnd    bool
}

type Trie struct {
    root *TrieNode
}

func (t *Trie) Insert(word string) { ... }
func (t *Trie) Search(word string) bool { ... }
func (t *Trie) StartsWith(prefix string) bool { ... }
```

### 【🔥🔥🔥🔥】并发计数 / Rate Limiter

```go
// 高频题：实现一个并发安全的限速器
type RateLimiter struct {
    mu       sync.Mutex
    tokens   float64
    max      float64
    rate     float64  // tokens per second
    lastTime time.Time
}

func (r *RateLimiter) Allow() bool {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    now := time.Now()
    elapsed := now.Sub(r.lastTime).Seconds()
    r.tokens += elapsed * r.rate
    if r.tokens > r.max {
        r.tokens = r.max
    }
    r.lastTime = now
    
    if r.tokens >= 1 {
        r.tokens--
        return true
    }
    return false
}
```

---

## 2. 标准解题流程（面试版）

### 五步法（控制 15-20 分钟）

```
Step 1 | 读题 & 确认边界（2 min）
─────────────────────────────
- "这个输入的取值范围是多少？"
- "会有重复元素吗？"
- "数组是有序还是无序？"
- "有没有极端情况需要考虑？比如空输入？"

Step 2 | 想思路，先口头描述（3 min）
─────────────────────────────
- "我先想想...暴力解法是 O(n²)，遍历每个元素和后面所有元素比较。"
- "但我想到可以用 Hash Map 把时间降到 O(n)。"
- "让我画一下思路..."

Step 3 | 开始编码，边写边说（8 min）
─────────────────────────────
- 先写函数签名（确定参数和返回值）
- 再写主逻辑，遇到不确定就先写注释
- 边写边解释："这里我用双指针，因为..."
- 如果卡住了，说出来："这块我需要思考一下..."

Step 4 | 自测用例（3 min）
─────────────────────────────
- "我来手动 trace 一下几个例子"
- 正常用例 + 边界用例（空输入、单元素、最大最小值）

Step 5 | 复杂度分析（2 min）
─────────────────────────────
- "时间复杂度是 O(n log n)，因为用了排序"
- "空间复杂度是 O(n)，因为我开了一个新数组"
- "如果要优化到 O(1) 空间，可以考虑 XXX，但那样可读性差"
```

---

## 3. Go 特有注意事项

### 陷阱清单

| 陷阱 | 说明 | 如何避免 |
|------|------|---------|
| **闭包捕获变量** | for g 中的 g 在循环后被修改 | 传参或创建局部副本 |
| **defer 执行顺序** | LIFO，最后一个 defer 最先执行 | 注意清理顺序 |
| **nil channel** | 向 nil channel 发送会永久阻塞 | 确保 channel 已初始化 |
| **切片共享底层数组** | 截取 slice 仍共享底层数组 | 需要 copy 时使用 make+copy |
| **select 全阻塞** | 没有 default 时 select 永远阻塞 | 加 default 分支或 timeout |

### 经典坑题及回答

```go
// 坑题 1：for 循环闭包
func main() {
    tasks := []func(){}
    for _, v := range []int{1, 2, 3} {
        tasks = append(tasks, func() { fmt.Println(v) })
    }
    for _, task := range tasks {
        task()
    }
}
// 输出：3 3 3 （不是预期的 1 2 3！）
// 修复：func() { v := v; fmt.Println(v) } 或在函数参数中传递

// 坑题 2：defer 中的匿名函数捕获
func foo() {
    s := []string{"a", "b", "c"}
    for _, v := range s {
        defer fmt.Print(v)
    }
}
// 输出：cba（defer LIFO）
```

---

## 4. 面试心态与沟通技巧

### 正确的心态

```
✅ 面试官想看的是你怎么思考，不是你能不能在 5 分钟内写出完美代码
✅ 卡壳了很正常，说出来比沉默强一万倍
✅ 即使最后没做出来，展示了正确的思路也能拿高分
❌ 一上来就闷头敲键盘（等于放弃了展示思考过程的机会）
❌ 面试官提示后还说"我知道"然后继续犯同样的错误
❌ 写完代码不说复杂度分析
```

### 有效沟通模板

```
当我理解题目时：
"好的，我的理解是需要实现一个 X，满足 Y 约束，对吗？"

当我要换思路时：
"我刚才是用 A 方法，但我发现有个问题是 B，所以我想尝试 C 方法。"

当我遇到困难时：
"这部分我有点不确定，我有两个思路：方案一是 X，方案二是 Y。我觉得方案一更好因为..."

当我快完成时：
"基本上框架已经搭好了，接下来我需要补充一些边界条件的处理。"

当完全不会时：
"这道题我没见过类似的，不过根据题目要求，我会先从暴力解法开始，然后再考虑优化。我的初步想法是..."
```

---

## 延伸阅读

- [LeetCode 热题 TOP 100](https://leetcode.cn/studyplan/top-interview-questions/)
- [Go 语言并发编程最佳实践](https://go101.org/article/concurrency.html)
