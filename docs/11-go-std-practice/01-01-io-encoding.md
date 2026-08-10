# Go 标准库生产实践：io / encoding / time / testing

> 考察频率：★★★☆☆  优先级：P1

## 面试官考察意图

考察候选人对 Go 标准库的掌握深度——不仅是"能用"，更要理解"为什么这样设计"以及"在生产环境中可能踩的坑"。
高级工程师在 5 年以上经验后，对标准库的理解深度会成为和其他候选人的关键区分点。面试官想看你是否踩过真实的坑、是否有系统性总结。

---

## 核心答案（30 秒版）

Go 标准库质量极高，但有三个高频踩坑点：
1. **json 序列化性能**：标准库用反射，线上高频调用时是 CPU 大户，sonic 是目前最优解
2. **time.Timer 泄漏**：只用 `time.After` 但不清理底层 Timer，是生产环境 goroutine 泄漏的常见原因（Go 1.23 已改善）
3. **io.Reader 组合**：理解 `io.Reader`/`io.Writer` 的组合模式是写出高效代码的基础

---

## 深度展开

## 1. io 包：Reader/Writer 组合模式

### 1.1 接口设计的哲学

```go
// Go 的接口设计哲学：最小化接口，只做一件事
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

**单方法接口是 Go 最强大的设计**：任何实现了 `Read` 的类型都可以和 `io.Reader` 兼容，形成无限组合。

### 1.2 常用组合工具

```go
import "io"

// TeeReader: 读的同时复制到另一个 writer（调试/日志场景）
r := strings.NewReader("hello world")
var buf bytes.Buffer
tee := io.TeeReader(r, &buf)
_, _ = io.ReadAll(tee)
fmt.Println(buf.String()) // "hello world"

// MultiWriter: 一个数据写入多个 writer（日志多端同步）
file, _ := os.Create("/tmp/log.txt")
http.ResponseWriter res; // 假设
mw := io.MultiWriter(file, os.Stdout)
io.WriteString(mw, "log entry") // 同时写入文件和 stdout

// Pipe: goroutine 之间的流式传输
r, w := io.Pipe()
go func() {
    defer w.Close()
    // 生产者逻辑
    fmt.Fprint(w, "data from goroutine")
}()
io.Copy(os.Stdout, r)

// LimitReader: 限制读取字节数（防止读取无限数据）
limited := io.LimitReader(r, 1024) // 最多读 1KB
```

### 1.3 bufio 的缓冲艺术

```go
// 无缓冲：每次 Read 调用一次系统调用
// 有缓冲：用 []byte 缓冲减少系统调用次数

// 场景：网络请求（高频小数据）→ 用 bufio
br := bufio.NewReaderSize(conn, 4096) // 自定义缓冲大小
line, _ := br.ReadString('\n')

// 场景：文件顺序读取大文件 → bufio.NewReader
f, _ := os.Open("/tmp/largefile.txt")
defer f.Close()
scanner := bufio.NewScanner(f) // 按行读取，超大行需 SetBuffer
scanner.Buffer(make([]byte, 0, 64*1024), 1024*1024) // 增大单行缓冲
for scanner.Scan() {
    fmt.Println(scanner.Text())
}

// 场景：写文件（批量写）→ bufio.NewWriter
fw, _ := os.Create("/tmp/output.txt")
bw := bufio.NewWriterSize(fw, 64*1024) // 64KB 缓冲
for _, s := range lines {
    bw.WriteString(s + "\n")
}
bw.Flush() // 重要！不 flush 数据还在缓冲里
```

### 1.4 零拷贝：io.WriterTo 接口

```go
// os.File 实现了 io.WriterTo，可以直接调用 sendfile 系统调用（零拷贝）
// 标准库 net/http 在发送静态文件时自动使用这个优化

// 手写一个高效的 FileServerHandler
type staticHandler struct {
    root string
}

func (h *staticHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    f, err := os.Open(h.root + r.URL.Path)
    if err != nil {
        http.NotFound(w, r)
        return
    }
    defer f.Close()
    
    // 如果 File 实现了 WriteTo，使用零拷贝
    if wt, ok := f.(io.WriterTo); ok {
        wt.WriteTo(w) // 直接调用底层 sendfile，避免用户态→内核态拷贝
        return
    }
    
    // 否则回退到普通 Copy
    io.Copy(w, f)
}
```

---

## 2. encoding/json：性能陷阱与替代方案

### 2.1 标准库为什么慢

```go
// 标准库 json 的核心问题：运行时反射
// 每次 marshal/unmarshal 都需要：
// 1. 遍历结构体字段（reflect.Type.FieldByIndex）
// 2. 调用接口方法（interface{} 间接调用）
// 3. 动态分配字符串（没有预计算偏移）

