# HTTP Chunked Transfer Encoding 详解

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：分块传输、Transfer-Encoding、chunked、HTTP/1.1、Content-Length

---

## 🎯 面试官考察意图

- 是否理解没有 Content-Length 时客户端如何知道响应何时结束
- 能否说清 chunked 编码的格式和工作原理
- 在生产环境中是否遇到过 chunked 相关的问题（流式输出、SSE、日志实时推送）
- 是否了解 chunked 在 WebSocket、gRPC 等协议中的应用

---

## ⚡ 核心答案（30秒）

> **Chunked Transfer Encoding** 是 HTTP/1.1 引入的一种编码方式，让服务器在不知道总长度的情况下也能发送响应。它将数据分成一个个带长度前缀的"块"（chunk），最后以空 chunk（长度为 0）结束。
>
> 典型应用：大文件下载、流式 AI 输出、Server-Sent Events（SSE）、实时日志推送、WebSocket 通信等场景。

---

## 🔬 深度展开

### 1. 为什么需要 Chunked？

```go
// 传统请求-响应（已知长度）：
GET /api/users HTTP/1.1
Host: api.example.com
    ↓
HTTP/1.1 200 OK
Content-Length: 1024
{"users":[...]}
        ↑ Content-Length 告诉客户端总共读 1024 字节

// 问题：流式响应无法使用 Content-Length！
// ① SSE（Server-Sent Events）：实时推送，不知道多少条消息
// ② Streaming AI（如 ChatGPT）：逐 token 输出
// ③ 文件生成中：边生成边返回
// ④ 实时日志推送：持续产生数据
// ⑤ gRPC streaming RPC：服务端持续发送消息

// 解决方案：Chunked Transfer Encoding
```

### 2. Chunked 编码格式

```
HTTP 响应体结构：
┌─────────────────────────────────┐
│ [Chunk Size](HEX)\r\n            │ ← 每个块的长度（十六进制）
│ [Chunk Data]                     │ ← 块数据
│ \r\n                             │ ← 行尾标记
│                                  │
│ [Chunk Size]\r\n                  │ ← 下一个块
│ [Chunk Data]                     │
│ \r\n                             │
│                                  │
│ 0\r\n                             │ ← 最后一个空块（长度 0）
│ [Trailer Headers]                │ ← 可选尾部头部
│ \r\n                              │
└─────────────────────────────────┘
```

#### 实际抓包示例

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

e                    ← "14" = 20 字节的数据
hello world!!!       ← 这正好是 14 个字符
0                    ← 最后的空 chunk，表示结束

或者更完整的情况：
HTTP/1.1 200 OK
Content-Type: application/octet-stream
Transfer-Encoding: chunked
X-Extra-Header: value   ← 可选 trailer headers

a                     ← chunk size: 16 bytes hex = decimal 10
Hello World!          ← exactly 10 chars
8                     ← next chunk size: 8 bytes
 Good Morning        ← exactly 8 chars (including leading space)
0                     ← final empty chunk
Trailer-Name: value   ← optional trailer headers before CRLF
                      ← extra blank line terminates
```

#### Go 代码实现

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

// 用 Chunked 实现流式时间推送服务
func streamHandler(w http.ResponseWriter, r *http.Request) {
    // 关键：必须设置 Transfer-Encoding: chunked
    w.Header().Set("Transfer-Encoding", "chunked")
    w.Header().Set("Content-Type", "text/plain")
    
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Streaming not supported", http.StatusInternalServerError)
        return
    }
    
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-r.Context().Done():
            fmt.Fprintln(w, "Client disconnected")
            flusher.Flush()
            return
        case now := <-ticker.C:
            // Go 的 http.ResponseWriter.Write 会自动添加 chunk 长度
            fmt.Fprintf(w, "%s\n", now.Format(time.RFC3339))
            flusher.Flush()  // 强制刷新缓冲区到客户端
        }
    }
}

func main() {
    http.HandleFunc("/stream", streamHandler)
    http.ListenAndServe(":8080", nil)
}
```

---

### 3. Go 中的 Chunked 实践

#### 后端：实现 Chunked Server

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "strings"
    "time"
)

// SseEvent - Server-Sent Events 事件格式
type SseEvent struct {
    ID    string
    Event string
    Data  string
}

func (e SseEvent) Format() string {
    var sb strings.Builder
    if e.ID != "" {
        fmt.Fprintf(&sb, "id: %s\n", e.ID)
    }
    if e.Event != "" {
        fmt.Fprintf(&sb, "event: %s\n", e.Event)
    }
    fmt.Fprintf(&sb, "data: %s\n", e.Data)
    sb.WriteString("\n")
    return sb.String()
}

// SSE handler - 典型的流式响应
func sseHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("X-Accel-Buffering", "no") // nginx 禁用缓冲
    
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Not streaming!", 500)
        return
    }
    
    eventID := 0
    tick := time.NewTicker(2 * time.Second)
    defer tick.Stop()
    
    ctx := r.Context()
    for {
        select {
        case <-ctx.Done():
            return
        case t := <-tick.C:
            eventID++
            event := SseEvent{
                ID:    fmt.Sprintf("%d", eventID),
                Event: "message",
                Data:  fmt.Sprintf("Server time: %s | Active connections: %d",
                    t.Format(time.RFC3339), getActiveConnections()),
            }
            fmt.Fprint(w, event.Format())
            flusher.Flush()
        }
    }
}

