[🏠 首页](../../../README.md) · [📦 Linux / 操作系统基础](../README.md)

---

# 进程间通信：管道、消息队列、共享内存、信号量与 Socket

> 考察频率：★★★☆☆  优先级：P1
> 关键词：IPC 分类、每种方式的特点、Go 中的对应实现

---

## 面试官考察意图

这道题在面试中出现频率不高，但一旦出现就是在考察候选人的**系统编程底蕴**。
初级只知道"进程间通信有管道、消息队列"，高级要能讲清楚**每种 IPC 机制的适用场景、与 Go 对应实现的映射关系、以及如何选择最合适的 IPC 方式**。

---

## 核心答案（30 秒版）

| IPC 方式 | 特点 | Go 对应 |
|---------|------|---------|
| **管道（Pipe）** | 面向字节流、半双工、有亲缘关系 | `os.Pipe()` |
| **消息队列（Message Queue）** | 面向消息、异步、无亲缘关系 | `message queue`（SysV）|
| **共享内存（Shared Memory）** | 最快（零拷贝）、需同步 | `mmap` + `sync.Mutex` |
| **信号量（Semaphore）** | 计数器、用于进程同步 | `sync.Semaphore`（SysV）|
| **Socket** | 面向字节流、可无亲缘关系、最通用 | `net.Conn` |

**选择原则：**
- 同机同进程：**channel / sync.Mutex**（Go 原生）
- 同机跨进程：**mmap + 信号量** 或 **Unix Domain Socket**
- 夸机器：**TCP Socket / HTTP / gRPC**

---

## 深度展开

### 1. 管道（Pipe）

#### 1.1 匿名管道

```bash
# Shell 中的管道（半双工）
cat /var/log/app.log | grep ERROR | head -20

# 程序中创建匿名管道
# Go 对应 os.Pipe()
r, w, err := os.Pipe()
```

**特点：**
- 半双工，数据从写端流入，从读端流出
- 只能在有亲缘关系（父子进程）的进程间使用
- 面向字节流，无消息边界
- 读端阻塞直到有数据，写端满时阻塞（管道容量通常 64KB）

#### 1.2 命名管道（FIFO）

```bash
# 创建命名管道
mkfifo /tmp/my_fifo

# 两个无关进程可以通过这个 FIFO 通信
# 进程 A
echo "hello" > /tmp/my_fifo

# 进程 B
cat /tmp/my_fifo
```

**Go 中使用 FIFO：**
```go
fifo, err := os.OpenFile("/tmp/my_fifo", os.O_RDONLY, 0666)
defer fifo.Close()
data := make([]byte, 1024)
n, _ := fifo.Read(data)
fmt.Println(string(data[:n]))
```

#### 1.3 Go 的 io.Pipe

```go
// Go 提供的 io.Pipe() 用于内存中的管道
// 常用于将 Reader 的输出直接连接到 Writer

pr, pw := io.Pipe()

go func() {
    defer pw.Close()
    // 将数据写入管道的写端
    pw.Write([]byte("hello from pipe"))
}()

buf := make([]byte, 1024)
n, _ := pr.Read(buf)
fmt.Println(string(buf[:n])) // "hello from pipe"
```

---

### 2. 消息队列（Message Queue）

#### 2.1 SysV 消息队列

```bash
# 创建消息队列
ipcmk -Q

# 查看
ipcs -q

# 删除
ipcrm -q <msgid>
```

**消息队列特点：**
- 面向消息（有边界），每条消息有类型和正文
- 异步，发送方不阻塞
- 可以被多个进程读写（无亲缘关系）
- 队列容量受系统限制

#### 2.2 Go 中的消息队列

Go 标准库不直接支持 SysV 消息队列，通常用第三方库：

