# net/url · http body · os.Root · netip：标准库生产加固实践

> 本篇文章聚焦「上线才会炸」的标准库细节：URL 编码错位导致的签名失败、Request.Body 被读空、Scanner 的 64KB 天花板、路径穿越、随机数用错、优雅退出的竞态。**每一个都是真实故障的高频根因。**

## 面试官考察意图

标准库的 API 谁都会调，但**边界条件**才是区分「写过 demo」和「扛过线上」的地方。面试官问这些题，想确认三件事：

1. 你是否知道标准库 API 的**隐式约束**（Body 只能读一次、Scanner 默认 64KB、rand 全局锁）
2. 你是否在真实项目里踩过并系统性地解决了这些问题
3. 你是否理解 Go 团队对这些问题的**官方演进方向**（os.Root、netip、OnceFunc、rand/v2）

---

## 🔥🔥🔥🔥🔥 题目一：net/url 编码陷阱 —— QueryEscape / PathEscape / JoinPath

### 核心答案

Go 的 `net/url` 有**两套完全不同的转义规则**，用错就会导致服务端收到的参数与客户端拼的不一致：

- `url.QueryEscape` — 把**空格转成 `+`**，用于 query string 的 key/value
- `url.PathEscape` — 把**空格转成 `%20`**，用于 URL 路径段
- `url.PathEscape` **不转义 `+`**，`url.QueryEscape` 也不转义 `+` → `+` 在 query 里语义是空格，**这是签名错位的头号元凶**

再加一条：**拼接 URL 不要用字符串加法，用 `url.JoinPath` + `url.Values.Encode()`**。

### 详细解析

#### 1. 三套转义规则对照表

| 字符 | QueryEscape（query value） | PathEscape（路径段） | url.QueryEscape 编码后 | PathEscape 编码后 |
|---|---|---|---|---|
| 空格 | 编码 | 编码 | `+` | `%20` |
| `+` | **不编码** | **不编码** | `+` | `+` |
| `/` | 编码 | **不编码** | `%2F` | `/` |
| `?` | 编码 | 编码 | `%3F` | `%3F` |
| `&` | 编码 | 编码 | `%26` | `%26` |
| `=` | 编码 | 编码 | `%3D` | `%3D` |

注意最反直觉的两行：

- QueryEscape 把空格变 `+`，而 `+` 本身**不转义**。所以字符串 `"a b"` 和 `"a+b"` 编码后都是 `a+b` —— 服务端 `url.ParseQuery` **无法区分**它们，两者都会被解码成 `"a b"`。这就是签名/校验失败经典根因。
- PathEscape **不转义 `/`**！所以 `PathEscape("a/b")` == `"a/b"`，直接用作路径段会**多出一级路径**。

#### 2. 正确姿势：签名串必须转义两次

签名场景（阿里云/腾讯云/微信支付）的规范是 **RFC 3986**，不是 Go 的 QueryEscape：

```go
// ❌ 错误：Go 的 QueryEscape 会漏掉 + / * 等字符，与其他语言 SDK 不一致
signStr := url.QueryEscape(value)

// ✅ 正确：自定义 RFC3986 编码（阿里云 OSS 官方实现思路）
func rfc3986Escape(s string) string {
    escaped := url.QueryEscape(s)
    // QueryEscape 不处理的字符，按 RFC3986 补齐
    escaped = strings.ReplaceAll(escaped, "+", "%20")
    escaped = strings.ReplaceAll(escaped, "*", "%2A")
    escaped = strings.ReplaceAll(escaped, "%7E", "~")
    return escaped
}
```

**面试必答**：为什么不能直接用 `url.QueryEscape` 做签名？
因为签名算法是**跨语言约定**（Java SDK、Python SDK、Go SDK 必须产出同一个签名字符串），而 Go 的 `QueryEscape` 遵循的是 `application/x-www-form-urlencoded` 规则（`+` 代表空格），与 RFC3986 不一致。**跨语言签名必须自己实现 RFC3986**。

#### 3. URL 拼接：为什么不能用字符串加法

```go
// ❌ 危险：base 带 path、query 带特殊字符时全部错位
u := base + "/api/v1/" + id + "?q=" + keyword

// ✅ 正确一：PathEscape 处理路径段 + url.Values 处理 query
u := fmt.Sprintf("%s/api/v1/%s?%s", base, url.PathEscape(id), url.Values{"q": {keyword}}.Encode())

// ✅ 正确二（Go 1.19+）：JoinPath 自动处理分隔符与转义
joined, err := url.JoinPath(base, "api", "v1", id) // id 中的 / 会被保留为分隔符！
if err != nil { return err }

// ✅ 最稳：构造 url.URL 结构体，让标准库负责拼接
target := &url.URL{
    Scheme:   "https",
    Host:     "api.example.com",
    Path:     path.Join("/api/v1", id), // path.Join 会 Clean 掉 ../
    RawQuery: url.Values{"q": {keyword}}.Encode(),
}
fmt.Println(target.String())
```

⚠️ `url.JoinPath` 的坑：它**保留**参数里的 `/`，所以 `JoinPath(base, "a/b")` 会拼出 `base/a/b`。如果 `id` 来自用户输入且含 `../`，就存在**路径穿越风险**。需要用户输入参与路径时，必须 `url.PathEscape(id)`（PathEscape 会把 `/` 转成 `%2F`）。

#### 4. 实战：服务端解析的对称性问题

```go
// 服务端接受 ?q=%2B%2B 和 ?q=++ 是不一样的
r.URL.Query().Get("q") // 内部走 url.ParseQuery，+ 已被还原为空格

// 如果原始语义里 + 就是加号（比如 base64 token），必须显式处理
raw := r.URL.RawQuery                       // 拿原始串
vals, _ := url.ParseQuery(raw)              // 标准解析（+ → 空格）
// 想保留 + 的字面量：客户端必须传 %2B，服务端才会解出 +
```

