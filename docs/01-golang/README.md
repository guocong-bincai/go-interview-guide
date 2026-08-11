# 01 · Go 语言深度

> 考察频率：★★★★★  优先级：P0

---

## 01-runtime · 运行时原理

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `31-01-gmp.md` | [GMP 调度模型：G/M/P 的角色、work stealing、hand off](01-runtime/31-01-gmp.md) |
| ✅ | `32-02-gc.md` | [GC 三色标记、混合写屏障、STW 优化历程](01-runtime/32-02-gc.md) |
| ✅ | `33-03-memory-alloc.md` | [Go 内存分配：mspan/mcache/mcentral/mheap](01-runtime/33-03-memory-alloc.md) |
| ✅ | `34-04-stack.md` | [goroutine 栈增长、栈缩容、连续栈 vs 分段栈](01-runtime/34-04-stack.md) |
| ✅ | `35-05-scheduler-source-code.md` | [Go 调度器源码走读：schedule()/findRunnable()/sysmon](01-runtime/35-05-scheduler-source-code.md) |
| ✅ | `10-09-io-multiplexing.md` | [epoll/kqueue 与 Netpoller 原理、Go 网络 I/O 模型](01-runtime/10-09-io-multiplexing.md) |
| ✅ | `11-11-goroutine-lifecycle.md` | [goroutine 创建/运行/阻塞/退出全生命周期、泄漏原因、pprof/trace 排查](01-runtime/11-11-goroutine-lifecycle.md) |
| ✅ | `37-11-gc-tuning.md` | [GOGC/GOMEMLIMIT 调优、Green Tea GC、生产问题排查](01-runtime/37-11-gc-tuning.md) |

---

## 02-concurrency · 并发编程

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `38-01-channel.md` | [channel 底层结构、发送/接收流程、select 实现](02-concurrency/38-01-channel.md) |
| ✅ | `40-02-sync.md` | [Mutex/RWMutex 实现、sync.Once、sync.Pool](02-concurrency/40-02-sync.md) |
| ✅ | `41-03-atomic.md` | [atomic 原理、CAS、无锁数据结构](02-concurrency/41-03-atomic.md) |
| ✅ | `02-04-patterns.md` | [并发模式：Pipeline、Fan-out/Fan-in、errgroup](02-concurrency/02-04-patterns.md) |
| ✅ | `42-05-context.md` | [context 底层、取消传播、超时控制](02-concurrency/42-05-context.md) |
| ✅ | `44-08-goroutine-vs-thread.md` | [goroutine vs OS 线程：栈大小、调度开销、为什么轻量](02-concurrency/44-08-goroutine-vs-thread.md) |
| ✅ | `45-deadlock.md` | [死锁四个条件、Coffman 条件、channel/Mutex 死锁、活锁、go-deadlock 排查](02-concurrency/45-deadlock.md) |
| ✅ | `46-race-condition.md` | [data race 产生条件、`-race` 检测、常见场景](02-concurrency/46-race-condition.md) |
| ✅ | `77-channel-vs-mutex.md` | [Channel vs Mutex：何时用哪个？Go Proverbs 实战解读](02-concurrency/77-channel-vs-mutex.md) |
| ✅ | `78-panic-recover.md` | [Panic / Recover 正确姿势：异常处理边界、HTTP Handler 兜底、goroutine 保护](02-concurrency/78-panic-recover.md) |
| ✅ | `79-context-value-pitfalls.md` | [Context Value 传递陷阱：key 类型、超时管理、反模式与最佳实践](02-concurrency/79-context-value-pitfalls.md) |
| ✅ | `79-08-waitgroup.md` | [WaitGroup：协程同步计数器，Add/Done/Wait 机制与常见陷阱](02-concurrency/79-08-waitgroup.md) |
| ✅ | `80-functional-options.md` | [函数选项模式（Functional Options）：API 优雅设计、Option 接口实现](02-concurrency/80-functional-options.md) |
| ✅ | `80-pool-best-practices.md` | [sync.Pool 最佳实践与误区：对象池设计哲学、New回调、Get/Put生命周期](02-concurrency/80-pool-best-practices.md) |

