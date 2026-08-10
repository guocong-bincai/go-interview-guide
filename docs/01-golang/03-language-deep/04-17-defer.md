# Go defer 底层原理与高频面试题

> 考察频率：★★★★★  优先级：P0  
> 关键词：defer、return 赋值顺序、defer 栈、LIFO、panic、参数求值时机

---

## 面试官考察意图

考察候选人对 Go defer 语句的深层理解深度。
初级只知道"defer 是延迟执行"，高级要能讲清楚 **defer 的注册机制、return 执行顺序与返回值赋值的交互、defer 栈的 LIFO 特性、参数预计算、以及 defer 在 panic 恢复中的作用**，并能回答"defer 和 return 谁先执行"这类经典陷阱题。

---

## 核心答案（30 秒版）

Go defer 执行流程（以有名返回值函数为例）：

```
return value
    ↓
1️⃣ 先将返回值赋给一个局部变量（还没真正 return）
2️⃣ 再执行 defer（defer 可以修改这个局部变量）
3️⃣ 最后真正 return 出去（返回的是被 defer 修改后的值）
```

**三个关键规则**：
- defer 按**栈（LIFO）顺序**执行，最后注册的 defer 最先执行
- defer 注册时，**参数就预计算好了**（不是执行时）
- defer 可以**修改有名返回值**，但无法修改字面量返回值（除非用指针）

---

## 深度展开

### 1. defer 的本质：链表注册

Go 编译器将 defer 翻译为 runtime.deferproc，在函数入口处将 defer 函数注册到一个链表中。

```go
// 伪代码：defer 在编译后的等价形式
func foo() {
    // 编译器插入：注册 defer
    deferproc(func() { fmt.Println("first") })  // 后注册，先执行
    deferproc(func() { fmt.Println("second") }) // 先注册，后执行
    
    fmt.Println("main")
    // 编译器插入：函数退出前统一执行 defer 链表
    // return runtime.deferreturn()
}
```

**defer 链表是头插法**，所以最后注册的 defer 在链表头部，先执行。

### 2. return 与 defer 的执行顺序

#### 场景一：无名返回值（基本类型）

```go
func test1() int {
    var a = 1
    defer func() {
        a = 2  // 修改的是局部变量 a，不是返回值
    }()
    return a  // 返回 1，因为 return 时已经确定返回值
}
```

**执行顺序**：
```
return a
  ↓
① 返回值 = 1（局部变量 a 的值拷贝给返回值）
② defer 执行：a = 2（修改的是 a，不是返回值）
③ return 1
```

#### 场景二：有名返回值（命名的返回值）

```go
func test2() (a int) {
    a = 1
    defer func() {
        a = 2  // 修改的是有名返回值本身！
    }()
    return  // 返回 a，即 2
}
```

**执行顺序**：
```
return (无表达式，return 有名返回值)
  ↓
① defer 执行：a = 2（修改的是有名返回值）
② return a
```

**结论**：defer 能修改有名返回值，无法直接修改无名返回值（因为无名返回值是函数结束时的临时结果）。

#### 场景三：返回指针

```go
func test3() *int {
    a := 1
    defer func() {
        a = 2  // 修改 a，defer 执行后 a 变成 2
    }()
    return &a  // 返回的是 a 的地址，所以 defer 修改有效
}
```

### 3. defer 参数预计算

**关键规则**：defer 注册时就计算参数，而非执行时。

```go
func demo() {
    a := 1
    defer func(x int) {
        fmt.Println(x)  // 打印 1，不是 2
    }(a)  // ← 这里立即求值：a = 1
    
    a = 2
}
```

#### 常见陷阱：循环中捕获变量

```go
func wrong() {
    for i := 0; i < 3; i++ {
        defer fmt.Println(i)  // 打印：2, 1, 0（LIFO）但 i 值是？答案是 2, 1, 0
    }
}

func wrong2() {
    funcs := []func(){}
    for i := 0; i < 3; i++ {
        funcs = append(funcs, func() {
            fmt.Println(i)  // 打印：3, 3, 3（所有闭包共享同一个 i）
        })
    }
    for _, f := range funcs {
        f()  // 循环结束后 i=3
    }
}
```

**正确写法**：通过参数捕获

```go
func right() {
    funcs := []func(){}
    for i := 0; i < 3; i++ {
        v := i  // ← 每轮创建新变量
        funcs = append(funcs, func() {
            fmt.Println(v)  // 打印：0, 1, 2
        })
    }
    for _, f := range funcs {
        f()
    }
}
```

### 4. defer 与 panic 的交互

#### defer 在 panic 中的执行

```go
func main() {
    defer func() {
        fmt.Println("defer 1")  // panic 时仍然执行
    }()
    defer func() {
        fmt.Println("defer 2")  // 先执行（LIFO）
        // defer 中 recovered() 可以阻止 panic 向上传播
    }()
    panic("oops")
    fmt.Println("unreachable")
}
```

输出：
```
defer 2
defer 1
panic: oops
```

#### recover 的正确姿势

```go
func safe() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered:", r)
        }
    }()
    panic("oops")
}
```

**关键**：recover 必须在 defer 中直接调用，**不能**封装到另一个函数里（除非正确传递）。