**结论**：传输 base64（含 `+` `/` `=`）时，要么用 URL-safe base64（`base64.URLEncoding`，用 `-` `_` 替代），要么严格 `PathEscape`/`QueryEscape` 传输，**绝不要裸传**。

### 面试话术

> "`net/url` 有两套转义：QueryEscape 把空格转 `+`，PathEscape 转 `%20` 且不转义 `/`。做签名必须按 RFC3986 自己补齐 `+ * ~`，因为签名是跨语言约定，Go 的 QueryEscape 遵循的是 form-urlencoded 规则，跟 Java/Python SDK 对不齐就成了线上签名失败的头号原因。拼 URL 我优先构造 `url.URL` 结构体，而不是字符串加法。"

---

## 🔥🔥🔥🔥🔥 题目二：http.Request.Body 只能读一次 —— GetBody 与 MaxBytesReader

### 核心答案

`http.Request.Body` 是**一次性流**，读完就到底了。三个必知点：

1. **需要重放 Body 用 `req.GetBody`**（服务端请求为 nil，客户端请求由 `NewRequest` 自动填充）
2. **必须限制 Body 大小**，否则 `io.ReadAll` 会被恶意大请求打爆内存 → `http.MaxBytesReader`
3. **必须 Close + 并发场景要 Drain**，否则连接无法复用

### 详细解析

#### 1. 为什么 Body 只能读一次？

Body 底层是 `io.ReadCloser`（`http.body` / `*os.File` / `io.Pipe` 流），本质是**网络字节流**，没有 seek 能力。第一次 `Read` 消费掉的数据不会回来。

```go
body, _ := io.ReadAll(r.Body)
// 此时 r.Body 已 EOF

// ❌ 再读拿到的是空字符串
again, _ := io.ReadAll(r.Body) // again == ""

// ✅ 方案一：读完自己缓存在内存里（小 body）
// ✅ 方案二：需要重放时用 GetBody
```

#### 2. 客户端重放：GetBody 的正确用法

```go
req, _ := http.NewRequest("POST", url, bytes.NewReader(payload))
// NewRequest 对 *bytes.Reader / *strings.Reader / *bytes.Buffer 会自动设置 GetBody

// 重试 / 重定向 / 签名重算时拿到全新的 Body
if req.GetBody != nil {
    newBody, err := req.GetBody() // 返回一个新的 io.ReadCloser
    if err != nil { return err }
    defer newBody.Close()
    req.Body = newBody
}
```

⚠️ **手动构造 `&http.Request{Body: ...}` 时 `GetBody` 是 nil**，重试就会失效。生产里的 HTTP 重试中间件必须先检查 `GetBody != nil`，否则重试请求会带着空 body 发出去。

#### 3. 服务端必须限制 Body 大小（防 OOM）

```go
const maxBody = 1 << 20 // 1MB

func handler(w http.ResponseWriter, r *http.Request) {
    // MaxBytesReader 会在超限时返回错误，并设置 Connection: close
    r.Body = http.MaxBytesReader(w, r.Body, maxBody)

    body, err := io.ReadAll(r.Body)
    if err != nil {
        var maxErr *http.MaxBytesError
        if errors.As(err, &maxErr) {
            http.Error(w, "request body too large", http.StatusRequestEntityTooLarge) // 413
            return
        }
        http.Error(w, "bad request", http.StatusBadRequest)
        return
    }
    _ = body
}
```

**为什么用 `MaxBytesReader` 而不是先 `Content-Length` 判断？**
`Content-Length` 可以被伪造或缺失（chunked 编码时根本没有）。`MaxBytesReader` 是**边读边计数**，读超就中断，是唯一可靠的防线。Go 1.19+ 提供 `*http.MaxBytesError` 供 `errors.As` 判定。

#### 4. Drain + Close：连接复用的前提

```go
resp, err := client.Do(req)
if err != nil { return err }
defer func() {
    // 只读了部分 body 时，必须丢弃剩余内容，否则连接不会归还到连接池
    io.Copy(io.Discard, io.LimitReader(resp.Body, 4<<10)) // 最多丢弃 4KB
    resp.Body.Close()
}()
```

**面试追问「不 Drain 会怎样」**：Transport 无法确定连接上还有没有未读字节，只能**关闭该 TCP 连接**。表现为 `TIME_WAIT` 激增、连接池永远建不满、QPS 上不去。

#### 5. 一次读取 + 多次使用（文件上传 / 签名校验）

```go
// 上传文件既要算 SHA256 校验，又要转发给后端
hasher := sha256.New()
// TeeReader：读出的数据同时写进 hasher，不产生第二份内存拷贝
body := io.TeeReader(r.Body, hasher)

// 转发（流式，不落盘、不全量进内存）
resp, err := http.Post(backendURL, r.Header.Get("Content-Type"), body)
if err != nil { return err }
defer resp.Body.Close()

checksum := hex.EncodeToString(hasher.Sum(nil)) // 转发完成后校验值已算好
```

### 面试话术

> "Body 是一次性流，读完就 EOF。客户端要重试就得靠 `GetBody`，但手搓 `http.Request` 时它是 nil，重试中间件必须判空。服务端我第一件事就是 `http.MaxBytesReader` 包一层限制大小——`Content-Length` 可以伪造，chunked 时压根没有，只有边读边计数才防得住 OOM。拿到响应体如果只读了一部分，必须 Drain 再 Close，否则连接进不了连接池，表现就是 TIME_WAIT 暴涨和 QPS 上不去。"

---

## 🔥🔥🔥🔥 题目三：bufio.Scanner 的 64KB 陷阱

### 核心答案

`bufio.Scanner` 默认单 token 上限是 **`bufio.MaxScanTokenSize` = 64KB**。超过就 `Scan()` 返回 false，而 `Err()` 报 `bufio.Scanner: token too long`。

**最阴险的是**：很多人只判断 `for scanner.Scan()` 退出就完事，**不检查 `scanner.Err()`**，导致**数据被静默截断**——日志少了几万行，业务却以为读完了。

### 详细解析

#### 1. 复现与修复

