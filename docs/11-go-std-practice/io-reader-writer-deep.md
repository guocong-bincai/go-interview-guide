[🏠 首页](../../../README.md) · [📦 Go 标准库与工程实践](../README.md)

---

# io.Reader/Writer 设计模式与深度解析

## 面试官考察意图

考察候选人对 Go I/O 抽象的理解深度。
初级只能说出"Reader 读数据，Writer 写数据"，高级要能讲清楚 **接口设计哲学、装饰器模式、流水线处理、buffered I/O 性能优化、以及常见误用场景**，并能结合生产问题给出分析。

---

## 核心答案（30 秒版）

Go 的 `io.Reader/Writer` 是**最小接口设计**的典范，通过 `Reader`/`Writer`/`Closer`/`Seeker` 等原子接口的组合，构建出强大的 I/O 流水线。核心哲学是：**一切皆流，组合优于继承**。

| 接口 | 签名 | 作用 |
|------|------|------|
| `Reader` | `Read(p []byte) (n int, err error)` | 从数据源读取 |
| `Writer` | `Write(p []byte) (n int, err error)` | 写入目标 |
| `Closer` | `Close() error` | 释放资源 |
| `Seeker` | `Seek(offset int64, whence int) (int64, error)` | 移动读写位置 |

---

## 深度展开

### 1. 接口设计哲学

#### 1.1 最小接口原则（Interface Segregation）

```go
// ❌ 错误：把 Reader 和 Writer 合并成一个接口
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}

// ✅ 正确：拆分为独立接口，按需组合
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Writer interface {
    Write(p []byte) (n int, err error)
}

// 按需组合
type ReadWriter interface {
    Reader
    Writer
}
```

**为什么 Go 这样设计？**
- 函数签名只需要接收它需要的接口，而不是附带额外方法
- 任何实现了 `Read()` 的类型都可以作为 `Reader` 使用，无需显式声明"实现"接口

#### 1.2 装饰器模式（Decorator Pattern）

```
io.Reader 体系的核心用法：

os.File (实现了 Reader) 
    ↓
bufio.NewReader (包装，添加缓冲) 
    ↓
compress.NewReader (包装，添加压缩) 
    ↓
crypto.NewReader (包装，添加加密) 
    ↓
自定义 Transform 逻辑
```

### 2. 核心接口详解

#### 2.1 Reader：读取数据的契约

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

**契约理解：**

```
Read 行为规范：
1. 读取到 len(p) 字节 → 返回 n = len(p), err = nil
2. 数据源耗尽 → 返回 n < len(p), err = EOF
3. 出错 → 返回 n（已读取的数据）+ 错误
```

**关键点：Read 不保证填满 buffer**

```go
// ❌ 常见误用：假设 Read 总是填满 buffer
buf := make([]byte, 1024)
n, _ := r.Read(buf)  // 实际上可能只读了 10 字节
process(buf[:n])     // 只处理前 n 字节

// ✅ 正确写法：显式处理实际读取的字节数
for {
    n, err := r.Read(buf)
    if n > 0 {
        process(buf[:n])
    }
    if err == io.EOF {
        break
    }
    if err != nil {
        // 处理错误
    }
}
```

#### 2.2 Writer：写入数据的契约

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

**契约理解：**

```
Write 行为规范：
1. 全部写入 → 返回 n = len(p), err = nil
2. 部分写入 → 返回 n（已写入），err = nil（需重试剩余）
3. 出错 → 返回 n（已写入）+ 错误
```

**关键点：Write 可能部分写入，需循环重试**

```go
// ✅ 写入完整的正确姿势
func writeAll(w io.Writer, data []byte) error {
    n := 0
    for n < len(data) {
        written, err := w.Write(data[n:])
        if err != nil {
            return err
        }
        n += written
    }
    return nil
}
```

#### 2.3 Closer：资源释放

```go
type Closer interface {
    Close() error
}
```

**最佳实践：defer 关闭**

