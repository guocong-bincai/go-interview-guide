# HTTP 502 / 504 / 499 排查：Go 服务超时链路全景

> 考察频率：★★★★☆  优先级：P1
> 关键词：502 Bad Gateway、504 Gateway Timeout、499 Client Closed Request、Nginx proxy_read_timeout、Go http.Server 超时、context 超时传递、连接池耗尽

---

## 🎯 面试官考察意图

这是**"线上问题排查"类的高频题**，常以这样的形式出现：

> "用户反馈接口偶发 502，怎么排查？"
> "为什么我们服务没报错，但 Nginx 日志全是 504？"
> "Go 服务加了 WriteTimeout 之后，502 反而变多了，为什么？"

面试官想看的是你**从现象定位到层次**的能力：502 是网关的错还是后端的错？504 是后端慢还是网关 timeout 太短？499 是谁关的连接？

**5~8 年工程师的标志**：能画出完整的超时链路（客户端 → LB → Nginx → Go → DB/Redis），指出每一层的时间预算，并说明"超时必须是逐层递减的"。

---

## ⚡ 核心答案（30秒）

| 状态码 | 谁返回的 | 含义 | 常见根因 |
|--------|---------|------|---------|
| **502 Bad Gateway** | 网关/代理（Nginx、LB） | 网关成功连上后端，但没拿到**合法响应** | 后端进程挂了/重启、后端主动断连（超时关连接、SIGTERM 优雅退出不彻底）、协议不匹配（后端说 HTTP/2 网关说 HTTP/1.1）、端口没监听 |
| **504 Gateway Timeout** | 网关/代理 | 网关**连上了**后端，但**等响应超时了** | 后端处理慢（DB 慢查询、下游 RPC 慢、锁等待、GC）、网关 timeout 配太短、后端无限流排队导致处理时间超预算 |
| **499 Client Closed Request** | Nginx（非标准码，Nginx 特有） | **客户端在服务端响应前主动断开了连接** | 客户端超时太短、用户刷新/关页面、上游负载均衡器超时断开、移动端网络抖动 |

**一句话话术：**
> 502 是"后端不听话"（连接断了/响应不合法），504 是"后端太慢"，499 是"客户端等不及跑了"。502/504 是网关视角，499 是客户端视角——三者结合看，能快速判断是后端问题还是链路超时配置问题。

---

## 🔬 深度展开

### 1. 完整链路与超时预算

```
客户端            LB/CDN          Nginx            Go 服务          下游(DB/Redis/RPC)
  │                 │              │                 │                    │
  │──req───────────►│──req────────►│──proxy_pass────►│──query────────────►│
  │  timeout 10s    │ timeout 8s   │ read_timeout 5s │  ctx 3s            │ 2s
  │                 │              │                 │                    │
  │◄──resp──────────│◄──resp───────│◄──resp──────────│◄───resp────────────│
```

**黄金法则：超时必须逐层递减，且外层 > 内层之和 + 网络开销。**

```
客户端 10s  >  LB 8s  >  Nginx 5s  >  Go handler 3s  >  下游 2s
```

**违反这条会怎样：**
- 如果 Go 的 `WriteTimeout` 是 3s 而 Nginx 的 `proxy_read_timeout` 是 60s，Go 会在 3s 时主动断开连接 → Nginx 收到的是"连接被重置"而不是合法响应 → **报 502，而不是 504**。
- 如果 Nginx 是 5s 而 Go 是 30s，Nginx 在 5s 时断开 → Go 继续白跑 25s → 浪费资源，日志里还会出现奇怪的 `write: broken pipe`。

### 2. Go 服务的超时全景（最容易出错的地方）

```go
server := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,   // 读请求头超时（防 Slowloris）
    ReadTimeout:       10 * time.Second,  // 读整个请求（含 body）超时
    WriteTimeout:      15 * time.Second,  // 从读请求头结束开始，写响应超时
    IdleTimeout:       60 * time.Second,  // Keep-Alive 空闲超时
}
```

**各字段真实语义（面试常考）：**

