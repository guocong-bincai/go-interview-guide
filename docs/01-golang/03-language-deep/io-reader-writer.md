[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [💎 语言机制](../README.md)

---

# io.Reader / io.Writer：Go I/O 抽象层设计与流式处理

> 考察频率：★★★☆☆  优先级：P1（高频面试点）
> 关键词：io.Reader、io.Writer、ReadFull、 TeeReader、LimitReader、流式处理、buffered I/O、Pipeline

---

## 面试官考察意图

考察候选人对 Go I/O 抽象设计哲学的理解。
初级只知道"Reader 是读接口，Writer 是写接口"，高级要能讲清楚 **Go I/O 设计哲学（接口最小化 + 组合优于继承）、常见 Reader/Writer 实现、以及在 HTTP、文件、网络协议等场景中的流式处理优势**。

---

## 核心答案（30 秒版）

Go 的 I/O 核心是两个极简接口：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

**设计哲学：一切皆可流式处理**——不需要一次性把数据全部加载到内存，通过 `[]byte` 缓冲区逐段读写，适合大文件、网络流等内存敏感场景。

常见组合用法：
- `io.ReadAll` → 读取全部到内存（简单但不适合大文件）
- `io.Copy(dst, src)` → 流式复制（适合大文件）
- `bufio.NewReader/Writer` → 带缓冲的流式读写（减少系统调用）

---

## 深度展开

### 1. io.Reader 核心语义

`Read(p []byte) (n int, err error)` 的精确语义是面试高频坑：

```go
// 语义1：Read 必须尽量填满 p，除非遇到 EOF 或错误
// 解读：只要有数据，就应该填满 p 的可用空间
func ReadAtLeast(r io.Reader, p []byte, min int) (n int, err error) {
    if len(p) < min {
        return 0, errors.New("buf too short")
    }
    for n < min && err == nil {
        var nn int
        nn, err = r.Read(p[n:])  // 从 p[n:] 开始，填满为止
        n += nn
    }
    return
}

// 语义2：Read 返回的 err 不为 nil 时，n 可能 > 0（部分成功）
// 解读：即使出错，也已经读取了 n 个字节，调用者需要处理这些数据
data := make([]byte, 100)
n, err := r.Read(data)
if n > 0 {
    process(data[:n])  // 处理已读取的部分，即使 err != nil
}

// 语义3：读到 EOF 时，err = io.EOF（不一定有 io.EOF）
// 解读：最后一次 Read 可能返回 (n, io.EOF)，n > 0 有效
for {
    n, err := r.Read(buf)
    if n > 0 {
        fmt.Println("read:", buf[:n])
    }
    if err == io.EOF {
        break
    }
    if err != nil {
        fmt.Println("error:", err)
        break
    }
}
```

### 2. 常见 Reader 实现

#### 2.1 strings.Reader / bytes.Reader

```go
// strings.Reader：字符串 → 可读
r := strings.NewReader("hello world")
data := make([]byte, 5)
n, _ := r.Read(data)  // n=5, data="hello"

// bytes.Reader：[]byte → 可读，支持 Seek
r := bytes.NewReader([]byte("hello"))
r.Seek(3, io.SeekStart)
data := make([]byte, 3)
r.Read(data)  // "lo "
```

#### 2.2 bufio.Reader（缓冲读）

```go
// bufio.NewReader 包装底层 Reader，减少系统调用次数
// 默认缓冲区：4096 字节
r := bufio.NewReader(os.Stdin)

// 按行读取（常用）
for {
    line, err := r.ReadString('\n')
    if err == io.EOF {
        break
    }
    fmt.Print(line)
}
```

#### 2.3 limitReader：限制读取量

```go
// io.LimitReader 限制最多读取 n 个字节
r := io.LimitReader(strings.NewReader("hello world"), 5)
data, _ := io.ReadAll(r)  // "hello"（最多 5 字节）
```

### 3. 常见 Writer 实现

#### 3.1 os.Stdout / os.Stderr

```go
// 直接写标准输出
io.WriteString(os.Stdout, "hello\n")

// 错误输出
fmt.Fprintf(os.Stderr, "error: %v\n", err)
```

#### 3.2 bufio.Writer（缓冲写）

```go
// 减少 write 系统调用，批量刷盘
w := bufio.NewWriter(os.Stdout)
for i := 0; i < 1000; i++ {
    fmt.Fprintf(w, "line %d\n", i)
    // 不立即写盘，积累在缓冲区
}
w.Flush()  // 强制刷盘
```

#### 3.3 multiWriter：同时写入多个目标

```go
// io.MultiWriter 将一份数据同时写入多个 Writer
file, _ := os.Create("output.txt")
defer file.Close()

mw := io.MultiWriter(os.Stdout, file)  // 同时打印到屏幕和文件
mw.Write([]byte("hello\n"))  // 两处都收到
```

### 4. 经典组合工具函数

#### 4.1 io.Copy：流式复制

```go
// io.Copy(dst, src) 等价于：
//   var buf [32 * 1024]byte
//   for { n, _ := src.Read(buf[:]); dst.Write(buf[:n]) }

// ✅ 适合大文件：内存占用恒定（32KB buffer）
func copyLargeFile(src, dst string) error {
    srcFile, err := os.Open(src)
    if err != nil {
        return err
    }
    defer srcFile.Close()

    dstFile, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer dstFile.Close()

    _, err = io.Copy(dstFile, srcFile)  // 不会把整个文件加载到内存
    return err
}
```

#### 4.2 io.ReadAll vs 流式读取

```go
// ❌ io.ReadAll：不适合大文件（可能 OOM）
data, err := io.ReadAll(file)  // 把整个文件读入内存
// 1GB 文件 → 1GB 内存占用

// ✅ io.Copy：流式处理，内存占用恒定
_, err = io.Copy(dst, src)  // 32KB buffer 循环使用
```

#### 4.3 io.TeeReader：同时消费和传递

```go
// TeeReader：在读取的同时把数据复制到另一个 Writer
// 常用于"边读边写"的场景（如 HTTP 请求日志）
var buf bytes.Buffer
tee := io.TeeReader(
    strings.NewReader("hello world"),  // 数据源
    &buf,                              // 同时写入 buf
)
io.Copy(os.Stdout, tee)  // 输出到 stdout，同时 buf 里也有数据
fmt.Println("tee saved:", buf.String())  // "hello world"
```

#### 4.4 PipeReader / PipeWriter：内存中的管道

```go
// io.Pipe：同步内存管道，写端写入的数据立即被读端读取
r, w := io.Pipe()

go func() {
    fmt.Fprint(w, "data from writer")
    w.Close()
}()

data, _ := io.ReadAll(r)
fmt.Println(string(data))  // "data from writer"
```

### 5. HTTP 中的流式处理

#### 5.1 请求体流式读取

```go
// ✅ HTTP 请求体的流式处理
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // r.Body 是一个 io.ReadCloser，可以流式读取
    // 不要用 io.ReadAll(r.Body)，大文件会导致 OOM
    buf := bufio.NewReader(r.Body)
    for {
        line, err := buf.ReadString('\n')
        if err == io.EOF {
            break
        }
        processLine(line)
    }
}
```

#### 5.1 响应体流式写入（Server-Sent Events）

```go
// HTTP 响应的流式写入（适合长连接、SSE）
func handleSSE(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming unsupported", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    for {
        fmt.Fprintf(w, "data: %s\n\n", time.Now().Format("15:04:05"))
        flusher.Flush()  // 立即推送给客户端
        time.Sleep(1 * time.Second)
    }
}
```

### 6. 设计模式：Decorator（修饰器模式）

Go I/O 的精髓在于**层层包装**：

```go
// gzip 压缩 + bufio 缓冲 + 文件读写，全部通过接口组合
func compressedBufferedFile(filename string) (io.Reader, error) {
    file, err := os.Open(filename)
    if err != nil {
        return nil, err
    }

    // 层层包装：文件 → gzip → bufio
    gzReader, err := gzip.NewReader(file)
    if err != nil {
        file.Close()
        return nil, err
    }

    bufReader := bufio.NewReaderSize(gzReader, 64*1024)  // 64KB 缓冲

    // 返回装饰后的 Reader，调用者无需知道内部实现
    return &readerCloser{bufReader, []io.Closer{file, gzReader}}, nil
}

type readerCloser struct {
    io.Reader
    closers []io.Closer
}

func (rc *readerCloser) Close() error {
    for _, c := range rc.closers {
        c.Close()
    }
    return nil
}
```

---

## 高频追问

**Q：io.Copy 怎么做到内存占用恒定的？**
> 内部固定分配 `buf := make([]byte, 32*1024)`，循环使用这个 buffer，每次 Read 最多读 32KB。`io.Copy(dst, src) = for { n := src.Read(buf); if n == 0 { break }; dst.Write(buf[:n]) }`，内存峰值始终是 32KB。

**Q：bufio.Reader 的缓冲区大小怎么选？**
> 默认 4096 字节。文件/网络 I/O 场景通常 8KB~64KB 效果最好，过大浪费内存，过小增加系统调用。可以用 `bufio.NewReaderSize(r, size)` 手动指定。

**Q：HTTP 请求 body 读完还能再读吗？**
> 不能。`r.Body` 是流式，一旦读出数据就消费掉了。如果需要多次读取，用 `io.NopCloser(bytes.NewBuffer(bodyBytes))` 包装 bytes.Buffer 重新创建一个可重复读取的 Body。

**Q：Write 返回的 n 和 len(p) 不相等怎么处理？**
> 必须处理！Write 可能只写入了部分数据，需要循环重试：
```go
n, err := w.Write(p)
if err != nil {
    if n < len(p) {
        // 部分写入，需要从 p[n:] 继续写
        _, err = w.Write(p[n:])
    }
    return err
}
```

---

**[← 上一篇：Swiss Tables](./11-go1.24-swiss-tables.md)** · **[目录](../README.md)** · **[下一篇：Go 1.24 Weak 包](./16-go1.24-weak-package.md)**