```go
// ❌ 静默丢数据：超过 64KB 的行整行丢弃，循环直接结束
f, _ := os.Open("big.log")
defer f.Close()
sc := bufio.NewScanner(f)
for sc.Scan() {
    process(sc.Text())
}
// 少了这行 → 出问题只能靠猜
// if err := sc.Err(); err != nil { log.Fatal(err) }

// ✅ 修复一：显式扩缓冲（最大 10MB）
sc = bufio.NewScanner(f)
sc.Buffer(make([]byte, 64<<10), 10<<20) // 初始 64KB，上限 10MB
for sc.Scan() {
    if len(sc.Bytes()) > 1<<20 {
        log.Printf("警告：单行超 1MB (%d bytes)", len(sc.Bytes()))
    }
    process(sc.Text())
}
if err := sc.Err(); err != nil {
    return fmt.Errorf("scan: %w", err)
}
```

#### 2. 三套方案怎么选

| 方案 | 内存模型 | 适用场景 | 单行上限 |
|---|---|---|---|
| `bufio.Scanner` | 内部缓冲区（默认 64KB） | 结构化短行（日志、CSV） | 64KB（可 `Buffer` 调大） |
| `bufio.Reader.ReadString('\n')` | 动态增长，**无上限** | 行长不可控（用户输入、JSON 行） | 无（受内存限制） |
| `bufio.Reader.ReadSlice('\n')` | **零分配**，复用内部缓冲 | 极高频读取，需自己处理 `ErrBufferFull` | 缓冲区大小 |

```go
// ReadString：无上限，但每行都分配新 string
br := bufio.NewReaderSize(f, 1<<20)
for {
    line, err := br.ReadString('\n')
    if len(line) > 0 { process(strings.TrimRight(line, "\n")) }
    if err == io.EOF { break }
    if err != nil { return err }
}
```

⚠️ `ReadString` 在遇到**没有换行符结尾**的超大文件时会一次性吃掉整个文件到内存——所以它也不是无脑安全，超大文件还是要用 `ReadSlice` + 手动拼接，或加 `io.LimitReader`。

#### 3. Scanner 的另一个坑：分配与 GC

`sc.Text()` 每次返回**新 String**（拷贝），`sc.Bytes()` 返回的是**内部缓冲切片**——缓冲会在下一次 `Scan()` 时被覆盖，**不能长期持有**：

```go
// ❌ 灾难：所有元素都指向同一块内存，且内容被后续扫描覆盖
var results [][]byte
for sc.Scan() {
    results = append(results, sc.Bytes())
}

// ✅ 需要留存就拷贝一份
var results [][]byte
for sc.Scan() {
    b := make([]byte, len(sc.Bytes()))
    copy(b, sc.Bytes())
    results = append(results, b)
}
```

### 面试话术

> "Scanner 默认单行 64KB 上限，超了 `Scan()` 直接返回 false，如果只写 `for sc.Scan()` 不看 `Err()`，数据会被静默截断——这是我们线上排查过一次的坑。要么用 `sc.Buffer(initial, max)` 调大上限，要么换 `ReadString` 或 `ReadSlice`。另外 `sc.Bytes()` 返回的是会被覆盖的内部缓冲，要留存必须 copy。"

---

## 🔥🔥🔥🔥 题目四：os.Root（Go 1.24）—— 目录逃逸的官方解法

### 核心答案

用户可控路径拼接是**路径穿越（Path Traversal）漏洞**的根源。传统做法是 `filepath.Clean` + 前缀校验，但**校验和打开之间存在 TOCTOU 竞态**（符号链接可在两步之间被替换）。

Go 1.24 引入 `os.Root`：**把根目录固定在 fd 上，所有操作在内核层面按 `openat2` / `O_NOFOLLOW` 语义限制在该子树内**，从根本上消灭穿越。

### 详细解析

#### 1. 传统防法的三个漏洞

```go
// ❌ 漏洞一：Clean 不处理符号链接
// 攻击者先上传 strconv 命名的软链接指向 /etc/passwd
safePath := filepath.Join(baseDir, filepath.Clean(userInput))
// 输入 "../../etc/passwd" 被 Clean 成 "../../etc/passwd" → 逃逸

// ❌ 漏洞二：前缀校验被绕过
if !strings.HasPrefix(safePath, baseDir) { return err }
// 输入 ../baseDir_evil/x → HasPrefix(".../baseDir_evil/x", ".../baseDir") == true

// ❌ 漏洞三：TOCTOU（校验完到打开之间被换链接）
absPath, _ := filepath.Abs(relPath)
if !strings.HasPrefix(absPath, baseDir) { return err } // 此刻通过
f, _ := os.Open(absPath)                              // 此刻 baseDir 已被换成符号链接
```

#### 2. os.Root 的正确用法

```go
import "os"

func serveUserFile(username, filename string) ([]byte, error) {
    base, err := os.OpenRoot("/srv/uploads")
    if err != nil {
        return nil, err
    }
    defer base.Close()

    // 只访问 /srv/uploads/<username>/<filename>
    // 任何 ../、绝对路径、能逃出 root 的符号链接都会被拒绝（errors.Is(err, os.ErrNotExist)）
    f, err := base.Open(filepath.Join(username, filename))
    if err != nil {
        return nil, err
    }
    defer f.Close()
    return io.ReadAll(f)
}
```

`os.Root` 提供的全套方法：`Open` / `Create` / `OpenFile` / `Mkdir` / `Remove` / `Stat` / `Lstat` / `Rename` / `Link` / `Symlink` / `RemoveAll` / `ReadFile` / `WriteFile` / `MkdirAll` / `Chmod` / `Chown` 等（Go 1.25 补全了后几个）。

**语义保证**：
- 路径中的 `..` **不能**逃出 root
- **绝对路径**被当作相对 root 处理
- 解析过程中若遇到**指向 root 外的符号链接**，操作失败
- 安全性与内核支持挂钩（Linux `openat2` 最优，其他平台退回逐段校验）

#### 3. 治本方案：干脆别让用户控制路径