| 字段 | 起点 | 终点 | 坑 |
|------|------|------|-----|
| `ReadHeaderTimeout` | 连接建立 | 读完请求头 | 不设会被 Slowloris 攻击拖垮 |
| `ReadTimeout` | 连接建立 | 读完整个请求 body | 大文件上传会被误杀 |
| `WriteTimeout` | **读完请求头** | 写完响应 | ⚠️ 它的计时**包含 handler 执行时间**！所以它实际是"handler + 写响应"的总预算 |
| `IdleTimeout` | 响应写完 | 收到下一个请求 | 不设则用 `ReadTimeout` |

> ⚠️ **最容易踩的坑**：`WriteTimeout` 从"读完请求头"开始计时，**包含整个 handler 的执行时间**。所以如果你 handler 要跑 30s，`WriteTimeout=15s` 就会在 handler 跑到 15s 时（还没写响应）直接把连接关掉 → **Nginx 侧看到 502**。
>
> 正确做法：`WriteTimeout` 要大于你最慢接口的预期耗时；或者用 `http.TimeoutHandler` / context 做更精细的控制。

**推荐：用 context 逐层传递超时，而不是靠 `WriteTimeout` 兜底**

```go
func withTimeout(timeout time.Duration) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ctx, cancel := context.WithTimeout(r.Context(), timeout)
			defer cancel()

			done := make(chan struct{})
			tw := &timeoutWriter{ResponseWriter: w}

			go func() {
				defer close(done)
				next.ServeHTTP(tw, r.WithContext(ctx))
			}()

			select {
			case <-done:
				// 正常完成
			case <-ctx.Done():
				// 超时：返回 504（让客户端知道是超时，而不是连接被重置）
				tw.mu.Lock()
				if !tw.wroteHeader {
					w.WriteHeader(http.StatusGatewayTimeout)
					// 注意：这里不要写 body，避免和业务 goroutine 竞争
				}
				tw.mu.Unlock()
			}
		})
	}
}

type timeoutWriter struct {
	http.ResponseWriter
	mu          sync.Mutex
	wroteHeader bool
}

func (t *timeoutWriter) WriteHeader(code int) {
	t.mu.Lock()
	defer t.mu.Unlock()
	if !t.wroteHeader {
		t.wroteHeader = true
		t.ResponseWriter.WriteHeader(code)
	}
}
```

> 💡 标准库已经有 `http.TimeoutHandler`，它做的就是这件事。生产里优先用标准库，但要知道它的行为：超时后返回 503（不是 504），并且会调用 `panic(http.ErrAbortHandler)` 中止 handler。

### 3. 502 的六大根因（按线上出现频率排序）

| 排名 | 根因 | 特征 | 修法 |
|------|------|------|------|
| 1 | **后端主动断连（超时/WriteTimeout）** | Nginx error log: `upstream prematurely closed connection` | 检查后端 `WriteTimeout` 是否小于业务耗时 |
| 2 | **后端进程重启/崩溃** | 502 集中在发布时段，或 crash 时 | 优雅重启（先摘流再退出）、liveness probe 配好 |
| 3 | **端口/进程挂了** | 持续 502，`connection refused` | systemd/k8s 自动拉起 + 告警 |
| 4 | **协议不匹配** | 后端说 HTTP/2，Nginx 用 HTTP/1.1 直连 | `proxy_http_version 1.1` + 正确的 `grpc_pass` |
| 5 | **全连接队列溢出**（上题） | 突增流量时 502/超时 | 调 `somaxconn` + 修 accept 循环 |
| 6 | **连接池/端口耗尽** | `Cannot assign requested address` | 复用连接、调 `tcp_tw_reuse` |

**关键排查命令：**

```bash
# Nginx error log 里的关键短语（决定根因）
tail -f /var/log/nginx/error.log | grep -E "502|500|upstream"

# 高频短语对照：
# "upstream prematurely closed connection while reading response header"
#     → 后端主动断连（通常是后端 WriteTimeout 先到）
# "connect() failed (111: Connection refused)"
#     → 后端没监听 / 进程挂了
# "no live upstreams while connecting to upstream"
#     → 所有后端都被健康检查摘除
# "upstream timed out (110: Connection timed out)"
#     → 504，后端处理太慢
# "SSL_do_handshake() failed"
#     → 后端 HTTPS 证书问题
```