```go
// 使用 go-msq MQ 库（封装了 SysV 消息队列）
import "github.com/ActiveState/tail/blob/master/src/github.com/ActiveState/tail/winapi/child_process.go"

// 更常见的是使用外部消息队列中间件：
// - NATS（Go 实现的轻量级消息队列）
// - RabbitMQ
// - Kafka
```

**为什么 Go 不直接支持 SysV 消息队列？**
消息队列是 IPC 中较慢的方式，且 Go 的 channel 已经提供了进程内的消息传递能力。跨进程通常用更成熟的中间件。

---

### 3. 共享内存（Shared Memory）

#### 3.1 原理与优势

共享内存是**最快的 IPC 方式**：
- 无数据拷贝（直接映射到进程地址空间）
- 内核只负责同步访问，不参与数据传输
- 比 pipe/socket 快 10 倍以上

```
进程 A                    进程 B
[----共享内存区----] <---> [----共享内存区----]
     ↕ 操作系统同步             ↕ 操作系统同步
```

#### 3.2 mmap

```bash
# 命令行创建共享内存映射
# 将文件映射到内存
mmap -f /tmp/shared_file

# 创建匿名共享内存（进程 fork 后共享）
mmap -m --size=4096 /tmp/shm
```

#### 3.3 Go 中使用 mmap

```go
// 使用 mmap 进行文件映射
file, err := os.OpenFile("/tmp/data.bin", os.O_RDWR|os.O_CREATE, 0644)
if err != nil {
    return err
}
defer file.Close()

// mmap 文件
mmapFile, err := syscall.Mmap(
    int(file.Fd()),
    0,              // offset
    4096,           // length
    syscall.PROT_READ|syscall.PROT_WRITE,
    syscall.MAP_SHARED,
)
if err != nil {
    return err
}
defer syscall.Munmap(mmapFile)

// 使用 mmapFile 就像使用普通 slice
mmapFile[0] = 42
fmt.Println(mmapFile[0]) // 42
```

**Go mmap 库推荐：**
```go
import "github.com/torvalds/linux/blob/master/mm/mmap.go"
// 或使用标准库 syscall（已演示）
```

#### 3.4 共享内存 + 同步

共享内存本身不提供同步，需要配合：
- **Mutex（进程间）**：需要用 `syscall.Mutex` 或文件锁
- **信号量**：用于计数和同步
- **Go 的 sync.Map**：进程内共享（不跨进程）

```go
// 进程间共享内存 + 文件锁
import "golang.org/x/sys/unix"

func shmWithLock(file *os.File) {
    flock := unix.Flock_t{
        Type: unix.F_WRLCK,
        Whence: 0,
        Start: 0,
        Len:   0, // 整个文件
    }
    unix.FcntlFlock(uintptr(file.Fd()), unix.F_SETLK, &flock)
    // ... 临界区操作
    unix.FcntlFlock(uintptr(file.Fd()), unix.F_UNLCK, &flock)
}
```

**典型应用场景：**
- 进程间大块数据交换（mmap 比 pipe 快很多）
- 数据库缓存（Redis fork 后的共享内存）
- 多进程日志写入同一文件

---

### 4. 信号量（Semaphore）

#### 4.1 作用

信号量用于**进程间同步**，不是数据传输：
- 一个计数器，控制对共享资源的访问
- P 操作（-1）：获取资源
- V 操作（+1）：释放资源
- 为负时阻塞（资源不足）

#### 4.2 SysV 信号量

```bash
# 创建信号量
ipcmk -S

# 查看
ipcs -s

# 操作用 semtool（需安装）
```

#### 4.3 Go 中的信号量

Go sync 包中的 `Semaphore` 用于 goroutine 间的限流，不用于跨进程：

```go
// Go 中的信号量：runtime/Semaphore（内部使用）
import "golang.org/x/sync/semaphore"

sem := semaphore.NewWeighted(3) // 同时最多 3 个 goroutine

for _, task := range tasks {
    sem.Acquire(context.Background(), 1)
    go func() {
        defer sem.Release(1)
        // 处理任务
    }()
}
```