---

## 03-language-deep · 语言机制

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `03-01-interface.md` | [interface 内存布局、iface/eface、动态分发开销](03-language-deep/03-01-interface.md) |
| ✅ | `47-02-reflect.md` | [reflect 原理、性能代价、实际使用场景](03-language-deep/47-02-reflect.md) |
| ✅ | `48-03-generics.md` | [泛型用法与实现原理（GCShape Stenciling、性能对比）](03-language-deep/48-03-generics.md) |
| ✅ | `49-04-escape.md` | [逃逸分析、堆 vs 栈分配、如何避免不必要逃逸](03-language-deep/49-04-escape.md) |
| ✅ | `50-05-slice-map.md` | [slice/map 底层结构、扩容策略、并发安全问题](03-language-deep/50-05-slice-map.md) |
| ✅ | `14-06-memory-model.md` | [Go 内存模型、happens-before、内存对齐](03-language-deep/14-06-memory-model.md) |
| ✅ | `51-07-loop-iterators.md` | [Go 1.22 循环变量语义变更、range-over-int、Go 1.23 range-over-func](03-language-deep/51-07-loop-iterators.md) |
| ✅ | `52-08-compiler-optimize.md` | [内联决策、逃逸深度、BCE、常量折叠](03-language-deep/52-08-compiler-optimize.md) |
| ✅ | `57-13-cgo.md` | [CGO 调用链路、`CGO_ENABLED=0`、性能开销](03-language-deep/57-13-cgo.md) |
| ✅ | `16-14-unsafe.md` | [unsafe.Pointer、uintptr 区别、正确使用姿势](03-language-deep/16-14-unsafe.md) |
| ✅ | `60-18-string-byte.md` | [string 底层结构、string↔[]byte 零拷贝、为什么不可变](03-language-deep/60-18-string-byte.md) |
| ✅ | `04-17-defer.md` | [defer 执行顺序、defer+panic+recover 组合、性能损耗](03-language-deep/04-17-defer.md) |
| ✅ | `17-19-closure.md` | [闭包捕获变量原理、循环变量陷阱（Go 1.22 前后对比）](03-language-deep/17-19-closure.md) |
| ✅ | `61-20-make-new.md` | [make/new 区别、零值初始化、适用类型](03-language-deep/61-20-make-new.md) |
| ✅ | `18-22-go1.27-generic-methods.md` | [Go 1.27 泛型方法批准、设计哲学、与 Java/C++ 对比](03-language-deep/18-22-go1.27-generic-methods.md) |
| ✅ | `58-15-init.md` | [init 执行顺序、多包 init 依赖关系、init vs 全局变量](03-language-deep/58-15-init.md) |
| ✅ | `19-23-nil-vs-empty-slice.md` | [nil slice vs empty slice：JSON 序列化差异、append 行为、零值语义（新增）](03-language-deep/19-23-nil-vs-empty-slice.md) |
| ✅ | `78-nil-vs-closed-channel.md` | [nil Channel vs Closed Channel：发送接收行为差异、select default 检测、goroutine 泄漏防护](03-language-deep/78-nil-vs-closed-channel.md) |
| ✅ | `80-interface-composition-vs-embedding.md` | [Interface 组合 vs 嵌入：Method Set 规则、隐式实现、依赖注入最佳实践](03-language-deep/80-interface-composition-vs-embedding.md) |

---

## 04-performance · 性能调优

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `05-01-pprof.md` | [pprof 使用、CPU/内存/goroutine 火焰图分析](04-performance/05-01-pprof.md) |
| ✅ | `65-02-memory-leak.md` | [内存泄漏排查：goroutine 泄漏、全局变量、缓存失控](04-performance/65-02-memory-leak.md) |
| ✅ | `66-03-benchmark.md` | [基准测试规范、避免编译器优化干扰](04-performance/66-03-benchmark.md) |
| ✅ | `79-false-sharing.md` | [False Sharing（伪共享）：CPU 缓存行对齐、padding 技巧与多核性能优化](04-performance/79-false-sharing.md) |
| ✅ | `20-04-tuning-cases.md` | [真实调优案例：JSON 解析、字符串拼接、sync.Pool 实战](04-performance/20-04-tuning-cases.md) |