// benchmark 对比（来自 databricks/sonic 官方数据）
// 序列化一个 struct{ID int; Name string} 10000 次：
// 标准库：~1200ms
// sonic（JIT）：~120ms（10x 提升）
// json-iterator：~180ms（6x 提升）
```

### 2.2 json.Number：大整数精度陷阱

```go
// 小心：默认 unmarshal 会把大整数变成 float64，精度丢失
data := `{"order_id": 1234567890123456789}`
var v map[string]interface{}
json.Unmarshal([]byte(data), &v)
fmt.Println(v["order_id"]) // 1.2345678901234568e+18（精度丢失！）

// 修复：使用 json.Number 保留原始字符串
decoder := json.NewDecoder(bytes.NewReader([]byte(data)))
decoder.UseNumber()
decoder.Decode(&v)
num := v["order_id"].(json.Number)
fmt.Println(num.String()) // "1234567890123456789"（原始字符串）
```

### 2.3 自定义序列化

```go
type Money struct {
    int64 // 分（避免浮点精度问题）
}

func (m Money) MarshalJSON() ([]byte, error) {
    return []byte(fmt.Sprintf(`"%d.%02d"`, m.int64/100, m.int64%100)), nil
}

func (m *Money) UnmarshalJSON(data []byte) error {
    // 解析 "$123.45" 格式
    s := strings.Trim(string(data), `"`)
    parts := strings.Split(s, ".")
    if len(parts) != 2 {
        return errors.New("invalid money format")
    }
    dollars, _ := strconv.ParseInt(parts[0], 10, 64)
    cents, _ := strconv.ParseInt(parts[1], 10, 64)
    m.int64 = dollars*100 + cents
    return nil
}
```

### 2.4 性能对比与选型建议

| 方案 | 序列化速度 | 反序列化速度 | 内存占用 | 兼容性 |
|------|------------|--------------|----------|--------|
| 标准库 | ⭐ | ⭐ | 较高 | 100% |
| json-iterator | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 中等 | 95%（需测试） |
| sonic | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 较低 | 90%（需测试） |
| go-json | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 较低 | 90% |

```go
// 选型建议：
// 1. 普通 API（QPS < 1万）：标准库足够，稳定性优先
// 2. 高频内部服务（QPS 1万-50万）：go-json，兼容性好
// 3. 极致性能场景（QPS > 50万）：sonic，但需要充分测试边界情况

// 迁移示例：保留标准库接口，换实现
type JSONSerializer interface {
    Marshal(v any) ([]byte, error)
    Unmarshal(data []byte, v any) error
}

type StandardSerializer struct{}
func (s *StandardSerializer) Marshal(v any) ([]byte, error) { return json.Marshal(v) }
func (s *StandardSerializer) Unmarshal(data []byte, v any) error { return json.Unmarshal(data, v) }

type SonicSerializer struct{}
func (s *SonicSerializer) Marshal(v any) ([]byte, error) { return sonic.Marshal(v) }
func (s *SonicSerializer) Unmarshal(data []byte, v any) error { return sonic.Unmarshal(data, v) }

// 通过配置切换，不需要改业务代码
var serializer JSONSerializer = &StandardSerializer{}
if viper.GetBool("high_performance_mode") {
    serializer = &SonicSerializer{}
}
```

---

## 3. time：高频踩坑

### 3.1 time.After 泄漏（经典问题）

```go
// 问题代码：每次循环都创建一个新的 Timer，旧的不会 GC
func processLoop() {
    for {
        select {
        case msg := <-ch:
            handle(msg)
        case <-time.After(time.Second): // 每秒创建一个 Timer，永不回收（< Go 1.23）
            doCleanup()
        }
    }
}

// 修复方案 1：只创建一次（适用于循环处理）
func processLoopFixed() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    for {
        select {
        case msg := <-ch:
            handle(msg)
        case <-ticker.C:
            doCleanup()
        }
    }
}

// 修复方案 2：自己管理 timeout channel（避免每次创建 Timer）
// Go 1.23 改善：未回收的 Timer 可以被 GC 回收
// 但仍然建议显式管理，避免心智负担
```

### 3.2 time.AfterFunc 与 goroutine 泄漏

```go
// 问题代码：AfterFunc 启动一个 goroutine，但不清理
time.AfterFunc(5*time.Second, func() {
    fmt.Println("这个 goroutine 如果 AfterFunc 被调用多次会累积")
})
// 如果 thisFn 被调用 1000 次，就创建 1000 个 goroutine，都等着

