# 接口高延迟排查

> 考察频率：★★★★★  难度：★★★★★
> 关键词：pprof、链路追踪、GC 停顿、连接池、火焰图

## 排查思路全景

```
高延迟排查四象限
═══════════════════════════════════════

    外部依赖慢          内部代码慢
   ┌──────────┐       ┌──────────┐
   │ DB/Redis │       │ O(N²)   │
   │ MQ/GRPC  │       │ GC STW  │
   │ 外部 API │       │ 死循环  │
   └──────────┘       └──────────┘

    资源瓶颈            连接/线程
   ┌──────────┐       ┌──────────┐
   │ CPU 打满 │       │ 连接池  │
   │ 内存不足 │       │ goroutine│
   │ 磁盘 IO  │       │ 泄漏    │
   └──────────┘       └──────────┘
```

---

## 1. Go 性能分析工具（pprof）

### 开启 pprof

```go
package main

import (
	"net/http"
	_ "net/http/pprof"  // 导入即开启
)

func main() {
	// pprof 端点默认在 /debug/pprof/
	// 路由已自动注册
	http.ListenAndServe(":6060", nil)
}
```

```bash
# 常用 pprof 端点
/debug/pprof/          # 概览
/debug/pprof/profile   # CPU profile（30秒）
/debug/pprof/heap      # 内存 profile
/debug/pprof/goroutine # goroutine profile
/debug/pprof/block     # 阻塞 profile
/debug/pprof/mutex     # 互斥锁 profile
```

### CPU 耗时分析

```bash
# 1. 采集 30 秒 CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 2. 进入交互式界面
(pprof) top 10       # 查看耗时前10
(pprof) web         # 生成 SVG 火焰图
(pprof) list funcName  # 查看函数源码及耗时
```

```go
// 火焰图解读
// 顶层（top）是调用栈根部，越宽越耗时
// 从下往上看调用链

// 示例：某个接口 CPU 100%
// main.handleRequest    ████████████████████ 100ms
//   main.queryDB        ████████████████      80ms
//     main.process      ████████              50ms
//       db.scan         ██████                30ms
```

### 内存分析

```bash
# 采集内存 profile
go tool pprof http://localhost:6060/debug/pprof/heap

# 查看分配最多的对象
(pprof) top -inuse_objects

# 输出示例：
# Active Samples:
#           Flat  Flat%   Sum%  Cum Cum%  Func
#        2.51GB 42.13% 42.13%  2.51GB 42.13%  main.unmarshalJSON
#        1.80GB 30.23% 72.36%  4.31GB 72.36%  main.processEvents
```

### Goroutine 阻塞分析

```bash
# goroutine profile（看有没有泄漏）
go tool pprof http://localhost:6060/debug/pprof/goroutine

# block profile（看阻塞在哪里）
go tool pprof http://localhost:6060/debug/pprof/block

# mutex profile（看互斥锁争用）
go tool pprof http://localhost:6060/debug/pprof/mutex
```

---

## 2. GC 停顿排查

### Go GC 原理回顾

Go 使用**三色标记 + 写屏障**，STW（Stop The World）时间已从秒级降到亚毫秒级。

```go
// 查看 GC 统计
import (
	"runtime/debug"
)

func init() {
	// 开启 GC trace
	debug.SetGCPercent(100) // 默认，GC 触发时机
}

// 程序启动时加参数也可以
// GODEBUG=gctrace=1 ./app
```

```bash
# 运行时查看 GC 状态
GODEBUG=gctrace=1 go run main.go

# 输出示例：
# gc 1 @0.001s 0%: 0.018+0.21+0.002 ms clock, 0.14+0.12/0.15/0.11+0.016 ms cpu
# gc 2 @0.009s 0%: 0.013+0.19+0.002 ms clock, 0.10+0.10/0.14/0.10+0.015 ms cpu
# gc 3 @0.014s 0%: 0.012+0.21+0.002 ms clock, 0.10+0.11/0.17/0.11+0.016 ms cpu

# 字段解释：
# gc N @时间: GC次数 @启动后时间
# clock: STW时间 + GC标记时间 + 写屏障时间
# cpu: 各阶段 CPU 时间
```

### GC 导致的延迟