```go
// ✅ 最佳实践：文件名由服务端生成，用户输入只存映射
id := uuid.NewString()
diskPath := filepath.Join("/srv/uploads", id[:2], id) // 分片避免单目录过多文件
// 数据库存 (id, original_name, owner_id)，下载时用 id 查表
```

再加两道防线：
- `os.CreateTemp(dir, "upload-*")` 生成不可预测文件名（**注意 `dir` 必须固定可控**，不能用用户输入）
- 上传目录挂载时加 `nosuid,nodev,noexec`（Linux）

### 面试话术

> "路径穿越的老做法是 `filepath.Clean` 加前缀校验，但这有 TOCTOU 竞态——校验完到 open 之间符号链接可能被替换掉。Go 1.24 的 `os.Root` 把根目录固定在 fd 上，`..`、绝对路径、逃逸符号链接在内核层面直接被拒，这才是官方解法。当然最根本的还是别让用户控制磁盘路径，文件名服务端生成 UUID，原始名只存数据库映射。"

---

## 🔥🔥🔥🔥 题目五：sync.OnceFunc / OnceValue / OnceValues（Go 1.21）

### 核心答案

`sync.Once` 的经典误用是**忘记写错误处理**或**把 panic 传播出去**。Go 1.21 起官方提供了三个包装：

- `sync.OnceFunc(f)` → `func()`：`f` 只执行一次，**panic 也会被缓存**（后续调用直接再次 panic）
- `sync.OnceValue(f)` → `func() T`：返回值被缓存，**所有调用者拿到同一个值**
- `sync.OnceValues(f)` → `func() (T, error)`：最常用，**初始化错误也缓存**

一句话：**它们是「懒加载单例」的现代写法，代码量减少 60%，且天然并发安全。**

### 详细解析

#### 1. 对比：传统 sync.Once vs OnceValues

```go
// ❌ 传统写法：13 行，容易漏掉 "err 也要缓存" 这个细节
var (
    dbOnce sync.Once
    dbConn *sql.DB
    dbErr  error
)

func GetDB() (*sql.DB, error) {
    dbOnce.Do(func() {
        dbConn, dbErr = sql.Open("mysql", dsn)
    })
    return dbConn, dbErr
}

// ✅ Go 1.21+：4 行，且语义完全等价
var GetDB = sync.OnceValues(func() (*sql.DB, error) {
    return sql.Open("mysql", dsn)
})
```

⚠️ **注意 `sync.OnceValues` 会缓存失败结果**。如果你的初始化需要「失败后重试」，它**不适合**：

```go
// ❌ 错误：DB 第一次连接失败后，后续所有调用都拿到同一个错误，永远不会重试
var GetDB = sync.OnceValues(func() (*sql.DB, error) { ... })

// ✅ 需要失败重试 → 老实用互斥锁 + 状态判断
var (
    mu     sync.Mutex
    conn   *sql.DB
)
func GetDBRetryable() (*sql.DB, error) {
    mu.Lock()
    defer mu.Unlock()
    if conn != nil {
        if err := conn.Ping(); err == nil {
            return conn, nil
        }
        conn.Close() // 连接已死，重建
        conn = nil
    }
    db, err := sql.Open("mysql", dsn)
    if err != nil { return nil, err }
    conn = db
    return conn, nil
}
```

#### 2. OnceValue 缓存值 vs 每次都算

```go
// 配置解析：进程启动只解析一次，所有请求复用
var LoadConfig = sync.OnceValue(func() *Config {
    data, err := os.ReadFile("/etc/app/config.yaml")
    if err != nil { panic(err) }
    var c Config
    if err := yaml.Unmarshal(data, &c); err != nil { panic(err) }
    return &c
})

// 用的时候
cfg := LoadConfig() // 首次解析，后续直接返回同一指针
```

#### 3. panic 语义：这是最大的坑

```go
var init = sync.OnceFunc(func() {
    panic("boom")
})

func demo() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered:", r) // 每次调用都会走到这里
        }
    }()
    init() // panic: boom
    init() // 再调用仍然 panic: boom（f 不会重跑，但 panic 被重现）
}
```

**官方行为**：`OnceFunc` 内部用 `panic(nil)` 标记「已执行」，真 panic 会被重新抛出。所以**不要用 `OnceFunc` 包装可能 panic 的初始化**，应该包装返回 error 的版本。

#### 4. 与 SingleFlight 的区别（高频追问）

| 维度 | `sync.OnceFunc` 家族 | `singleflight.Group` |
|---|---|---|
| 生命周期 | **整个进程**只执行一次 | **每次并发窗口**合并一次 |
| 执行完再调用 | 直接返回缓存 | 会**重新执行** |
| 适用场景 | 单例初始化、配置加载 | 缓存击穿、热点查询合并 |

### 面试话术

> "`sync.Once` 手写单例容易漏掉「错误也要缓存」，Go 1.21 的 `sync.OnceValues` 一行搞定。但有个坑：它**会缓存失败结果**，如果初始化需要失败重试就不能用，得自己用 Mutex 加状态判断。另外它和 SingleFlight 完全不同——Once 是整个进程一次，SingleFlight 是每个并发窗口合并一次，后者会重新执行。"

---

## 🔥🔥🔥☆ 题目六：net/netip —— 告别 net.IP 的分配与比较陷阱

### 核心答案

`net.IP` 是 `[]byte`，**不可比较、可被意外修改、每次比较都要分配**。Go 1.18 引入的 `net/netip.Addr` 是**值类型（可比较、可直接做 map key）**，并且：

- **无堆分配**（16 字节数组 + zone 字符串，值语义）
- **`netip.Prefix` 支持 `Contains` / `Masked` / `Overlaps`**，比手写掩码运算安全
- **`ParseAddr` 比 `ParseIP` 严格**（拒绝前导零等歧义写法）

### 详细解析

#### 1. 三个致命差异