---

## 05-stdlib · 标准库

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `22-01-net-http.md` | [http.Server 底层结构、连接复用、超时控制](05-stdlib/22-01-net-http.md) |
| ✅ | `23-02-sync-map.md` | [sync.Map 设计原理、适用场景、与 RWMutex 对比](05-stdlib/23-02-sync-map.md) |
| ✅ | `24-03-errors.md` | [errors.Is/As、错误链包装、自定义错误类型、panic/recover 边界](05-stdlib/24-03-errors.md) |
| ✅ | `25-04-slog.md` | [log/slog 结构化日志、Handler/Context 链路、与 zap 对比](05-stdlib/25-04-slog.md) |
| ✅ | `68-05-embed.md` | [go:embed 底层机制、内存布局、与构建标签联动](05-stdlib/68-05-embed.md) |
| ✅ | `63-io-reader-writer.md` | [io.Reader/Writer 设计模式、流式处理、bufio、Decorator 模式](03-language-deep/63-io-reader-writer.md) |
| ✅ | `30-13-time.md` | [time.Timer/Ticker 正确用法、Stop/Reset 陷阱、内存泄漏、时区处理](05-stdlib/30-13-time.md) |
| ✅ | `60-18-string-byte.md` | [string 底层结构、string↔[]byte 零拷贝、为什么不可变（strings.Builder vs bytes.Buffer）](03-language-deep/60-18-string-byte.md) |
| ✅ | `79-graceful-shutdown.md` | [Graceful Shutdown：http.Server.Shutdown、信号处理、draining、K8s 配合](05-stdlib/79-graceful-shutdown.md) |

---

## 06-toolchain · 工程工具链

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `72-01-go-module.md` | [go.mod 核心语义、MVS 版本选择算法、私有模块、go.work](06-toolchain/72-01-go-module.md) |
| ✅ | `73-02-build-tags.md` | [Build Tags 语法、常用内置 Tag、自定义 Tag 实战](06-toolchain/73-02-build-tags.md) |

---

## 🔮 前沿/扩展（加分项，非面试高频）

> 以下内容为 Go 1.24~1.27 前沿特性，了解即可，不是面试必背内容。

| 文件 | 版本 | 内容 |
|---|---|---|
| `01-runtime/06-go1.26-runtime.md` | Go 1.26 | Green Tea GC / Goroutine Leak Profile |
| `01-runtime/07-flight-recorder.md` | Go 1.25 | trace.FlightRecorder 生产级 trace |
| `01-runtime/08-go1.25-gomaxprocs.md` | Go 1.25 | 容器自动感知 GOMAXPROCS |
| `03-language-deep/09-new-function.md` | Go 1.26 | new() 内置函数增强 |
| `03-language-deep/10-go1.26-stack-alloc.md` | Go 1.26 | append 推测性栈缓冲优化 |
| `03-language-deep/11-go1.24-swiss-tables.md` | Go 1.24 | Map 底层换用 Swiss Tables |
| `03-language-deep/12-type-construction.md` | Go 1.26 | 类型构造与循环检测编译器改进 |
| `04-performance/05-synctest.md` | Go 1.25 | testing/synctest 并发测试 |
| `04-performance/06-go-fix-inline.md` | Go 1.26 | go fix 与 //go:fix inline |
| `05-stdlib/06-go1.27-stdlib.md` | Go 1.27 | CutLast / Response Body Drain 等新 API |
| `05-stdlib/07-go1.26-crypto.md` | Go 1.26 | ML-KEM / HPKE 后量子密码学 |
| `05-stdlib/08-json-v2.md` | Go 1.25 | encoding/json v2 实验性 API |