```bash
# 定位是哪个后端返回的
# Nginx 日志格式加上 upstream 信息
log_format main '$remote_addr - $upstream_addr $upstream_status '
                '$upstream_response_time $request_time "$request"';
# upstream_status=502 说明后端返回的
# upstream_status="-" 说明连都没连上（connection refused）
# request_time >> upstream_response_time → 是 Nginx 侧慢（如客户端慢）
```

### 4. 504 的排查：找到"慢在哪一层"

```bash
# Nginx 的 upstream_response_time vs request_time
# upstream_response_time = 后端耗时
# request_time = 整个请求耗时（含等待客户端、Nginx 自身处理）
# 如果 upstream_response_time ≈ request_time ≈ 60s → 后端真的慢
# 如果 request_time >> upstream_response_time → 客户端上行慢（如大 body 上传）
```

**后端慢的定位路径：**

```
504 出现
  │
  ├─ 1. Go 服务的 pprof：/debug/pprof/profile?seconds=30
  │     ├─ 热点在 runtime.gcBgMarkWorker → GC 压力大
  │     ├─ 热点在 sync.(*Mutex).Lock → 锁竞争
  │     └─ 热点在 net.(*netFD).Read → 等下游
  │
  ├─ 2. 慢日志 / 链路追踪（TraceID 贯穿）
  │     └─ 看哪个 span 占比最大
  │
  ├─ 3. 下游依赖
  │     ├─ MySQL: 慢查询日志 / SHOW PROCESSLIST / Innodb_row_lock_waits
  │     ├─ Redis: 慢日志 / 大 key 扫描 / 热 key
  │     └─ RPC: 调用方耗时 vs 被调方耗时（差值 = 网络/排队）
  │
  └─ 4. 资源瓶颈
        ├─ CPU 打满 → 请求排队
        ├─ 内存不足 → GC 频繁 / OOM
        ├─ golang.org/x/net/http2: 并发流数超限（MaxConcurrentStreams）
        └─ 数据库连接池耗尽（sql.DB 的 MaxOpenConns 太小）
```

**最常见的 Go 侧 504 根因：`sql.DB` 连接池排队**

```go
db.SetMaxOpenConns(50)     // 太小 → 请求排队等连接
db.SetMaxIdleConns(25)
db.SetConnMaxLifetime(30 * time.Minute) // 不设 → 连接被 MySQL wait_timeout 掐断
```

> 现象特征：DB 本身不慢，但接口 P99 很高，且 `db.Stats().WaitCount` 持续增长。**这是最容易漏的 504 根因之一。**

### 5. 499 的真相：不是服务端的问题

Nginx 的 499 定义：**客户端在 Nginx 返回响应之前关闭了连接。**

```
客户端                    Nginx                    Go 服务
  │──req──────────────────►│──proxy_pass───────────►│
  │                        │                        │ 处理中（很慢）
  │  ✗ 超时，断开连接       │                        │
  │                        │◄────────── resp ───────│ 响应回来时
  │                        │ 发现客户端已走 → 记 499 │
```

**注意**：499 不代表这次请求"失败了"——**后端的业务逻辑可能已经完整执行了**（比如订单已创建）。所以：
- 看到 499 不要直接归因为"客户端问题"，要看看是不是**服务端太慢导致客户端超时**；
- **写接口必须幂等**，否则客户端重试会导致重复下单。

**499 的常见来源：**

| 来源 | 说明 |
|------|------|
| 客户端超时设置太短 | 移动端 SDK 默认 5s，服务端 P99 是 6s |
| 上游 LB 超时 | LB timeout 比 Nginx 短，LB 先断开 |
| 用户主动取消 | 用户关页面、点返回 |
| HTTP/2 客户端取消流 | 发送 `RST_STREAM` |
| 移动网络切换（4G→WiFi） | TCP 连接重建 |

**排查：**

```bash
# Nginx 日志里统计 499 的 URL 分布
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# 对比 upstream_response_time：如果 499 的 upstream_response_time 接近客户端 timeout
# 说明是"服务端太慢导致客户端放弃"
awk '$9 == 499 {print $7, $NF}' /var/log/nginx/access.log | head -20
```