```go
// 差异一：net.IP 是不可比较的 slice
ip1 := net.ParseIP("1.2.3.4")
ip2 := net.ParseIP("1.2.3.4")
fmt.Println(ip1.String() == ip2.String()) // true，但这是字符串比较（慢）
// fmt.Println(ip1 == ip2) // 编译错误：slice 不能比较

// ✅ netip 直接比
a1 := netip.MustParseAddr("1.2.3.4")
a2 := netip.MustParseAddr("1.2.3.4")
fmt.Println(a1 == a2) // true，一条指令，无分配

// 差异二：net.IP 可以被调用方改坏
func badReturn() net.IP {
    ip := net.ParseIP("1.2.3.4")
    return ip // 返回内部切片，调用方能改
}
ip := badReturn()
ip[15] = 99 // 原对象被修改！

// ✅ netip.Addr 是值类型，天然不可变
```

#### 2. 典型场景：IPv4 表示的唯一性

```go
// ⚠️ net.IP 的隐性坑：同一地址有两种表示
v4 := net.ParseIP("1.2.3.4")                  // 16 字节（v4-in-v6）
v4mapped := net.ParseIP("::ffff:1.2.3.4")     // 也是 16 字节
fmt.Println(v4.To4() != nil, v4mapped.To4() != nil) // true true
// 用 IP 做 map key（string(ip)）会得到两个不同的 key！

// ✅ netip 自动归一化（Unmap）
a := netip.MustParseAddr("::ffff:1.2.3.4")
fmt.Println(a.Is4())  // true —— 已自动 Unmap
fmt.Println(a)        // 1.2.3.4
```

**这就是「IP 白名单绕不过」类漏洞的常见根因**：黑名单用 `net.IP` 字符串存，攻击者用 v4-mapped 写法就绕过了。

#### 3. CIDR 判断：手写掩码 vs Prefix

```go
// ❌ 手写容易搞错字节序和位数
func inCIDR(ip net.IP, cidr string) bool {
    _, ipnet, _ := net.ParseCIDR(cidr)
    return ipnet.Contains(ip) // 这个其实没问题，但 net.IP 解析有歧义
}

// ✅ netip 版本：明确、可比较、零分配
var internalNet = netip.MustParsePrefix("10.0.0.0/8")

func isInternal(addrStr string) (bool, error) {
    addr, err := netip.ParseAddr(addrStr)
    if err != nil {
        return false, err
    }
    return internalNet.Contains(addr), nil
}

// 前缀也支持映射（映射到非公有地址段，防 SSRF）
var privateNets = []netip.Prefix{
    netip.MustParsePrefix("10.0.0.0/8"),
    netip.MustParsePrefix("172.16.0.0/12"),
    netip.MustParsePrefix("192.168.0.0/16"),
    netip.MustParsePrefix("127.0.0.0/8"),
    netip.MustParsePrefix("169.254.0.0/16"), // 云元数据服务！
    netip.MustParsePrefix("::1/128"),
    netip.MustParsePrefix("fc00::/7"),
}

func isBlocked(addr netip.Addr) bool {
    for _, p := range privateNets {
        if p.Contains(addr) {
            return true
        }
    }
    return false
}
```

⚠️ **SSRF 防护必须包含 `169.254.169.254`**（AWS/GCP/阿里云元数据服务），这是拿云凭证的经典入口。另外要注意：**DNS 解析后再校验 IP**，否则 DNS Rebinding 可绕过（校验时解析成公网 IP，请求时解析成内网 IP）。

#### 4. netip 的性能优势（benchmark 量级）

| 操作 | net.IP | netip.Addr |
|---|---|---|
| 解析 | ~120 ns，1 alloc | ~25 ns，**0 alloc** |
| 比较 | 需 `bytes.Equal` | 一条整数比较 |
| map key | 需转 string（alloc） | 直接当 key，0 alloc |

高频场景（限流按 IP 分桶、WAF 规则匹配）**netip 能省掉大量 GC 压力**。

### 面试话术

> "`net.IP` 是 `[]byte`，不可比较、能被改坏，而且要小心 v4-in-v6 的两种表示——同一地址用字符串当 key 会变成两个 key，这就是 IP 黑名单被绕过的一类根因。`net/netip.Addr` 是值类型，可比较、可做 map key、解析零分配，快 4~5 倍。做 SSRF 防护的时候还要特别注意把 `169.254.169.254` 云元数据地址也加进黑名单。"

---

## 🔥🔥🔥☆ 题目七：crypto/rand vs math/rand —— 随机数用错就是漏洞

### 核心答案

- `math/rand`（及 Go 1.22+ 的 `math/rand/v2`）：**伪随机，可预测**，仅用于模拟、测试、非安全场景
- `crypto/rand`：**密码学安全**，用于 token、session id、盐值、验证码
- Go 1.20 起 `math/rand` 的全局 Seed **自动随机化**（不再需要 `rand.Seed(time.Now().UnixNano())`）
- Go 1.22 起 `math/rand/v2` **移除了 `Seed`**，全局函数使用 ChaCha8 随机源
- Go 1.24 新增 `crypto/rand.Text()`，**专门用于生成密钥/token 的 base32 字符串**

### 详细解析

#### 1. 为什么 math/rand 不能用于安全场景

`math/rand` 是 **PCG/加法滞后斐波那契**类确定性算法，只要知道**内部状态**或足够多的**连续输出**，就能**完全预测后续所有输出**。

真实案例模式：某系统用 `math/rand.Intn(1000000)` 生成短信验证码 → 攻击者注册几个账号收集验证码序列 → 反推随机种子（种子空间只有 `2^31`，可暴力枚举）→ 批量猜中他人验证码。

#### 2. 正确写法