```go
// 错误写法：recover 被吞掉
func wrong() {
    defer catch()  // catch() 执行 recover()，但无法恢复当前 goroutine 的 panic
    panic("oops")
}

func catch() {
    recover()  // 这里调用 recover() 无效，因为 panic 不会跨越函数边界传播
}

// 正确写法：defer 直接调用 recover
func right() {
    defer func() {
        recover()  // 直接在 defer 中调用，有效
    }()
    panic("oops")
}
```

### 5. defer 的性能开销

#### defer 的代价

- **每次 defer 调用**：约 30~40ns（调用 runtime.deferproc）
- **函数退出时执行 defer**：额外遍历 defer 链表的开销

#### 高性能场景下的替代方案

```go
// 场景：高频函数中避免 defer
func highFreq() {
    // 使用手动控制而非 defer
    f, _ := os.Open("/tmp/data")
    // ... 业务逻辑 ...
    f.Close()  // 直接调用，不依赖 defer
}

// 或者：使用 defer 但接受其开销（大部分场景可忽略）
func lowFreq() {
    f, _ := os.Open("/tmp/data")
    defer f.Close()  // defer 的开销在此场景下可忽略
}
```

### 6. 常见面试题解析

#### Q1：输出什么？

```go
func main() {
    fmt.Println(f())
}

func f() (n int) {
    defer func() { n++ }()
    return 5
}
```

**答案**：6  
**解析**：return 5 先赋给 n（n=5），defer 执行 n++（n=6），return n（返回6）。

#### Q2：输出什么？

```go
func main() {
    fmt.Println(f())
}

func f() int {
    n := 5
    defer func() { n++ }()
    return n
}
```

**答案**：5  
**解析**：return n 将 n 的值拷贝给返回值（5），defer 执行 n++ 是在修改局部变量 n，不影响返回值。

#### Q3：输出什么？

```go
func main() {
    fmt.Println(f())
}

func f() (n int) {
    for i := 0; i < 3; i++ {
        defer func() { n++ }()
    }
    return
}
```

**答案**：3  
**解析**：循环注册 3 个 defer，return 时按 LIFO 执行：第1个 n++、第2个 n++、第3个 n++，共执行3次，最终 n=3。

#### Q4：defer 可以关闭文件、锁资源吗？

可以，这是 defer 最常见的用途：

```go
func writeFile() error {
    f, err := os.Create("/tmp/out.txt")
    if err != nil {
        return err
    }
    defer f.Close()  // 函数退出时自动关闭
    
    mu.Lock()
    defer mu.Unlock() // 同一函数内 unlock 在 Lock 之后执行（逆序）
    // 业务逻辑
    return nil
}
```

**注意**：同一个函数中 defer unlock 在 defer lock 之后执行（LIFO），但这是正确的——因为 Go 的 defer 是栈，后进先出，unlock 最后注册最先执行，正好在 return 之前。

#### Q5：defer 和 go 语句混用的坑

```go
func main() {
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Println(i)  // 问题：闭包中 i 的值不确定
        }()
    }
    wg.Wait()
}
```

**问题**：闭包捕获的是 i 的引用而非值，循环结束后 i=3，所以可能打印 0, 1, 3, 3, 3 或其他混乱结果。

**正确写法**：
```go
for i := 0; i < 3; i++ {
    v := i  // 每轮创建新变量
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(v)  // 打印 0, 1, 2
    }()
}
```

---

## 高频追问

**Q：defer 和 return 的执行顺序是什么？**

> return 先将返回值赋给变量，然后执行 defer（defer 可以修改有名返回值），最后真正 return。对于无名返回值，defer 无法修改返回值，因为返回值已经在 return 那一刻确定了。

**Q：defer 能拦截 panic 吗？**

> 可以，但必须在 defer 函数中直接调用 `recover()`。`recover()` 的作用范围是调用它的 goroutine 的当前 panic，不会跨函数传播。所以要把 `recover()` 直接放在 defer 闭包里，而不是调用一个封装函数。

**Q：defer 在什么情况下会失效？**

> 1. goroutine 崩溃（Go runtime 会终止整个 goroutine，defer 仍然执行）；2. 进程退出（os.Exit 不会执行 defer）；3. 死锁导致 goroutine 无法退出（defer 永远没机会执行）。

**Q：如何在循环中正确使用 defer？**

> 每轮循环创建新变量捕获循环变量的值：`v := i; defer func() { ... }(v)`。或者在循环体内部直接 defer，这样每次循环都会注册一个新的 defer。

**Q：defer 的性能瓶颈在哪里？**

> 每调用一次 defer 有约 30~40ns 的运行时开销，主要是调用 runtime.deferproc。函数退出时遍历 defer 链表也有开销。对于高频调用（如每秒调用几万次的函数），可以用手动资源管理替代 defer。

---

## 延伸阅读

- [Go spec: Defer statements](https://go.dev/ref/spec#Defer_statements)
- [runtime.deferproc 源码](https://github.com/golang/go/blob/master/src/runtime/panic.go)
- [Defer, Panic, Recover - Go Blog](https://go.dev/blog/defer-panic-and-recover)