### 6. Nginx 关键超时配置

```nginx
location /api/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Connection "";        # 启用 upstream keepalive

    # 连接建立超时（到后端的 TCP 握手）
    proxy_connect_timeout 3s;
    # 发送请求到后端超时
    proxy_send_timeout    10s;
    # 等待后端响应的超时 ← 504 的主要来源
    proxy_read_timeout    60s;

    # 请求体大小与缓冲
    client_max_body_size 10m;
    proxy_buffering on;                    # 大响应建议开启
    proxy_buffer_size 16k;
    proxy_buffers 8 16k;

    # 后端返回这些状态码时自动重试下一个 upstream
    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_next_upstream_tries 2;
    proxy_next_upstream_timeout 5s;
}
```

> ⚠️ `proxy_next_upstream` 的**重试可能导致写请求重复执行**！对于非幂等的 POST，要么不加 `http_500/502`，要么保证接口幂等。这是很多人不知道的坑。

### 7. Go 服务的优雅关闭（避免发布期间的 502）

```go
func main() {
	srv := &http.Server{Addr: ":8080", Handler: mux}

	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("listen: %v", err)
		}
	}()

	// 等待信号
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
	<-quit
	log.Println("shutting down...")

	// 1) 先停止接受新连接（LB 的健康检查会立刻失败，实现摘流）
	//    但在 K8s 中要注意：摘流和 SIGTERM 是并发的，需要 preStop sleep
	ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
	defer cancel()

	// 2) 等待存量请求处理完
	if err := srv.Shutdown(ctx); err != nil {
		log.Printf("forced shutdown: %v", err)
	}

	// 3) 关闭数据库连接等资源
	db.Close()
	log.Println("server exited")
}
```

**K8s 配套配置（关键！否则依然会 502）：**

```yaml
spec:
  terminationGracePeriodSeconds: 60    # 必须大于 Go 的 25s + preStop 的 10s
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 10"]  # 等 endpoints 摘除传播完成
    readinessProbe:
      httpGet: { path: /readyz, port: 8080 }
      periodSeconds: 2
      failureThreshold: 1
```

> **为什么需要 `preStop: sleep 10`？** K8s 删除 Pod 时，`SIGTERM` 下发和 Endpoints 摘除是**并行**的，而 kube-proxy 更新 iptables 需要时间。这 10 秒让流量先摘干净，再开始关闭进程。

### 8. 状态码排查速查表

| 现象 | 第一反应 | 关键证据 |
|------|---------|---------|
| 502 集中在发布时 | 优雅关闭不彻底 | `preStop` / `Shutdown` 是否配了 |
| 502 + `upstream prematurely closed` | 后端 timeout 比网关短 | 后端 `WriteTimeout` vs Nginx `proxy_read_timeout` |
| 502 持续 + `connection refused` | 后端没起来 | `ss -lnt`、进程状态 |
| 504 偶发 | 某个慢依赖 | TraceID → 慢 span |
| 504 突增 + CPU 正常 | 下游/DB 慢或连接池排队 | `db.Stats().WaitCount`、慢查询 |
| 499 突增 | 客户端超时 or 服务端变慢 | `upstream_response_time` vs 客户端 timeout |
| 502 只在高峰期 | 全连接队列溢出 | `nstat \| grep ListenOverflow` |

---

## ❓ 高频追问

### Q1：为什么加了 `WriteTimeout` 之后 502 变多了？

因为 `WriteTimeout` **从"读完请求头"开始计时，包含 handler 的执行时间**。如果 handler 耗时 > `WriteTimeout`，Go 会在 handler 还没写完响应时直接关掉 TCP 连接。Nginx 收到的是"连接被重置而没有完整响应" → 报 **502**。

**修法**：`WriteTimeout` 设为「最慢接口 P99 + 缓冲」，或者用 context 超时在 handler 内部提前返回 504，把超时变成"合法响应"。

**经验值**：内部网关 → 后端服务的 `WriteTimeout` 建议设为 `proxy_read_timeout` 的 80%，这样超时由后端"体面地"返回 504/503，而不是把连接搞断产生 502。

