# Unix Domain Socket (UDS) 与进程间通信

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：Unix Domain Socket、IPC、本地回环 vs UDS、抽象 namespace、socket 权限

---

## 🎯 面试官考察意图

- 是否了解微服务架构中常见的进程间通信方式
- 能否对比 Unix Domain Socket（UDS）和网络 TCP 连接的性能差异
- 是否掌握 Go 中使用 UDS 的实践方法
- 对容器化环境中 UDS 的应用场景的理解

---

## ⚡ 核心答案（30秒）

> **Unix Domain Socket** 是操作系统提供的用于**同一台机器上两个进程**之间高效通信的机制。与 TCP loopback（127.0.0.1）相比，UDS 不需要经过网络协议栈（跳过 IP/TCP 层），减少了数据拷贝和上下文切换，性能约快 **2~5 倍**。
>
> 在 Kubernetes 等容器中常见于 sidecar 代理（如 Envoy）、数据库代理等场景——主进程通过 UDS 将请求转发给同容器的旁路代理处理。

---

## 🔬 深度展开

### 1. UDS vs TCP Loopback 性能对比

```
TCP 连接（127.0.0.1:8080）：
┌──────┐    IP/TCP 协议栈     ┌──────┐
│Client│───────────────▶│Server│
│      │  send→recv(系统调用)  │      │
│      │  IP routing          │      │
│      │  TCP 状态管理         │      │
│      │  checksum 校验        │      │
└──────┘                      └──────┘
耗时：约 50-100μs/次

UDS 连接（/var/run/app.sock）：
┌──────┐       内核管道        ┌──────┐
│Client│═══════ direct copy ══►│Server│
│      │  直接缓冲区到缓冲区拷贝  │      │
│      │  无 IP/TCP 开销         │      │
└──────┘                       └──────┘
耗时：约 10-30μs/次

结论：UDS 比 TCP loopback 快 2~5 倍（低延迟 + 少 CPU 消耗）
```

#### 基准测试参考

| 指标 | TCP loopback | UDS |
|------|-------------|-----|
| RTT | ~80μs | ~20μs |
| CPU 开销 | 较高（协议栈处理） | 极低（纯内存拷贝） |
| 端口占用 | 需要端口号 | 不需要 |
| 适用场景 | 跨主机 / 通用 | 同机进程间 |
| 网络隔离 | 遵循防火墙规则 | 文件权限控制 |

---

### 2. UDS 的两种命名空间

#### 1. 文件系统路径（传统方式）

```go
// 监听
sockPath := "/tmp/my-service.sock"
os.Remove(sockPath) // 清理残留 socket
listener, err := net.Listen("unix", sockPath)

// 连接
conn, err := net.Dial("unix", sockPath)
```

```bash
# 查看 socket 文件
ls -la /tmp/my-service.sock
# srwxr-xr-x  1 root  root  0 Aug 30 03:00 /tmp/my-service.sock
# s = socket file type

# 设置权限（重要！）
chmod 666 /tmp/my-service.sock  # 允许所有用户访问
chmod 777 /tmp/my-service.sock  # 允许所有用户读写执行
```

**注意**：
- socket 文件存在于文件系统中
- 客户端需要知道路径才能连接
- 文件权限控制谁可以连接
- 旧 socket 文件可能残留，需手动 `os.Remove`

#### 2. 抽象 namespace（Abstract Namespace，Linux 特有）

```go
// Linux 特有：以 \x00 开头的名称不使用文件系统
// 进程重启后不会残留文件

sockPath := "\x00my-service-abstract"
listener, err := net.Listen("unix", sockPath)
conn, err := net.Dial("unix", sockPath)
```

**优势**：
- 不创建磁盘文件，更干净
- 不受文件删除影响（即使原进程挂了，名字还在）
- 更安全：没有文件权限问题，只有通过 pid namespace 内的引用才能访问

**限制**：
- 仅 Linux 支持
- 其他系统（macOS/BSD）使用 AF_UNIX 但不同抽象语义
- Go 1.11+ 原生支持

---

### 3. Go 实战示例

#### UDS Echo Server

```go
package main

import (
    "io"
    "log"
    "net"
    "os"
    "time"
)

func main() {
    sockPath := "/tmp/uds-echo-server.sock"
    
    // 清理旧的 socket 文件
    os.Remove(sockPath)
    
    listener, err := net.Listen("unix", sockPath)
    if err != nil {
        log.Fatalf("Listen error: %v", err)
    }
    defer os.Remove(sockPath)
    
    // 设置 socket 文件权限
    os.Chmod(sockPath, 0666)
    
    log.Printf("UDS server listening on %s", sockPath)
    
    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Printf("Accept error: %v", err)
            continue
        }
        
        go handleConn(conn)
    }
}

func handleConn(conn net.Conn) {
    defer conn.Close()
    
    buf := make([]byte, 4096)
    for {
        n, err := conn.Read(buf)
        if err == io.EOF {
            return
        }
        if err != nil {
            log.Printf("Read error: %v", err)
            return
        }
        
        _, err = conn.Write(buf[:n])
        if err != nil {
            log.Printf("Write error: %v", err)
            return
        }
    }
}
```

