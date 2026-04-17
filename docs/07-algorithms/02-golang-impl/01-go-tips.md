# Go 刷题常用技巧

> 考察频率：★★★☆☆  题型识别：各类算法题中的 Go 特色实现

## 1. 排序与比较

### 内置 sort 包

```go
// int slice 排序
sort.Ints([]int{3, 1, 2})          // 原地排序
sorted := sort.IntsAreSorted(arr) // 检查是否已排序

// 字符串排序
sort.Strings([]string{"banana", "apple", "cherry"})

// 自定义排序（按长度）
sort.Slice(arr, func(i, j int) bool {
    return len(arr[i]) < len(arr[j])
})

// 稳定排序（维持相等元素原始顺序）
sort.SliceStable(arr, func(i, j int) bool {
    return arr[i] < arr[j]
})

// 查找（要求已排序）
i := sort.SearchInts(sortedArr, target)      // 返回第一个 >= target 的索引
i := sort.Search(len(arr), func(i int) bool { return arr[i] >= target })
```

### 优先队列（container/heap）

```go
import "container/heap"

type IntHeap []int

func (h IntHeap) Len() int            { return len(h) }
func (h IntHeap) Less(i, j int) bool  { return h[i] < h[j] } // 小顶堆
func (h IntHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any)         { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() any {
    old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}

// 使用
h := &IntHeap{2, 1, 5}
heap.Init(h)
heap.Push(h, 0)
heap.Pop(h) // 0
```

---

## 2. 字符串处理

### 常用操作

```go
// 字符串转 []byte（修改需拷贝）
b := []byte("hello")

// 字符串拼接（大量拼接用 strings.Builder）
var sb strings.Builder
for _, s := range strs {
    sb.WriteString(s)
}
result := sb.String()

// 分割
parts := strings.Split("a,b,c", ",")
fields := strings.Fields("  hello world  ") // 按空白字符分割

// 查找
strings.Contains("hello", "ll")      // true
strings.Index("hello", "l")          // 2
strings.HasPrefix("hello", "he")      // true
strings.ReplaceAll("hello", "l", "L")

// 字符转数字
n, _ := strconv.Atoi("123")
s := strconv.Itoa(123)
```

### rune vs byte

```go
// Go 字符串是 UTF-8 编码，遍历时注意 rune
s := "hello世界"
for i, r := range s { // r 是 rune 类型
    fmt.Printf("%d: %c\n", i, r)
}

// 中文字符处理
runes := []rune(s)
fmt.Println(string(runes[5:])) // 世界
```

---

## 3. 容器与数据结构

### map 操作

```go
// 创建与访问
m := make(map[string]int)
m["a"] = 1

// 判断 key 是否存在（重要！）
if v, ok := m["a"]; ok {
    fmt.Println("found:", v)
}

// 遍历（顺序随机）
for k, v := range m {
    fmt.Println(k, v)
}

// 删除
delete(m, "a")

// 嵌套 map
m := make(map[string]map[string]int)
if _, ok := m["outer"]["inner"]; !ok {
    m["outer"] = make(map[string]int)
}
```

### 结构体做 map key（需要可比较）

```go
// 可比较：结构体所有字段都可比较
type Point struct{ X, Y int }
m := map[Point]int{}

// 不可直接用作 key：slice、map、func
// 需要用其哈希值或自定义 key
```

---

## 4. 常用算法模板

### 去重

```go
// int 去重
func dedup(nums []int) []int {
    seen := make(map[int]bool)
    result := []int{}
    for _, n := range nums {
        if !seen[n] {
            seen[n] = true
            result = append(result, n)
        }
    }
    return result
}

// string 去重（保持顺序）
func uniqueBytes(s string) string {
    seen := [256]bool{}
    var result []byte
    for _, b := range []byte(s) {
        if !seen[b] {
            seen[b] = true
            result = append(result, b)
        }
    }
    return string(result)
}
```

### 计数

```go
// 有限字符集计数
count := [26]int{}
count['a' - 'a']++

// 通用计数
count := make(map[rune]int)
for _, c := range s {
    count[c]++
}
```

### 前缀和

```go
// 一维前缀和
prefix := make([]int, len(nums)+1)
for i, v := range nums {
    prefix[i+1] = prefix[i] + v
}
// [l..r] 的和
sum := prefix[r+1] - prefix[l]

// 二维前缀和
prefix := make([][]int, m+1)
for i := range prefix {
    prefix[i] = make([]int, n+1)
}
for i := 1; i <= m; i++ {
    for j := 1; j <= n; j++ {
        prefix[i][j] = prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1] + matrix[i-1][j-1]
    }
}
```

### 差分数组

```go
// 区间批量加法，最后求前缀和得到最终数组
diff := make([]int, n+1)
for _, op := range ops {
    diff[op[0]] += op[2]
    diff[op[1]+1] -= op[2]
}
```

---

## 5. 数学库（math）

```go
import "math"

math.MaxInt   // 最大 int（需强转）
math.MinInt   // 最小 int
math.MaxFloat64
math.Inf(1)   // 正无穷
math.IsInf(v, 1)

math.Abs(-1.5)  // 1.5
math.Ceil(1.2)   // 2.0
math.Floor(1.8)  // 1.0
math.Round(1.5)  // 2（四舍五入）

// 整数绝对值（防溢出）
func abs(x int) int {
    if x < 0 {
        return -x
    }
    return x
}
```

---

## 6. 常见坑点

### 1. 切片传参是引用，但 append 可能触发扩容

```go
func modify(s []int) {
    s = append(s, 4) // 不影响原切片！
}
func modifyPtr(s *[]int) {
    *s = append(*s, 4) // 影响原切片
}
```

### 2. map 并发不安全

```go
// 错误
go func() { m["a"] = 1 }()
go func() { _ = m["a"] }()

// 加锁或用 sync.RWMutex
var mu sync.RWMutex
mu.Lock()
m["a"] = 1
mu.Unlock()
```

### 3. for range 循环变量是复用

```go
// 错误：所有 goroutine 共享相同的 i
for _, v := range values {
    go func() { fmt.Println(v) }()
}

// 正确：传参
for _, v := range values {
    v := v // 局部 shadow
    go func() { fmt.Println(v) }()
}
```

### 4. 切片初始化

```go
// make([]int, 0, n) vs make([]int, n)
// make([]int, 0, n): len=0, cap=n，可append
// make([]int, n): len=n, cap=n，已填零
```

### 5. 字符串不可变

```go
s := "hello"
// s[0] = 'H' // 编译错误
b := []byte(s); b[0] = 'H'; s = string(b) // 必须这样改
```

---

## 7. 常用标准库速查

| 功能 | 包 | 关键函数 |
|------|-----|---------|
| 最大/小堆 | container/heap | heap.Push/Pop |
| 双向队列 | container/list | List.Push/Pop |
| 排序 | sort | Ints, Slice, Search |
| 字符串 | strings | Split, Contains, Builder |
| 字符转换 | strconv | Atoi, Itoa |
| 整型转换 | strconv | FormatInt, ParseInt |
| 数学 | math | Abs, Max, Min, Ceil, Floor |
| 随机 | math/rand | Intn, Shuffle |
| 时间 | time | Now, Sleep, Parse |
| JSON | encoding/json | Marshal, Unmarshal |