### Q2：502 和 503 有什么区别？

| 码 | 含义 | 场景 |
|----|------|------|
| 502 | 网关从后端拿到**非法响应**或**连不上** | 进程挂了、断连、协议错 |
| 503 | 服务**暂时不可用**，通常是**主动**返回 | 限流、熔断、维护、`http.TimeoutHandler` 超时、K8s 就绪检查失败 |

503 通常是后端有意返回（带 `Retry-After`），是"我知道我扛不住了"；502 是链路上的意外。

### Q3：客户端超时该设多长？

```
客户端 timeout = 后端 P99.9 耗时 + 网络抖动余量
```

**分场景建议：**

| 接口类型 | 建议超时 |
|---------|---------|
| 纯内存/缓存查询 | 200~500ms |
| 单次 DB 查询 | 1~2s |
| 多次 DB + RPC 编排 | 3~5s |
| 报表/导出类 | 10~60s |
| 同步写操作 | 3~5s（必须幂等） |

**关键原则**：客户端超时 > 服务端超时。否则客户端先放弃，产生大量 499，而服务端还在白干活。

### Q4：如何避免"服务端还在处理但客户端已超时"的资源浪费？

1. **把客户端超时通过 Header 传下来**（如 `X-Request-Timeout` 或标准的 `grpc-timeout`），服务端用 `context.WithTimeout` 对齐；
2. **监听 `r.Context().Done()`**，请求取消时立刻中断下游调用（DB 用 `QueryContext`，RPC 用 ctx 传递）；
3. **写接口保证幂等**，因为客户端重试不可避免。

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	// 客户端断开时 ctx 会 Done，下游查询要带 ctx 才能及时中断
	rows, err := db.QueryContext(ctx, "SELECT ... WHERE id = ?", id)
	if err != nil {
		if errors.Is(err, context.Canceled) {
			// 客户端已离开，无需再返回任何东西
			return
		}
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	defer rows.Close()
	// ...
}
```

### Q5：怎么做 502/504 的监控告警？

| 层级 | 指标 | 告警阈值 |
|------|------|---------|
| 网关 | 5xx 比例、`upstream_response_time` P99 | 5xx > 1% 或 P99 > 阈值 |
| 网关 | `upstream_status` 分布 | 502 突增 |
| 应用 | 请求耗时直方图 P99/P99.9 | 分位数劣化 |
| 应用 | goroutine 数、内存、GC pause | 异常增长 |
| 依赖 | DB `WaitCount`、Redis 慢查询、RPC 错误率 | 非零即关注 |
| 内核 | `ListenOverflows`、`conntrack` 使用率、TIME_WAIT 数 | 持续增长 |

**告警要点**：用**分位数**而不是平均值（平均值会掩盖长尾），并且**按接口维度细分**（整体 1% 不代表核心接口没挂）。

---

## 📋 总结 checklist

- [ ] 能准确区分 502 / 503 / 504 / 499 的返回方和含义
- [ ] 能画出「客户端 → LB → Nginx → Go → 下游」的超时预算图并说明递减原则
- [ ] 知道 `WriteTimeout` 从"读完请求头"开始计时且**包含 handler 执行时间**
- [ ] 知道 Nginx error log 高频短语对应的根因
- [ ] 知道 499 不代表请求未执行，写接口必须幂等
- [ ] 知道 `proxy_next_upstream` 重试对非幂等请求的危险
- [ ] 能说出 K8s 优雅下线为什么需要 `preStop: sleep`
- [ ] 排查时能想到 `sql.DB.WaitCount` 和 `ListenOverflows` 这两个隐藏指标

---

## 📚 延伸阅读

- [Nginx: proxy_read_timeout 与 upstream 相关指令](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [RFC 9110: 502 / 503 / 504 定义](https://www.rfc-editor.org/rfc/rfc9110.html)
- [Go net/http Server 超时字段文档](https://pkg.go.dev/net/http#Server)
- [Nginx 499 状态码的来源](https://nginx.org/en/docs/http/ngx_http_log_module.html)