#### HTTP 服务绑定 UDS

```go
package main

import (
    "fmt"
    "log"
    "net"
    "net/http"
    "os"
)

func main() {
    sockPath := "/tmp/http-server.sock"
    os.Remove(sockPath)
    
    // 创建 Unix domain socket listener
    listener, err := net.Listen("unix", sockPath)
    if err != nil {
        log.Fatalf("Create listener error: %v", err)
    }
    os.Chmod(sockPath, 0666)
    defer os.Remove(sockPath)
    
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello from UDS!")
    })
    
    // 关键：http.Serve 需要 Listener
    server := &http.Server{}
    log.Fatal(server.Serve(listener))
}
```

#### Nginx 转发到 UDS

```nginx
upstream uds_backend {
    server unix:/tmp/http-server.sock;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://uds_backend;
        
        # 传递真实的 RemoteAddr
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # UDS 连接的 buffer 配置
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
}
```

---

### 4. 应用场景

#### 场景1：Kubernetes Sidecar Pattern

```
┌─────────────────────────────────────────┐
│  Pod                                    │
│  ┌──────────┐   UDS   ┌─────────────┐  │
│  │  App     │◄════════►│  Envoy/Proxy │  │
│  │  :8080   │   反向代理  │  :15000     │  │
│  └──────────┘          └─────────────┘  │
│                    ▲                     │
│  外部流量 ──────────┼─────────────────────┘
│                    │
└─────────────────────────────────────────┘
```

- 应用只监听 127.0.0.1:8080 或 UDS
- Sidecar 代理通过 UDS 接收请求并做安全策略、监控
- Istio/Linkerd 广泛使用此模式

#### 场景2：数据库连接代理

```
┌──────────┐   UDS    ┌──────────┐
│ App      │◄════════►│ Proxy    │
│ MySQL    │          │ :3307    │
│ :3306    │          │ (加密/限流)│
└──────────┘          └──────────┘
```

- 应用连接 UDS `/var/run/mysql-proxy.sock`
- Proxy 负责 TLS 终止、连接池管理、SQL 审计
- 应用代码不变，只需改 DSN 从 tcp host:port 改为 unix socket 路径

---

### 5. 安全注意事项

```go
// ✅ 推荐做法
os.Remove(sockPath)       // 防止已有 socket 干扰
os.Chmod(sockPath, 0660)  // 限制访问权限（所有者读写，同组可读）
os.Chown(sockPath, uid, gid) // 设置所有者

// ❌ 危险做法
os.Chmod(sockPath, 0777)  // 所有人都能读写（可能被注入攻击）
// 不清理残留 socket 文件 → 新进程绑定失败
```

```python
# Docker/K8s 最佳实践
# 使用临时目录存放 socket
SOCKET_PATH = "/var/run/my-app/app.sock"
os.makedirs("/var/run/my-app", mode=0o755, exist_ok=True)
```

---

## ❓ 高频追问

**Q：UDS 能跨机器使用吗？**

> 不能。UDS 依赖于同一台机器的文件系统命名空间。跨机器通信必须用 TCP/IP 或其他分布式协议。不过在 K8s 中，通过 `hostNetwork: true` 可以让 Pod 共享宿主机的 network namespace，此时宿主机上的 UDS 可以被多个 Pod 访问。

**Q：Go 中如何优雅关闭 UDS 服务器？**

> ① 先关闭 listener → 阻止新连接；② 等待已有连接处理完毕（设置 deadline 超时）；③ `os.Remove(socketPath)` 清理文件。可以用 `sync.WaitGroup` 追踪活跃连接数。

**Q：为什么 Kubernetes 默认不用 UDS 做 Service？**

> UDS 仅限于单节点内通信。Kubernetes Service 需要跨 Node 负载均衡，所以基于 ClusterIP（虚拟 IP）+ kube-proxy（iptables/ipvs）。但在 Node 本地的 Pod ↔ Pod 通信中，CoreDNS 也使用 UDP/TCP loopback 而非 UDS。

---

## 📋 总结 checklist

- [ ] 理解 UDS 与传统 TCP loopback 的性能差异及原因
- [ ] 知道文件系统命名空间和抽象 namespace 的区别
- [ ] 能在 Go 中实现 UDS server 和 client
- [ ] 了解 UDS 在 Kubernetes sidecar 中的应用
- [ ] 知道 UDS 的安全配置要点

---

## 📚 延伸阅读

- [Go net package - Unix listeners](https://pkg.go.dev/net#DialUnix)
- [UNIX(7) man page](https://man7.org/linux/man-pages/man7/unix.7.html)
- [Linux Abstract Namespace](https://man7.org/linux/man-pages/man7/unix.7.html)
- [Performance of Unix Domain Sockets vs TCP](https://www.freebsd.org/cgi/man.cgi?query=unix&sektion=7&apropos=0&manpath=FreeBSD+13.2-RELEASE)