// 修复：用 context + select 代替 AfterFunc
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
select {
case <-result:
    return result
case <-ctx.Done():
    return nil // 超时
}
```

### 3.3 Go 1.23 Timer/Ticker 改进

```go
// Go 1.23 之前的 Timer 泄漏：
// Timer 如果没有被 Stop()，底层 channel 和 goroutine 不会被 GC
// Go 1.23：Timer 类型实现了接口，未被引用的 Timer 会被 GC 回收

// 但注意：这不意味着你可以乱用 Timer/Ticker
// 正确的资源管理仍然是最佳实践

// Go 1.23 新 API：time.Timer 和 time.Ticker 实现 cleanup
timer := time.NewTimer(5 * time.Second)
// 超过 duration 后，timer 会被 GC 回收（不再需要显式 Stop）
```

### 3.4 时区处理陷阱

```go
// 错误：每次请求都调用 LoadLocation（耗性能）
loc, _ := time.LoadLocation("Asia/Shanghai") // 网络 IO，/usr/share/zoneinfo
_, _ = time.LoadLocation("Asia/Shanghai") // 重复调用

// 正确：缓存 location 对象（全局变量或 init）
var shanghaiLoc = time.FixedZone("CST", 8*3600) // 不依赖文件系统

// 或者用 time.LoadLocation 从不变化的配置
var localTZ, _ = time.LoadLocation("America/New_York") // 应用启动时加载一次

// 数据库时区陷阱
// MySQL 的 datetime 是没有时区的字符串表示
// Java 的 Timestamp 存的是 UTC 毫秒数
// Go 读取时：需要明确转换，否则会出现 8 小时偏差
row := db.QueryRow("SELECT created_at FROM orders LIMIT 1")
var t time.Time
row.Scan(&t) // t 是数据库 driver 解析后的值，取决于 driver 实现

// 安全做法：数据库存 UTC，Go 存 time.Time，JSON 序列化时转本地时间
```

### 3.5 单调时钟 vs 挂钟

```go
// time.Now() 返回的是"墙上时钟"（wall clock），可以被 NTP 调整
// time.Since(start) 基于"单调时钟"，不受 NTP 影响

start := time.Now()
time.Sleep(100 * time.Millisecond)
elapsed := time.Since(start) // 精确，无 NTP 影响

// 但注意：monotonic clock 在进程重启后会重新计数
// 所以测量长时任务时，定期记录 wall clock 做 checkpoint

// Duration 的精度问题
// Duration 表示纳秒，最大约 290 年（math.MaxInt64 / int64(time.Second)）
// time.Duration 是 int64 的别名，单位是纳秒
d := 1*time.Hour + 30*time.Minute + 45*time.Second
fmt.Println(d) // "2h30m45s"
```

---

## 4. testing：最佳实践

### 4.1 Table-Driven Test：Go 测试的标准模式

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name    string
        a, b    int
        want    int
        wantErr bool
    }{
        {"正常相加", 1, 2, 3, false},
        {"负数", -1, -1, -2, false},
        {"溢出", math.MaxInt, 1, 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Add(tt.a, tt.b)
            if (err != nil) != tt.wantErr {
                t.Errorf("Add() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if got != tt.want {
                t.Errorf("Add() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

### 4.2 TestMain：全局 setup/teardown

```go
var (
    db   *sql.DB
    redis *redis.Client
)

func TestMain(m *testing.M) {
    // 全局 setup
    var err error
    db, err = initDB()
    if err != nil {
        fmt.Printf("DB init failed: %v\n", err)
        os.Exit(1)
    }
    redis, err = initRedis()
    if err != nil {
        fmt.Printf("Redis init failed: %v\n", err)
        os.Exit(1)
    }

    // 运行所有测试
    code := m.Run()

    // teardown（资源清理）
    db.Close()
    redis.Close()

    os.Exit(code)
}
```

### 4.3 Subtests 与并行测试

```go
func TestPipeline(t *testing.T) {
    t.Run("顺序处理", func(t *testing.T) {
        // 这个 subtest 不会并行执行
    })

    t.Run("并行处理", func(t *testing.T) {
        t.Parallel() // 标记为可并行，后续 sibling subtest 会同时运行
        // 并行测试内容
    })
}

// 并行测试注意事项：
// 1. 每个 subtest 都要用 t.Parallel()
// 2. 确保测试之间没有共享状态冲突
// 3. 使用 subtest 名字做资源隔离（如：test DB 名称包含子测试名）
```

### 4.4 Mock 最佳实践

```go
// 接口隔离：依赖注入 + mock
type UserRepo interface {
    GetUser(ctx context.Context, id int64) (*User, error)
}

