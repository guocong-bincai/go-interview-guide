# Go 内存模型

> 考察频率：★★★★☆  难度：★★★★☆

## 什么是 Happens-Before

Go 内存模型定义了**在同一地址上写操作 w1 和读操作 r2 之间是否存在保证**。

> 如果 w1 在 happens-before r2，则 r2 一定能读到 w1 写入的值。

### 三种顺序保证

| 顺序 | 说明 |
|------|------|
| **Happens-Before** | 之前操作完成后才执行后续操作 |
| **Synchronizes-With** | 通道/锁等同步原语建立的 happens-before |
| **Memory Order** | CPU 乱序执行下，编译器和硬件的 reorder 限制 |

---

## Synchronizes-With 关系

### 1. Channel

```go
c := make(chan int)

// 发送端
go func() {
    c <- 1  // Sends-before
}()

// 接收端
<-c  // Receives-after
```

| 场景 | 保证 |
|------|------|
| `c <- v` | happens-before `<-c` 完成 |
| `close(c)` | happens-before 接收方收到零值 |
| `v := <-c` | happens-before `c` 的发送完成 |

### 2. Mutex / RWMutex

```go
var mu sync.Mutex
var x int

go func() {
    mu.Lock()
    x = 1       // Write
    mu.Unlock() // Release
}()

go func() {
    mu.Lock()
    print(x)    // Read
    mu.Unlock() // Acquire
}()
```

- `mu.Lock()` happens-before `mu.Unlock()`（同一个 goroutine）
- `Unlock()` happens-before 下一个 `Lock()`（不同 goroutine）

### 3. Once

```go
var once sync.Once
var x int

go once.Do(func() { x = 1 })
go once.Do(func() { print(x) }) // x == 1
```

`once.Do(f)` 保证 `f()` 执行完成后，所有等待的 goroutine 才继续执行。

### 4. sync.WaitGroup

```go
var wg sync.WaitGroup
var x int

wg.Add(1)
go func() {
    x = 42
    wg.Done()
}()

wg.Wait()
print(x) // x == 42
```

`Done()` happens-before `Wait()` 返回。

### 5. atomic

```go
var x int64

go func() { atomic.AddInt64(&x, 1) }()
go func() { atomic.AddInt64(&x, 1) }()
// 无锁保证顺序，结果确定（两次 +1 = 2），但不保证具体顺序
```

`sync/atomic` 提供 **sequentially consistent**（顺序一致性）或 **acquire/release** 语义。

---

## 常见坑：竞态读

### 延迟初始化问题

```go
// 不安全：x 可能被部分构造
var x *int
go func() { x = new(int); *x = 42 }()
go func() { print(*x) }()

// 安全：用 sync.Once
var once sync.Once
var x int
go func() {
    once.Do(func() { x = 42 })
}()
go func() { print(x) }()

// 安全：用 channel
ch := make(chan int)
go func() { ch <- 42 }()
go func() { print(<-ch) }()
```

### for-range loop 变量捕获

```go
// Go 1.22 之前：所有 goroutine 共享循环变量 i
for _, v := range []int{1, 2, 3} {
    go func() {
        fmt.Println(v) // 输出 3, 3, 3（不确定）
    }()
}

// Go 1.22+：每个迭代创建新变量，安全
for _, v := range []int{1, 2, 3} {
    go func(val int) {
        fmt.Println(val) // 输出 1, 2, 3
    }(v)
}
```

**在 Go 1.22 之前**：for 循环的循环变量在整个循环中共享，goroutine 创建时捕获的是变量的引用而非值。

### 闭包捕获引用

```go
// 泄漏
var funcs []func()
for _, v := range []int{1, 2, 3} {
    funcs = append(funcs, func() { print(v) })
}
for _, f := range funcs {
    f() // 输出 3, 3, 3
}

// 修复：传参
var funcs []func()
for _, v := range []int{1, 2, 3} {
    val := v // 每个迭代创建新变量
    funcs = append(funcs, func() { print(val) })
}
```

---

## 内存对齐

### 什么是内存对齐

编译器会根据字段类型将结构体字段安排在特定偏移量上，保证每个字段的起始地址是其大小的倍数。

```go
type A struct {
    a bool   // 1 byte，偏移 0
    // 编译器插入 padding，假设按 8 字节对齐：
    // padding 7 bytes
    b int64  // 8 bytes，偏移 8
} // A 大小 = 16 bytes

type B struct {
    a bool   // 1 byte，偏移 0
    b int64  // 8 bytes，偏移 8
    c bool   // 1 byte，偏移 16
    // 末尾 padding 7 bytes
} // B 大小 = 24 bytes
```

### 用 unsafe 验证

```go
import "unsafe"

func main() {
    var a A
    var b B
    fmt.Println(unsafe.Sizeof(a)) // 16
    fmt.Println(unsafe.Sizeof(b)) // 24

    fmt.Println(unsafe.Offsetof(a.a)) // 0
    fmt.Println(unsafe.Offsetof(a.b)) // 8

    fmt.Println(unsafe.Offsetof(b.a)) // 0
    fmt.Println(unsafe.Offsetof(b.b)) // 8
    fmt.Println(unsafe.Offsetof(b.c)) // 16
}
```

### 调整字段顺序优化大小

```go
type Optimized struct {
    b int64  // 8 bytes，偏移 0
    a bool   // 1 byte，偏移 8
    c bool   // 1 byte，偏移 9
    // padding 6 bytes
} // Optimized 大小 = 16 bytes（比 B 小 8 bytes）
```

### 对齐系数

| 平台 | 最大对齐字节 |
|------|------------|
| 32位 | 8 |
| 64位 | 8 |

### 面试追问

**Q：为什么要有内存对齐？**
> CPU 访问内存以字（word）为单位，如果数据跨字边界，CPU 需要分两次访问。内存对齐让每次访问都是对齐的，减少访存次数，提升性能。代价是浪费一些 padding 空间。

**Q：如何减少结构体大小？**
> 按从大到小排列字段：8字节字段 → 4字节字段 → 1字节字段。减少内部 padding。

**Q：Go 结构体可以指定对齐吗？**
> 通过 `go:align` 注释（编译器内部使用），但一般不推荐。可以使用 `unsafe.Alignof()` 检查对齐系数。