```go
// ✅ 正确：defer 确保关闭
f, err := os.Open("data.txt")
if err != nil {
    return err
}
defer f.Close()

// ✅ 网络请求的关闭模式
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
```

### 3. 组合接口

#### 3.1 ReadWriteCloser

```go
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

#### 3.2 ReaderFrom / WriterTo（高效批量 I/O）

```go
// WriterTo：如果类型实现了 WriteTo，直接用高效方法
type WriterTo interface {
    WriteTo(w Writer) (n int64, err error)
}

// ReaderFrom：如果类型实现了 ReadFrom，直接用高效方法
type ReaderFrom interface {
    ReadFrom(r Reader) (n int64, err error)
}
```

**高效拷贝示例：**

```go
// 低效：Buffer → File → Buffer
buf := new(bytes.Buffer)
buf.ReadFrom(file)  // 多次拷贝

// 高效：File → Buffer（实现了 ReadFrom）
file.ReadFrom(buf)   // 直接写入 buf 底层数组
```

#### 3.3 Seeker：随机访问

```go
type Seeker interface {
    Seek(offset int64, whence int) (int64, error)
}

// whence 参数：
const (
    SeekStart   = 0  // 相对于文件开头
    SeekCurrent = 1  // 相对于当前位置
    SeekEnd     = 2  // 相对于文件结尾
)
```

### 4. 实用工具函数

#### 4.1 io.Copy：通用拷贝

```go
// 内部实现（简化）
func Copy(dst Writer, src Reader) (written int64, err error) {
    return CopyBuffer(dst, src, nil)
}

// 等价于
buf := make([]byte, 32*1024)  // 32KB buffer
io.CopyBuffer(dst, src, buf)
```

**使用示例：**

```go
// 复制文件
src, _ := os.Open("src.txt")
dst, _ := os.Create("dst.txt")
defer src.Close()
defer dst.Close()
io.Copy(dst, src)

// 复制到 HTTP 响应
file, _ := os.Open("large.bin")
defer file.Close()
io.Copy(w, file)  // w 是 http.ResponseWriter

// 复制到标准输出
file, _ := os.Open("log.txt")
defer file.Close()
io.Copy(os.Stdout, file)
```

#### 4.2 io.ReadAll：读取全部

```go
// 内部实现
func ReadAll(r Reader) ([]byte, error) {
    return readAll(r, make([]byte, 0, 4096))  // 容量从 4KB 开始，按 2 倍扩容
}

// 注意：对于大文件慎用，可能导致内存爆炸
// ✅ 大文件应该用 streaming
// ❌ 禁止在大文件场景使用
data, _ := io.ReadAll(largeFile)  // 可能 OOM
```

#### 4.3 io.MultiReader：合并多个 Reader

```go
// 将多个 Reader 合并为一个逻辑 Reader
r1 := strings.NewReader("Hello ")
r2 := strings.NewReader("World ")
r3 := strings.NewReader("!")

mr := io.MultiReader(r1, r2, r3)
data, _ := io.ReadAll(mr)  // "Hello World !"
```

**应用场景：**

```go
// 拼接多个数据源
reader := io.MultiReader(
    bytes.NewReader(header),
    file,
    bytes.NewReader(footer),
)
io.Copy(w, reader)
```

#### 4.4 io.MultiWriter：同时写入多个目标

```go
// 同时写入多个 Writer
f, _ := os.Create("log.txt")
var buf bytes.Buffer

mw := io.MultiWriter(f, &buf)  // 同时写入文件和内存 buffer
mw.Write([]byte("log entry"))   // 两处都写入
```

**应用场景：**

```go
// 同时记录日志和发送监控
mw := io.MultiWriter(
    os.Stdout,
    logFile,
    metricsWriter,
)
io.Copy(mw, body)
```

#### 4.5 io.Pipe：管道（同步内存 I/O）

```go
// 创建内存管道，一端写一端读，阻塞式传递数据
r, w := io.Pipe()