**跨进程信号量需要 SysV 接口（不推荐，Go 中很少使用）：**
- 优先用 `os.File` + `fcntl(F_SETLK)` 文件锁
- 或 Unix Domain Socket

---

### 5. Socket（Unix Domain Socket）

#### 5.1 为什么最重要

Socket 是**最通用的 IPC 方式**，唯一支持跨机器通信：
- 字节流（TCP）或数据报（UDP）
- 无亲缘关系进程可通信
- Unix Domain Socket 比 TCP Loopback 快（同一机器）

#### 5.2 Unix Domain Socket

```bash
# 查看 Unix Domain Socket
ss -x

# 文件系统中的 socket 文件
ls -la /var/run/docker.sock
```

#### 5.3 Go 中的 Unix Domain Socket

```go
// 服务端
listener, err := net.Listen("unix", "/tmp/app.sock")
defer os.Remove("/tmp/app.sock")
conn, _ := listener.Accept()
defer conn.Close()

// 客户端
conn, err := net.Dial("unix", "/tmp/app.sock")
defer conn.Close()
conn.Write([]byte("hello"))
```

**Go gRPC 常用 Unix Socket 做同机通信：**
```go
// gRPC Unix Socket 连接（避免 TCP 栈开销）
conn, err := grpc.Dial(
    "unix:///var/run/grpc.sock",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

---

### 6. 横向对比

| IPC 方式 | 速度 | 跨进程 | 跨机器 | 数据类型 | Go 支持度 |
|---------|------|--------|--------|----------|----------|
| 匿名管道 | 快 | ❌（需亲缘）| ❌ | 字节流 | ⭐⭐⭐ `os.Pipe` |
| 命名管道 | 快 | ✅ | ❌ | 字节流 | ⭐⭐ `os.OpenFile` |
| 消息队列 | 中等 | ✅ | ❌ | 消息 | ⭐ 第三方库 |
| 共享内存 | 最快 | ✅ | ❌ | 任意 | ⭐⭐ `syscall.Mmap` |
| 信号量 | 快 | ✅ | ❌ | 计数器 | ⭐ `syscall` |
| TCP Socket | 快 | ✅ | ✅ | 字节流 | ⭐⭐⭐ `net` |
| Unix Socket | 最快 | ✅ | ❌（同机）| 字节流 | ⭐⭐⭐ `net.Listen` |

**Go 工程师的选择建议：**
1. **同进程内并发**：`goroutine` + `channel` + `sync.Mutex`
2. **跨进程（同一机器）**：优先 **Unix Domain Socket**（简单、稳定）
3. **大块数据共享**：**mmap** + 文件锁
4. **跨机器**：TCP/HTTP/gRPC（外部中间件）

---

## 高频追问

**Q：Go 的 channel 和 IPC 有什么关系？**
A：Go channel 是**进程内**（同进程内 goroutine 间）的消息传递机制，本质是内存共享 + 同步。channel 不是传统 IPC，因为 IPC 定义的是进程间通信，channel 是协程间通信。跨进程通信请用 Socket 或 mmap。

**Q：共享内存为什么最快？**
A：因为零拷贝。数据直接从进程 A 的地址空间映射到进程 B，无需经过内核中转（管道/Socket 都要经过内核缓冲区）。代价是需要应用层自己实现同步（如文件锁）。

**Q：mmap 和 shared memory 是一个东西吗？**
A：mmap 是创建共享内存的手段之一。mmap 将文件或匿名内存映射到进程地址空间；如果映射的是匿名内存（`MAP_ANON`），就是共享内存。Linux 中 mmap 底层调用的就是共享内存机制。

---

## 延伸阅读

- Linux man: `man 7 pipe`, `man 7 mq`, `man 7 shm`, `man 7 sem`
- Go syscall 包文档：`godoc.org/syscall`
- 《Unix 环境高级编程》第 15 章"进程间通信"