```go
import (
    "crypto/rand"
    "encoding/base64"
    "encoding/hex"
    "math/big"
)

// ✅ 生成会话 token（32 字节 = 256 bit 熵）
func NewSessionToken() (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil { // crypto/rand.Read 保证填满
        return "", err
    }
    return base64.RawURLEncoding.EncodeToString(b), nil // URL-safe，无 padding
}

// ✅ 生成数字验证码（安全随机）
func SecureCode(digits int) (string, error) {
    max := new(big.Int).Exp(big.NewInt(10), big.NewInt(int64(digits)), nil)
    n, err := rand.Int(rand.Reader, max) // 均匀分布在 [0, max)
    if err != nil {
        return "", err
    }
    return fmt.Sprintf("%0*d", digits, n), nil
}

// ✅ Go 1.24+：一行拿到 26 字符的 base32 token（128 bit 熵）
func NewAPIKey() string {
    return "sk_" + rand.Text()
}
```

⚠️ **不要用 `rand.Intn` + 取模**做验证码：`rand.Intn(1000000)` 在数学上均匀，但 `math/rand` 的**种子可预测**才是问题；而 `x % n` 这种取模写法（对 `rand.Uint64()`）会引入**模偏差**，`crypto/rand.Int` 内部做了拒绝采样避免了这一点。

#### 3. Go 1.22+ math/rand/v2 的变化

```go
import (
    "math/rand"
    "math/rand/v2"
)

// 旧（Go 1.20 前需要手动 seed；1.20 后自动随机化但 API 仍然混乱）
rand.Seed(time.Now().UnixNano()) // Go 1.20 起已废弃
n := rand.Intn(100)

// 新：全局函数用 ChaCha8，无 Seed
n2 := rand.IntN(100) // 注意是大写 N！

// 新的并发友好用法：每个 goroutine 独立源，避免全局锁竞争
r := rand.New(rand.NewPCG(seed1, seed2))
n3 := r.IntN(100)

// 加密级伪随机（快但不可用于安全场景）
cr := rand.NewChaCha8([32]byte{})
```

| 特性 | math/rand（v1） | math/rand/v2 |
|---|---|---|
| 全局 Seed | Go 1.20 起自动随机 | **移除 Seed** |
| 默认源 | 加法滞后斐波那契 | **ChaCha8** |
| 整数 API | `Intn` | `IntN`（大写） |
| 并发源 | 全局锁 | `rand.New(rand.NewPCG(...))` 每 goroutine 一套 |

#### 4. crypto/rand 的工程注意点

```go
// 坑一：crypto/rand.Read 在 Go 1.24+ 才保证永不返回错误
//       更早版本必须检查 err（熵池不可用时返回错误）
b := make([]byte, 16)
if _, err := rand.Read(b); err != nil { // 不要忽略
    return err
}

// 坑二：Linux 上 crypto/rand 用 getrandom(2)，不会阻塞（除非极早期启动）
//       不要在 init() 里做大熵量操作

// 坑三：比较 token 必须用常数时间比较，防时序攻击
import "crypto/subtle"
if subtle.ConstantTimeCompare(got, want) != 1 {
    return errors.New("token mismatch")
}
```

### 面试话术

> "安全场景必须用 `crypto/rand`，`math/rand` 是确定性算法，收集足够输出就能反推状态——用 `math/rand` 生成验证码是典型的可被预测漏洞。Go 1.20 后 `math/rand` 全局 Seed 自动随机化了，1.22 的 `rand/v2` 干脆移除了 Seed 并换成 ChaCha8，还支持每个 goroutine 独立 PCG 源避免全局锁。生成 token 我现在直接用 Go 1.24 的 `crypto/rand.Text()`，一行拿到 128 位熵的 base32 串。比较 token 要用 `subtle.ConstantTimeCompare` 防时序攻击。"

---

## 🔥🔥🔥☆ 题目八：signal.NotifyContext + http.Server.Shutdown —— 生产级优雅退出

### 核心答案

生产级优雅退出的**四步顺序**不能错：

1. `signal.NotifyContext(SIGTERM, SIGINT)` 拿到取消信号
2. **摘流量**（K8s readiness 探针失败 / 注册中心注销）—— 必须**早于**关服务器
3. `srv.Shutdown(ctx)` 停止接受新连接，**等待存量请求完成**
4. 关闭下游资源（DB 连接池、MQ 生产者），超时兜底 `srv.Close()` 强杀

**最容易错的一点**：`Shutdown` 之后立刻退出进程，**根本没等请求处理完**。

### 详细解析

#### 1. 完整可运行骨架

```go
package main

import (
    "context"
    "errors"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    srv := &http.Server{
        Addr:              ":8080",
        Handler:           newRouter(),
        ReadHeaderTimeout: 5 * time.Second,  // 防 Slowloris
        ReadTimeout:       15 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       90 * time.Second,
    }

    // 监听信号，ctx 会在收到 SIGTERM/SIGINT 时被取消
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGTERM, syscall.SIGINT)
    defer stop() // 必须 defer：恢复信号默认行为，否则二次 Ctrl+C 无效

    errCh := make(chan error, 1)
    go func() {
        slog.Info("server starting", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            errCh <- err
        }
    }()

    select {
    case err := <-errCh:
        slog.Error("server failed", "err", err)
        os.Exit(1)

    case <-ctx.Done():
        slog.Info("shutdown signal received")
        stop() // 立刻恢复默认行为：之后再来信号直接强杀，给运维留后手

        // ⚠️ 关键点：Shutdown 用独立的 ctx，不能用已经取消的 ctx！
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()

        if err := srv.Shutdown(shutdownCtx); err != nil {
            slog.Error("graceful shutdown failed, forcing close", "err", err)
            _ = srv.Close() // 超时兜底：强杀残留连接
        }

        // 再关下游依赖
        if err := closeDependencies(shutdownCtx); err != nil {
            slog.Error("close deps", "err", err)
        }
        slog.Info("server exited cleanly")
    }
}
```

#### 2. 三个高频坑

**坑一：用被取消的 ctx 去 Shutdown —— 立刻返回 context.Canceled，等于没优雅**

```go
// ❌ ctx 已经被 cancel 了
if err := srv.Shutdown(ctx); err != nil { ... }

// ✅ 新建一个带超时的、独立的 ctx
shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
srv.Shutdown(shutdownCtx)
```