// goroutine 1: 写入（阻塞直到有人读）
go func() {
    w.Write([]byte("data"))
    w.Close()
}()

// goroutine 2: 读取
io.Copy(os.Stdout, r)  // 输出 "data"
```

**应用场景：**

```go
// 动态构建请求体（先处理再发送）
pipeReader, pipeWriter := io.Pipe()

go func() {
    // 压缩写入（无需先全量加载到内存）
    gw := gzip.NewWriter(pipeWriter)
    json.NewEncoder(gw).Encode(data)
    gw.Close()
    pipeWriter.Close()
}()

// 发送请求，体是压缩后的流
req, _ := http.NewRequest("POST", url, pipeReader)
req.Header.Set("Content-Encoding", "gzip")
http.DefaultClient.Do(req)
```

### 5. bufio 缓冲 I/O

#### 5.1 Buffered Reader

```go
// 减少系统调用次数，提升读取性能
br := bufio.NewReader(file)
data, _ := br.ReadBytes('\n')   // 按行读取，内部有多次 Read
line, _ := br.ReadString('\n')  // 同上
bytes, _ := br.ReadBytes('\n')

// 按词读取（空格分隔）
sc := bufio.NewScanner(br)
sc.Split(bufio.ScanWords)
for sc.Scan() {
    word := sc.Text()
}
// 默认 token 大小 64KB，可通过 sc.Buffer 调整
```

#### 5.2 Buffered Writer

```go
// 减少系统调用次数
bw := bufio.NewWriter(file)
for _, s := range lines {
    bw.WriteString(s)  // 先写入 buffer
    bw.WriteString("\n")
}
bw.Flush()  // 必须 flush，否则数据丢失

// NewReaderSize / NewWriterSize：自定义 buffer 大小
// 大文件可用 64KB 或更大
```

#### 5.3 性能对比

| 方式 | 系统调用 | 适用场景 |
|------|---------|---------|
| `file.Read()` | 每字节一次 | 小文件 |
| `bufio.Reader` | 每 buffer 一次 | 大文件、流式处理 |
| `io.Copy` | 按 buffer 次数 | 通用拷贝 |

### 6. 自定义 Reader 实现：流水线处理

#### 6.1 示例：分词 Reader

```go
type WordCounter struct {
    r         io.Reader
    delimiter byte
    count     int
}

func NewWordCounter(r io.Reader, delimiter byte) *WordCounter {
    return &WordCounter{r: r, delimiter: delimiter}
}

func (wc *WordCounter) Read(p []byte) (int, error) {
    n, err := wc.r.Read(p)
    for _, b := range p[:n] {
        if b == wc.delimiter {
            wc.count++
        }
    }
    return n, err
}

// 使用
wc := NewWordCounter(file, ' ')
io.Copy(os.Stdout, wc)
fmt.Printf("Word count: %d\n", wc.count)
```

#### 6.2 TeeReader：同时读取和记录

```go
// 读取数据的同时写入到另一个 Writer（用于日志/调试）
var buf bytes.Buffer
tee := io.TeeReader(originalReader, &buf)

// 读取的数据会被同时写入 buf
data, _ := io.ReadAll(tee)
// 此时 buf 包含了已读取的数据，可用于调试或记录
```

#### 6.3 LimitReader：限制读取长度

```go
// 只读取前 N 字节
lr := io.LimitReader(r, 1024)  // 最多读取 1KB
data, _ := io.ReadAll(lr)
```

### 7. 生产问题与最佳实践

#### 7.1 问题：未关闭资源导致文件句柄泄漏

```go
// ❌ 错误：没有 defer close
f, err := os.Open("big.txt")
data := make([]byte, 1024)
f.Read(data)  // 如果中途出错，f 永远不关闭

// ✅ 正确
f, err := os.Open("big.txt")
if err != nil {
    return err
}
defer f.Close()
data := make([]byte, 1024)
_, err = f.Read(data)
```

#### 7.2 问题：大文件读取导致 OOM

```go
// ❌ 错误：把大文件全部读入内存
data, _ := os.ReadFile("large.bin")  // 10GB 文件 → OOM

