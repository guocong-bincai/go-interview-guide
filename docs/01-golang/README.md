# 01 · Go 语言深度

> 考察频率：★★★★★  优先级：P0

## 文章清单

### 01-runtime · 运行时原理
- [✅] `01-gmp.md` — GMP 调度模型：G/M/P 的角色、work stealing、hand off
- [✅] `02-gc.md` — GC 三色标记、混合写屏障、STW 优化历程
- [✅] `03-memory-alloc.md` — Go 内存分配：mspan/mcache/mcentral/mheap
- [✅] `04-stack.md` — goroutine 栈增长、栈缩容、连续栈 vs 分段栈
- [⏳] `05-scheduler-source.md` — Go 调度器源码走读：schedule()/findRunnable()/sysmon（**P0 待认领**）

### 02-concurrency · 并发编程
- [✅] `01-channel.md` — channel 底层结构、发送/接收流程、select 实现
- [✅] `02-sync.md` — Mutex/RWMutex 实现、sync.Once、sync.Pool
- [✅] `03-atomic.md` — atomic 原理、CAS、无锁数据结构
- [✅] `04-patterns.md` — 并发模式：Pipeline、Fan-out/Fan-in、errgroup
- [✅] `05-context.md` — context 底层、取消传播、超时控制

### 03-language-deep · 语言机制
- [✅] `01-interface.md` — interface 内存布局、iface/eface、动态分发开销
- [✅] `02-reflect.md` — reflect 原理、性能代价、实际使用场景
- [✅] `03-generics.md` — 泛型用法与实现原理（GCShape Stenciling、性能对比）
- [✅] `04-escape.md` — 逃逸分析、堆 vs 栈分配、如何避免不必要逃逸
- [✅] `05-slice-map.md` — slice/map 底层结构、扩容策略、并发安全问题
- [✅] `06-memory-model.md` — Go 内存模型、happens-before、内存对齐
- [✅] `07-loop-iterators.md` — Go 1.22 循环变量语义变更、range-over-int、Go 1.23 range-over-func
- [⏳] `08-compiler-optimize.md` — 内联决策、逃逸深度、BCE、常量折叠（**P1 待认领**）

### 04-performance · 性能调优
- [✅] `01-pprof.md` — pprof 使用、CPU/内存/goroutine 火焰图分析
- [✅] `02-benchmark.md` — 基准测试规范、避免编译器优化干扰
- [✅] `03-memory-leak.md` — 内存泄漏排查：goroutine 泄漏、全局变量、缓存失控
- [✅] `04-tuning-cases.md` — 真实调优案例：JSON 解析、字符串拼接、sync.Pool 实战