为什么 `Shutdown` 要接收 ctx 而不是自己超时？因为**只有调用方知道业务能容忍多久**——30 秒还是 5 分钟取决于是否存在长连接/大文件下载。

**坑二：没摘流量就关服务器 —— 用户看到 502**

```yaml
# K8s 正确姿势：readiness + preStop + terminationGracePeriodSeconds 三件套
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]  # 等 Endpoint 从 Service 摘除
terminationGracePeriodSeconds: 35          # 必须 > Shutdown 超时(30s) + preStop(5s)
```

顺序必须是：**SIGTERM 到达 → preStop sleep 等摘流量 → 应用 Shutdown 等存量请求 → 进程退出**。

**坑三：忽略了 Keep-Alive 长连接**

`Shutdown` **不会主动关闭空闲的 Keep-Alive 连接**（除非连接进入 Idle 状态）。若客户端一直 keep-alive 且不发请求，`Shutdown` 会一直等到 ctx 超时。

解决：设 `IdleTimeout`（让空闲连接自然关闭），或在网关层主动发 `Connection: close`。

#### 3. 后台 goroutine 也要管

```go
// 主流程 wait group：所有后台任务都要能被 ctx 取消
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    ticker := time.NewTicker(time.Minute)
    defer ticker.Stop() // ⚠️ 别忘了
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            syncJob(ctx)
        }
    }
}()

// Shutdown 之后
done := make(chan struct{})
go func() { wg.Wait(); close(done) }()
select {
case <-done:
case <-time.After(5 * time.Second):
    slog.Warn("background jobs did not exit in time")
}
```

#### 4. 优雅退出验证清单

```bash
# 1. 发 SIGTERM
kill -TERM $(pgrep -f myserver)

# 2. 观察日志顺序（必须严格按此序）
#    "shutdown signal received" → "server exited cleanly"

# 3. 并发压测下验证零 5xx
hey -z 30s -c 50 http://localhost:8080/api  # 中途发 SIGTERM，观察非 200 响应数

# 4. 验证长连接被告知关闭（不是直接 RST）
curl -v http://localhost:8080/ -H 'Connection: keep-alive'
```

### 面试话术

> "优雅退出四步：signal.NotifyContext 收信号 → 摘流量（K8s 里靠 preStop sleep 等 Endpoint 摘除）→ srv.Shutdown 等存量请求 → 关下游依赖。最大的坑是用已经被 cancel 的 ctx 去调 Shutdown，会立刻返回 context.Canceled，等于没生效——必须新建一个带超时的独立 ctx。另外 terminationGracePeriodSeconds 要大于 preStop 加 Shutdown 超时的总和，否则 K8s 会 SIGKILL 掉进程。"

---

## 🔥🔥☆☆ 题目九：httptest + httputil —— 不改业务代码测 HTTP

### 核心答案

`net/http/httptest` 提供两种测试模式：

- `httptest.NewRecorder()` — **进程内**测 handler，零网络开销，毫秒级
- `httptest.NewServer()` — 起**真实 TCP 服务**，用于测 client、超时、重试

外加 `httptest.NewTLSServer()`（测 TLS 逻辑）和 `httptest.NewRequest()`（构造请求）。

**关键区别**：`NewRecorder` 走的是 `ServeHTTP` 直接调用，**没有真实网络层**，所以**测不出超时、连接重用、TLS、chunked 编码**等行为——那些必须用 `NewServer`。

### 详细解析

#### 1. Handler 测试：NewRecorder（99% 的场景用这个）

```go
func TestCreateUser(t *testing.T) {
    h := NewUserHandler(fakeStore)

    req := httptest.NewRequest(http.MethodPost, "/users",
        strings.NewReader(`{"name":"alice"}`))
    req.Header.Set("Content-Type", "application/json")

    rec := httptest.NewRecorder()
    h.ServeHTTP(rec, req) // 直接调用，无网络

    if rec.Code != http.StatusCreated {
        t.Fatalf("got %d, want %d, body=%s", rec.Code, http.StatusCreated, rec.Body.String())
    }

    var got User
    if err := json.Unmarshal(rec.Body.Bytes(), &got); err != nil {
        t.Fatal(err)
    }
    if got.Name != "alice" {
        t.Errorf("got %q, want alice", got.Name)
    }
    if ct := rec.Header().Get("Content-Type"); ct != "application/json; charset=utf-8" {
        t.Errorf("Content-Type = %q", ct)
    }
}
```

#### 2. Client 测试：NewServer + 超时/重试验证

```go
func TestClientTimeout(t *testing.T) {
    slow := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(200 * time.Millisecond) // 模拟慢后端
        w.WriteHeader(http.StatusOK)
    }))
    defer slow.Close() // ⚠️ 必须 defer Close，否则 goroutine 泄漏 + 端口泄漏

    c := &http.Client{Timeout: 50 * time.Millisecond}

    _, err := c.Get(slow.URL)
    if err == nil {
        t.Fatal("expected timeout error")
    }

    var netErr net.Error
    if !errors.As(err, &netErr) || !netErr.Timeout() {
        t.Fatalf("expected a timeout error, got %v", err)
    }
}
```

⚠️ `httptest.NewServer` 用的是**真实 TCP + 随机端口**，所以：
- 必须 `defer srv.Close()`
- 依赖**本机网络栈**，CI 里沙箱禁网时可能失败（此时改用 `NewRecorder`）

#### 3. 表驱动 + 子测试的标准结构