func getActiveConnections() int {
    return 42 // placeholder
}
```

#### 前端：接收 Chunked 流

```javascript
// 方式1: fetch API（推荐）
async function subscribe() {
    const response = await fetch('/stream', {
        headers: { 'Accept': 'text/event-stream' }
    });
    
    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    
    while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        const text = decoder.decode(value, { stream: true });
        console.log('Received:', text);
    }
}

// 方式2: EventSource（原生支持 SSE）
const evtSource = new EventSource('/stream');
evtSource.onmessage = function(e) {
    console.log('SSE event:', JSON.parse(e.data));
};
evtSource.onerror = function() {
    console.error('Connection lost');
};

// 方式3: Axios (HTTP Client)
import axios from 'axios';

async function fetchStream() {
    const response = await axios.get('/stream', {
        responseType: 'stream'
    });
    
    response.data.on('data', (chunk) => {
        console.log('Data received:', chunk.toString());
    });
}
```

#### Go HTTP Client 消费 Chunked 响应

```go
// Go 客户端自动解码 chunked 响应
resp, err := http.Get("https://example.com/stream")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()

// io.ReadAll 会自动处理 chunked 和 Content-Length
body, err := io.ReadAll(resp.Body)
fmt.Println(string(body))
```

---

### 4. Chunked vs Content-Length 对比

| 维度 | Content-Length | Chunked Transfer-Encoding |
|------|---------------|--------------------------|
| 适用场景 | 已知长度的响应 | 流式/未知长度响应 |
| 性能 | 略优（浏览器预分配） | 略低（需解析块头） |
| 缓冲 | Nginx 可缓存完整响应后转发 | 必须实时转发（不可缓冲） |
| CDN 友好 | ✅ 可缓存 | ❌ 大多数 CDN 不支持 chunked |
| 压缩 | ✅ gzip 后可知长度 | ❌ 动态内容无法压缩后再加 Content-Length |

---

### 5. 常见陷阱

#### 陷阱1：Nginx 缓冲导致 chunked 失效

```nginx
# Nginx 默认会缓冲 chunked 响应直到写完
# 这违背了流式响应的目的！

location /stream {
    # 必须关闭缓冲
    proxy_buffering off;
    
    # 如果代理的是 HTTP/1.0（无 chunked），也需要手动处理
    # proxy_http_version 1.1;
    # proxy_set_header Connection "";
}
```

#### 陷阱2：Kubernetes Ingress 缓冲

```yaml
# 对于流式端点，需要在 Ingress annotation 中禁用缓冲
annotations:
  nginx.ingress.kubernetes.io/proxy-buffering: "off"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"  # 流可能很长
```

#### 陷阱3：Go client 读取 chunked 时的内存

```go
// 小误区：chunked 响应不需要考虑内存
// 如果用 io.ReadAll() 读取 chunked 响应，仍会全部读入内存
// 大数据量流式响应应该用 bufio.Scanner 或逐块读取

buf := make([]byte, 4096)
for {
    n, err := resp.Body.Read(buf)
    if n > 0 {
        process(buf[:n])  // 逐块处理
    }
    if err == io.EOF {
        break
    }
}
```

---

### 6. 与其他协议的关联

```
WebSocket 帧：
[FIN][RSV][Opcode][Mask?][Payload Len][Mask Key][Payload Data...]
                                          ↑ Payload 就是 chunked-like 的消息

gRPC Frame：
[Reserved(1 bit)][Compressed?(1 bit)][Message Length(31 bits)]+[Data...]
                                        ↑ Length-Prefixed 也是 chunked 风格

SSE (Server-Sent Events)：
data: {"status":"ok"}
id: 42
event: message

data: {"status":"error","msg":"timeout"}

                                         ↑ 每行都以 \r\n 分隔，类似于 chunked
```

---

## ❓ 高频追问

**Q：Chunked 编码可以在 HTTP/1.0 中使用吗？**

> 不能。`Transfer-Encoding` 和 `chunked` 只在 HTTP/1.1（RFC 7230）中定义。HTTP/1.0 没有这个特性。不过大多数服务器兼容性地支持它。

**Q：Chunked 编码能配合 gzip 一起使用吗？**

> 可以但有限制。gzip 压缩是在 chunked 之后进行的——服务器先分块发送原始数据，每条消息单独压缩（或全量压缩后分块）。但如果用反向代理（如 Nginx），代理可能会尝试缓冲整个响应来做 gzip 压缩，这会延迟流式输出。此时需要配置 `proxy_http_request_body on` 来避免缓冲。

**Q：chunked 编码对安全性有影响吗？**

> 有一定影响。chunked 允许攻击者通过控制 chunk 长度字段来探测服务器的行为（Slowloris 攻击的部分原理）。另外，chunked 的 trailer headers 可能被用来绕过某些安全检查。现代框架通常已修复这些问题。

---

## 📋 总结 checklist

- [ ] 能说清 chunked 编码解决的核心问题（未知长度的流式响应）
- [ ] 理解 chunked 的格式：HEX 长度 + 数据 + \r\n + 最终空 chunk
- [ ] 能在 Go 中实现流式服务和消费 chunked 响应
- [ ] 知道 chunked 与 Content-Length 的适用场景对比
- [ ] 了解 Nginx/K8s 中对 chunked 响应的配置要点

---

## 📚 延伸阅读

- [RFC 7230 Section 4.1 - Chunked Transfer Coding](https://tools.ietf.org/html/rfc7230#section-4.1)
- [Go net/http - Flush support](https://pkg.go.dev/net/http#Flusher)
- [MDN - HTTP response splitting](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding)
- [Server-Sent Events - WHATWG Spec](https://html.spec.whatwg.org/multipage/server-sent-events.html)
