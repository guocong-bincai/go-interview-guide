# 01 · Go 语言深度

> 考察频率：★★★★★  优先级：P0

---

## 01-runtime · 运行时原理

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `01-gmp.md` | GMP 调度模型：G/M/P 的角色、work stealing、hand off |
| ✅ | `02-gc.md` | GC 三色标记、混合写屏障、STW 优化历程 |
| ✅ | `03-memory-alloc.md` | Go 内存分配：mspan/mcache/mcentral/mheap |
| ✅ | `04-stack.md` | goroutine 栈增长、栈缩容、连续栈 vs 分段栈 |
| ✅ | `05-scheduler-source-code.md` | Go 调度器源码走读：schedule()/findRunnable()/sysmon |
| ✅ | `09-io-multiplexing.md` | epoll/kqueue 与 Netpoller 原理、Go 网络 I/O 模型 |
| ✅ | `11-goroutine-lifecycle.md` | goroutine 创建/运行/阻塞/退出全生命周期、泄漏原因、pprof/trace 排查 |
| ✅ | `11-gc-tuning.md` | GOGC/GOMEMLIMIT 调优、Green Tea GC、生产问题排查 |

---

## 02-concurrency · 并发编程

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `01-channel.md` | channel 底层结构、发送/接收流程、select 实现 |
| ✅ | `02-sync.md` | Mutex/RWMutex 实现、sync.Once、sync.Pool |
| ✅ | `03-atomic.md` | atomic 原理、CAS、无锁数据结构 |
| ✅ | `04-patterns.md` | 并发模式：Pipeline、Fan-out/Fan-in、errgroup |
| ✅ | `05-context.md` | context 底层、取消传播、超时控制 |
| ✅ | `08-goroutine-vs-thread.md` | goroutine vs OS 线程：栈大小、调度开销、为什么轻量 |
| ✅ | `deadlock.md` | 死锁四个条件、Coffman 条件、channel/Mutex 死锁、活锁、go-deadlock 排查 |
| ✅ | `race-condition.md` | data race 产生条件、`-race` 检测、常见场景 |

---

## 03-language-deep · 语言机制

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `01-interface.md` | interface 内存布局、iface/eface、动态分发开销 |
| ✅ | `02-reflect.md` | reflect 原理、性能代价、实际使用场景 |
| ✅ | `03-generics.md` | 泛型用法与实现原理（GCShape Stenciling、性能对比） |
| ✅ | `04-escape.md` | 逃逸分析、堆 vs 栈分配、如何避免不必要逃逸 |
| ✅ | `05-slice-map.md` | slice/map 底层结构、扩容策略、并发安全问题 |
| ✅ | `06-memory-model.md` | Go 内存模型、happens-before、内存对齐 |
| ✅ | `07-loop-iterators.md` | Go 1.22 循环变量语义变更、range-over-int、Go 1.23 range-over-func |
| ✅ | `08-compiler-optimize.md` | 内联决策、逃逸深度、BCE、常量折叠 |
| ✅ | `13-cgo.md` | CGO 调用链路、`CGO_ENABLED=0`、性能开销 |
| ✅ | `14-unsafe.md` | unsafe.Pointer、uintptr 区别、正确使用姿势 |
| ✅ | `18-string-byte.md` | string 底层结构、string↔[]byte 零拷贝、为什么不可变 |
| ✅ | `17-defer.md` | defer 执行顺序、defer+panic+recover 组合、性能损耗 |
| ✅ | `19-closure.md` | 闭包捕获变量原理、循环变量陷阱（Go 1.22 前后对比） |
| ✅ | `20-make-new.md` | make/new 区别、零值初始化、适用类型 |
| ✅ | `22-go1.27-generic-methods.md` | Go 1.27 泛型方法批准、设计哲学、与 Java/C++ 对比 |
| ✅ | `15-init.md` | init 执行顺序、多包 init 依赖关系、init vs 全局变量 |
| ✅ | `23-nil-vs-empty-slice.md` | nil slice vs empty slice：JSON 序列化差异、append 行为、零值语义（新增）|

---

## 04-performance · 性能调优

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `01-pprof.md` | pprof 使用、CPU/内存/goroutine 火焰图分析 |
| ✅ | `02-memory-leak.md` | 内存泄漏排查：goroutine 泄漏、全局变量、缓存失控 |
| ✅ | `03-benchmark.md` | 基准测试规范、避免编译器优化干扰 |
| ✅ | `04-tuning-cases.md` | 真实调优案例：JSON 解析、字符串拼接、sync.Pool 实战 |

---

## 05-stdlib · 标准库

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `01-net-http.md` | http.Server 底层结构、连接复用、超时控制 |
| ✅ | `02-sync-map.md` | sync.Map 设计原理、适用场景、与 RWMutex 对比 |
| ✅ | `03-errors.md` | errors.Is/As、错误链包装、自定义错误类型、panic/recover 边界 |
| ✅ | `04-slog.md` | log/slog 结构化日志、Handler/Context 链路、与 zap 对比 |
| ✅ | `05-embed.md` | go:embed 底层机制、内存布局、与构建标签联动 |
| ✅ | `io-reader-writer.md` | io.Reader/Writer 设计模式、流式处理、bufio、Decorator 模式 |
| ✅ | `time.md` | time.Timer/Ticker 正确用法、Stop/Reset 陷阱、内存泄漏、时区处理 |
| ✅ | `18-string-byte.md` | string 底层结构、string↔[]byte 零拷贝、为什么不可变（strings.Builder vs bytes.Buffer） |

---

## 06-toolchain · 工程工具链

| 状态 | 文件 | 考点 |
|---|---|---|
| ✅ | `01-go-module.md` | go.mod 核心语义、MVS 版本选择算法、私有模块、go.work |
| ✅ | `02-build-tags.md` | Build Tags 语法、常用内置 Tag、自定义 Tag 实战 |

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