```go
func TestRouter(t *testing.T) {
    router := newRouter()

    tests := []struct {
        name       string
        method     string
        path       string
        body       string
        wantCode   int
        wantSubstr string
    }{
        {"健康检查", http.MethodGet, "/healthz", "", http.StatusOK, `"status":"ok"`},
        {"未授权", http.MethodGet, "/api/v1/users", "", http.StatusUnauthorized, "unauthorized"},
        {"非法 JSON", http.MethodPost, "/api/v1/users", `{bad`, http.StatusBadRequest, "invalid"},
        {"超大 body", http.MethodPost, "/api/v1/users",
            strings.Repeat("a", 2<<20), http.StatusRequestEntityTooLarge, "too large"},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel() // 子测试并行

            req := httptest.NewRequest(tc.method, tc.path, strings.NewReader(tc.body))
            rec := httptest.NewRecorder()
            router.ServeHTTP(rec, req)

            if rec.Code != tc.wantCode {
                t.Errorf("code = %d, want %d", rec.Code, tc.wantCode)
            }
            if !strings.Contains(rec.Body.String(), tc.wantSubstr) {
                t.Errorf("body %q should contain %q", rec.Body.String(), tc.wantSubstr)
            }
        })
    }
}
```

#### 4. httputil：反向代理与请求转储

```go
// 测反向代理：用 NewServer 起「后端」，再用 proxy 转发
func TestReverseProxy(t *testing.T) {
    backend := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 验证代理正确传递了 X-Forwarded-For / X-Real-IP
        if r.Header.Get("X-Forwarded-For") == "" {
            t.Error("missing X-Forwarded-For")
        }
        w.Header().Set("X-Backend", "v1")
        fmt.Fprint(w, "hello from backend")
    }))
    defer backend.Close()

    target, _ := url.Parse(backend.URL)
    proxy := httputil.NewSingleHostReverseProxy(target)

    front := httptest.NewServer(proxy)
    defer front.Close()

    resp, err := http.Get(front.URL + "/foo")
    if err != nil { t.Fatal(err) }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    if string(body) != "hello from backend" {
        t.Errorf("body = %q", body)
    }
    if resp.Header.Get("X-Backend") != "v1" {
        t.Error("response header not passed through")
    }
}
```

⚠️ **`httputil.DumpRequest` 的坑**：`DumpRequestOut` 会读 body 且**不还原**，需要 `DumpRequest(req, true)` 并用 `req.GetBody()` 恢复。生产里不要用它做日志（会破坏 body）。

调试工具：

```go
// 仅开发环境：转储请求便于排查
if os.Getenv("DEBUG") == "1" {
    dump, _ := httputil.DumpRequest(req, true)
    slog.Debug("request dump", "raw", string(dump))
}
```

### 面试话术

> "handler 测试用 `httptest.NewRecorder`，进程内零网络开销；但要测超时、连接复用、TLS 这些网络行为就必须用 `httptest.NewServer` 起真实 TCP 服务，记得 `defer Close`。另外 CI 沙箱禁网时 `NewServer` 会失败。反向代理用 `httputil.NewSingleHostReverseProxy`，测的时候后端也用 `NewServer` 造，正好还能验证 X-Forwarded-For 有没有透传。"

---

## 高频追问汇总

**Q1：`url.Values.Encode()` 的排序规则是什么？**
按 key 的**字典序**排序（`sort.Strings`），所以同一组参数无论插入顺序如何，`Encode()` 结果**稳定**——这正是它能用于签名的原因。但注意同 key 多值时的顺序是**插入顺序**，签名时要确认双方约定。

**Q2：`http.MaxBytesReader` 超限后处理器还能继续写响应吗？**
可以。`MaxBytesReader` 会在超限时给 `ResponseWriter` 写入的响应自动加 `Connection: close`，并让 `Read` 返回 `*http.MaxBytesError`。但**必须在写响应之前**判定，否则你写了一半 200 再来 413 会造成协议错误。

**Q3：`os.Root` 在 macOS/Windows 上的安全保证一样吗？**
不完全一样。Linux 上若有 `openat2(2)`，可一次性原子解析完整路径；其他平台退回**逐段打开校验**，理论上仍有极小窗口。所以 `os.Root` 是「大幅降低风险」，但不等于绝对安全，配合 `nosuid,nodev,noexec` 挂载 + 服务端生成文件名才是完整方案。

**Q4：`sync.OnceValue` 返回的指针被调用方修改了怎么办？**
`OnceValue` 只保证「返回同一个值」，不保证不可变。返回指针时调用方仍能改内容。要真正只读，返回**值类型**（`Config` 而非 `*Config`）或内部持有 `sync.RWMutex` 做热更新。

**Q5：`netip.ParseAddr` 和 `net.ParseIP` 的行为差异还有哪些？**
`ParseAddr` **拒绝**前导零（`010.1.1.1` → 明确报错），`ParseIP` 在部分情况下会接受歧义写法；`ParseAddr` 支持带 zone 的 IPv6（`fe80::1%eth0`）并保留到 `Addr.Zone()`；`ParseIP` 丢失 zone 信息。**安全场景（IP 白名单）必须用 `ParseAddr`**——歧义解析是绕过黑名单的常见手法。

**Q6：优雅退出时正在处理的长连接（WebSocket / SSE）怎么办？**
`Shutdown` **不会**等待被 `Hijack` 的连接。必须自己维护一份活跃连接表：收到信号后先给连接发「服务重启」控制帧，等待客户端重连（或用 `ctx` 让 handler 感知取消），最后 `Close`。SSE 则靠 `ctx.Done()` 让 handler 返回。

---

## 延伸阅读

- [Go 1.24 release notes: os.Root](https://go.dev/doc/go1.24#os) — 官方文档与安全语义
- [go.dev/blog: Netip](https://go.dev/blog/netip) — netip 设计动机与性能对比
- [Go 1.21 release notes: sync.OnceFunc](https://go.dev/doc/go1.21#sync) — 三个 Once 家族 API
- [crypto/rand documentation](https://pkg.go.dev/crypto/rand) — Go 1.24 新增 Text() / Read 保证
- [net/http MaxBytesReader](https://pkg.go.dev/net/http#MaxBytesReader) — 与 MaxBytesError 配合的官方示例
- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal) — 路径穿越攻击模式全集
- [Kubernetes: Pod termination lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) — preStop 与 grace period 的官方时序

---

**[← 上一篇：reflect / regexp / 中间件 / filepath](./04-04-reflect-regexp-middleware.md)** · **[返回目录](../README.md)**