// ✅ 正确：流式处理
f, _ := os.Open("large.bin")
defer f.Close()
reader := bufio.NewReaderSize(f, 64*1024)  // 64KB buffer
for {
    buf := make([]byte, 64*1024)
    n, err := reader.Read(buf)
    if n > 0 {
        process(buf[:n])
    }
    if err == io.EOF {
        break
    }
    if err != nil {
        return err
    }
}
```

#### 7.3 问题：混淆 io.ReadFull 和 io.ReadAll

```go
// io.ReadFull：必须填满 buffer 或返回错误
buf := make([]byte, 10)
n, err := io.ReadFull(r, buf)  // 读不够 10 字节 → err
// 适用于：协议头、长度前缀已知的数据

// io.ReadAll：读直到 EOF，可能很大
data, _ := io.ReadAll(r)  // 无上限，看数据源有多大
// 适用于：小数据、测试数据
```

#### 7.4 性能优化：原地操作减少分配

```go
// ❌ 每次读取都分配新 buffer
for {
    buf := make([]byte, 4096)  // 每次循环分配
    n, _ := r.Read(buf)
    if n == 0 { break }
    process(buf[:n])
}

// ✅ 复用 buffer（goroutine 安全，需注意 buffer 生命周期）
buf := make([]byte, 4096)
for {
    n, _ := r.Read(buf)
    if n == 0 { break }
    process(buf[:n])
}

// ✅ 最优：sync.Pool 减少分配压力
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}
for {
    buf := bufPool.Get().([]byte)
    n, _ := r.Read(buf)
    if n == 0 {
        bufPool.Put(buf)
        break
    }
    process(buf[:n])
    bufPool.Put(buf)
}
```

### 8. 时间标准库的 I/O

```go
// time.Duration.String() 写入 io.Writer
var buf bytes.Buffer
d := 5 * time.Second
n, _ := buf.WriteString(d.String())  // "5s"

// 解析时间字符串
t, err := time.Parse("2006-01-02", "2025-01-01")

// time.Time.AppendFormat
buf := new(strings.Builder)
t.AppendFormat(buf, "2006-01-02 15:04:05")
```

---

## 高频追问

**Q：io.Reader 的 Read 什么时候返回 EOF？**

- 数据源完全耗尽时返回 `io.EOF`
- `Read` 可能返回 `n > 0, err = nil` 后再次调用才返回 `io.EOF`
- 永远不要假设 `Read` 只调用一次就能读完所有数据

**Q：bufio.NewReader 和 io.MultiReader 的区别？**

- `bufio.NewReader`：给底层 Reader 添加缓冲，减少系统调用
- `io.MultiReader`：将多个 Reader 逻辑合并为一个顺序 Reader

**Q：如何实现一个带超时读取的 Reader？**

```go
type timeoutReader struct {
    r   io.Reader
    t   time.Duration
}

func (tr *timeoutReader) Read(p []byte) (int, error) {
    ch := make(chan error, 1)
    go func() {
        n, err := tr.r.Read(p)
        ch <- err
    }()
    
    select {
    case err := <-ch:
        return 0, err
    case <-time.After(tr.t):
        return 0, context.DeadlineExceeded
    }
}
```

**Q：io.Copy 和 io.ReadAll 哪个性能更好？**

- 大数据量：`io.Copy` 更好（循环使用固定 buffer，无额外分配）
- 小数据量：差别不大，`ReadAll` 更方便

---

## 延伸阅读

- [Go io 包文档](https://pkg.go.dev/io)
- [Go blog: The io.Reader interface](https://blog.golang.org/io)
- [Go blog: Pipelines](https://blog.golang.org/pipelines)（I/O 流水线模式）
- [Go io 包源码](https://github.com/golang/go/blob/master/src/io/io.go)

---

**[← 上一篇：Go 标准库实践](../README.md)** · **[返回目录](../README.md)**