// 使用 testify/mock（推荐，比 gomock 更简洁）
type MockUserRepo struct {
    mock.Mock
}

func (m *MockUserRepo) GetUser(ctx context.Context, id int64) (*User, error) {
    args := m.Called(ctx, id)
    return args.Get(0).(*User), args.Error(1)
}

func TestUserService_GetUser(t *testing.T) {
    mockRepo := new(MockUserRepo)
    svc := NewUserService(mockRepo)

    mockRepo.On("GetUser", mock.Anything, int64(123)).Return(&User{ID: 123, Name: "test"}, nil)

    user, err := svc.GetUser(context.Background(), 123)
    assert.NoError(t, err)
    assert.Equal(t, "test", user.Name)
    mockRepo.AssertExpectations(t)
}

// gomock vs testify/mock 对比：
// gomock: 代码生成，强类型，但学习成本高，修改接口需要重新生成
// testify/mock: 纯手写，更灵活，但缺少静态检查
// 推荐：testify/mock，适合快速迭代
```

### 4.5 Fuzz Testing

```go
// Fuzz Testing：自动生成边界输入，发现隐藏 bug
// Go 1.18+ 内置支持

func FuzzReverse(f *testing.F) {
    testcases := []string{"Hello, World", " ", "!12345"}
    for _, tc := range testcases {
        f.Add(tc) // 告诉 fuzzer 这些种子输入格式
    }

    f.Fuzz(func(t *testing.T, s string) {
        // fuzzer 会不断生成各种 s 输入来测试这个函数
        r := Reverse(s)
        reversed := Reverse(r)
        if s != reversed {
            t.Errorf("Reverse(%q) = %q, Double Reverse should be equal", s, r)
        }
    })
}

// 运行 fuzz 测试
// go test -fuzz=FuzzReverse -fuzztime=10s
// fuzzer 会自动找到：空字符串、超长字符串、特殊字符、UTF-8 边界等

// 生产级 fuzz 测试案例：JSON 解析
func FuzzJSON(f *testing.F) {
    f.Fuzz(func(t *testing.T, data []byte) {
        var v map[string]interface{}
        err := json.Unmarshal(data, &v)
        if err != nil {
            return // 跳过非法 JSON（预期行为）
        }
        // 验证：序列化后再反序列化，结果一致
        encoded, err := json.Marshal(v)
        if err != nil {
            t.Errorf("re-marshal failed: %v", err)
        }
        var v2 map[string]interface{}
        json.Unmarshal(encoded, &v2)
        // 比较 ...
    })
}
```

### 4.6 覆盖率与性能测试

```bash
# 统计覆盖率
go test -cover -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# 性能测试基准
go test -bench=. -benchmem -run=^$
# -benchmem: 显示内存分配
# -benchtime=10s: 延长基准测试时间，更精确

# 生成 CPU profile
go test -cpuprofile=cpu.prof -bench=.
go tool pprof -http=:8080 cpu.prof

# 生成内存 profile
go test -memprofile=mem.prof -bench=.
go tool pprof -http=:8080 mem.prof
```

---

## 高频追问

**Q1: bufio 缓冲和直接 io.Reader 读取，性能差多少？**
通常 3-10 倍，取决于场景。在高并发网络请求中，bufio 可以减少 50%+ 系统调用次数。注意：对于顺序读取大文件，直接用 `ioutil.ReadAll`（内部用 32KB 缓冲）通常足够。

**Q2: Go 的 encoding/json 怎么处理循环引用？**
标准库直接返回 `错误：json: unsupported value: encountered recursively`（具体错误信息）。不能序列化，必须用自定义的 `MarshalJSON`/`UnmarshalJSON` 打破循环，或者在数据结构设计时避免循环引用。

**Q3: time.LoadLocation 每次都读文件吗？**
是的，`LoadLocation` 会读 `/usr/share/zoneinfo` 下的时区数据文件（有 OS 缓存，但仍有 syscall 开销）。生产环境应该用 `time.FixedZone` 或在应用启动时加载一次。

**Q4: 为什么 time.Duration 是 int64 的别名而不是自定义类型？**
因为数学运算可以直接用 `time.Second * 5`（int64 * int64 = int64），不需要类型转换。如果用自定义类型，每次都要强制转换，代码会非常冗长。

---

## 相关面经

- [GMP 调度](./01-golang/01-runtime/01-gmp.md) - goroutine 调度与 time.Sleep 的关系
- [GC 机制](./01-golang/01-runtime/02-gc.md) - GC 对 Timer/Ticker 的影响（Go 1.23 改进）
- [sync 原语](./01-golang/02-concurrency/02-sync.md) - 并发测试中的同步原语使用