```go
// 问题代码：频繁小对象分配导致 GC 压力大
func badExample() {
	var result []int
	for i := 0; i < 10000; i++ {
		// 每次循环创建新对象，触发 GC
		data := process(i)  // 返回新 struct
		result = append(result, data.Value)
	}
}

// 优化：对象池 + 减少分配
func goodExample() {
	pool := sync.Pool{
		New: func() interface{} {
			return &Result{}
		},
	}
	
	var result []int
	for i := 0; i < 10000; i++ {
		obj := pool.Get().(*Result)
		processInto(obj, i)  // 复用对象
		result = append(result, obj.Value)
		pool.Put(obj)  // 放回池子
	}
}
```

### 强制 GC + 内存分析

```go
// 定期采集内存 baseline，对比增长
package main

import (
	"log"
	"runtime"
	"time"
)

func monitorMemory() {
	var m1, m2 runtime.MemStats
	runtime.ReadMemStats(&m1)
	
	ticker := time.NewTicker(10 * time.Second)
	for range ticker.C {
		runtime.ReadMemStats(&m2)
		
		allocatedDiff := m2.Alloc - m1.Alloc
		gcDiff := m2.NumGC - m1.NumGC
		
		log.Printf("Alloc: %d KB, GC: %d times (interval)",
			allocatedDiff/1024, gcDiff)
		
		m1 = m2
	}
}
```

---

## 3. 连接池问题

### DB 连接池

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"time"

	_ "github.com/go-sql-driver/mysql"
)

// 连接池配置不当导致的延迟
func dbPoolIssue() {
	db, err := sql.Open("mysql", "user:pass@tcp(localhost:3306)/db")
	if err != nil {
		panic(err)
	}
	
	// 连接池配置
	db.SetMaxOpenConns(100)         // 最大打开连接数
	db.SetMaxIdleConns(10)          // 空闲连接数
	db.SetConnMaxLifetime(time.Hour) // 连接最大生命周期
	db.SetConnMaxIdleTime(10 * time.Minute) // 空闲超时
	
	// 问题1：连接池耗尽（100个并发请求，第101个等待）
	// 表现：接口超时，db.Stats().InUse > 0
	
	// 问题2：连接泄露（忘记 Close）
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	
	row := db.QueryRowContext(ctx, "SELECT * FROM users WHERE id=?", 1)
	var name string
	if err := row.Scan(&name); err != nil {
		// row 不要 defer Close，sql.Row 自动管理
	}
	
	fmt.Println(name)
}
```

### Redis 连接池

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

func redisPoolIssue() {
	rdb := redis.NewClient(&redis.Options{
		PoolSize:     100,            // 连接池大小
		MinIdleConns: 10,             // 最小空闲连接
		PoolTimeout:  3 * time.Second, // 获取连接超时
		ReadTimeout:  time.Second,    // 读超时
		WriteTimeout: time.Second,   // 写超时
	})
	
	ctx := context.Background()
	
	// 问题：Pipeline 滥用
	// 每个命令单独一次 RTT，应该用 Pipeline 合并
	
	// 错误用法：10次网络往返
	for i := 0; i < 10; i++ {
		rdb.Set(ctx, fmt.Sprintf("key%d", i), "value", 0)
	}
	
	// 正确用法：1次网络往返
	pipe := rdb.Pipeline()
	for i := 0; i < 10; i++ {
		pipe.Set(ctx, fmt.Sprintf("key%d", i), "value", 0)
	}
	_, err := pipe.Exec(ctx)
	if err != nil {
		panic(err)
	}
}
```

---

## 4. 链路追踪（Distributed Tracing）

### OpenTelemetry + Go

```go
package main

import (
	"context"
	"fmt"
	"log"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/jaeger"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.4.0"
	"go.opentelemetry.io/otel/trace"
)

func initTracer() (func(), error) {
	// Jaeger exporter
	exp, err := jaeger.New(jaeger.WithCollectorEndpoint(
		jaeger.WithEndpoint("http://localhost:14268/api/traces"),
	))
	if err != nil {
		return nil, err
	}
	
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exp),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceNameKey.String("order-service"),
		)),
	)
	
	otel.SetTracerProvider(tp)
	return func() { tp.Shutdown(context.Background()) }, nil
}

func handler(ctx context.Context, orderID string) error {
	// 从 context 提取 trace
	tracer := otel.Tracer("order-service")
	ctx, span := tracer.Start(ctx, "handleOrder",
		trace.WithAttributes(
			attribute.String("order.id", orderID),
		),
	)
	defer span.End()
	
	// 查询数据库
	ctx, dbSpan := tracer.Start(ctx, "queryDB")
	err := queryDB(ctx, orderID)
	dbSpan.End()
	if err != nil {
		span.RecordError(err)
		return err
	}
	
	// 发 MQ
	ctx, mqSpan := tracer.Start(ctx, "sendMQ")
	err = sendMQ(ctx, orderID)
	mqSpan.End()
	
	return nil
}

func queryDB(ctx context.Context, id string) error {
	// 模拟 DB 调用
	return nil
}

func sendMQ(ctx context.Context, id string) error {
	// 模拟 MQ 调用
	return nil
}

func main() {
	shutdown, err := initTracer()
	if err != nil {
		log.Fatal(err)
	}
	defer shutdown()
	
	ctx := context.Background()
	if err := handler(ctx, "order-001"); err != nil {
		log.Printf("Handler error: %v", err)
	}
}
```

### 常见延迟链路分析

```bash
# Jaeger UI 查看链路
# http://localhost:16686

# 典型延迟链路
# handleOrder (100ms)
#   ├── queryDB (80ms)     ← DB 慢
#   │     └── scan (75ms)  ← 全表扫描
#   └── sendMQ (15ms)
#         └── redis SET (10ms) ← 网络抖动
```

---

## 5. 实战排查流程

```go
// 标准排查入口
package main

import (
	"fmt"
	"net/http"
	_ "net/http/pprof"
	"runtime"
	"runtime/debug"
	"time"
)

func diagnose(endpoint string) {
	// 1. 查 GC
	debug.SetGCPercent(100)
	
	// 2. 查 goroutine 数量（泄漏？）
	var stats runtime.MemStats
	runtime.ReadMemStats(&stats)
	fmt.Printf("Goroutines: %d\n", runtime.NumGoroutine())
	fmt.Printf("Memory: Alloc=%d KB, Sys=%d KB, GC=%d\n",
		stats.Alloc/1024, stats.Sys/1024, stats.NumGC)
	
	// 3. HTTP 客户端超时
	client := &http.Client{
		Timeout: 3 * time.Second,
	}
	
	// 4. 链路追踪
	// 结合 OpenTelemetry 标记 span
}
```

### 常见延迟原因速查表

| 症状 | 可能原因 | 排查工具 |
|------|---------|---------|
| 偶发高延迟 | GC STW | `GODEBUG=gctrace=1` |
| 持续高延迟 | CPU 打满 | `pprof/profile` |
| 内存持续增长 | 内存泄漏 | `pprof/heap` |
| goroutine 暴涨 | goroutine 泄漏 | `pprof/goroutine` |
| DB 请求慢 | 缺索引/连接池 | `EXPLAIN` + 连接池统计 |
| Redis 请求慢 | bigkey/网络 | `slowlog` |
| 外部 API 慢 | 超时/重试 | 链路追踪 |

---

## 面试话术

**Q：接口 P99 延迟高，怎么排查？**

> P99 高说明有少量请求特别慢，通常是 GC 停顿、外部依赖抖动、或者代码里有 O(N²) 的角落。先用 pprof 采 CPU 和 goroutine profile，看有没有异常的函数耗时；如果持续高，看内存 profile 是否有泄漏；如果偶发，加链路追踪看是不是下游依赖的 P99 拖累了。Go 的话配合 `GODEBUG=gctrace=1` 看 GC 情况。

**Q：Go 的 GC 停顿能完全消除吗？**

> 不能完全消除，但 Go 1.19+ 的 GC STW 已经做到亚毫秒级（<100μs），对大多数应用影响不大。如果 GC 停顿真的影响到了 SLA（比如金融交易系统），可以考虑：1）减少对象分配频率（sync.Pool）；2）用 `runtime.GC()` 主动触发 + 控制时间；3）看是不是用了 `debug.SetGCPercent` 把阈值调太高；4）考虑换 Go 版本，新版本 GC 持续优化。

**Q：连接池耗尽是什么现象？怎么处理？**

> 现象是请求 hang 住不动，goroutine 在等 DB/Redis 连接，`db.Stats().InUse` 数量等于 `MaxOpenConns`。处理：1）先加连接池上限；2）排查慢查询占着连接不放；3）检查有没有泄露（没 Close）；4）加超时（`SetConnMaxLifetime`）；5）最终方案是做读写分离 + 连接池隔离。
