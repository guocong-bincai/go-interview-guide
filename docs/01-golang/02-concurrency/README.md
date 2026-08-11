# 02 Concurrency 模块

> 📌 共 13 道高频面试题 ｜ ✅ 已按面试频率排序（★★★★★ → ★☆☆☆）

---

## 📋 题目索引（点击直接跳转阅读）

| 序号 | 📄 文件名 | 🔥 频率 | 💡 考点 & 跳转 |
|---|---|---|---|
| 01 | `02-04-patterns.md` | `★★★★★` | [Go 并发模式：Pipeline、Fan-out/Fan-in、errgroup](./02-04-patterns.md) · 键词：Pipeline、Fan-out/Fan-in、errgroup、select、超时控制
| 02 | `13-06-singleflight.md` | `★★★★☆` | [SingleFlight：并发去重与请求合并](./13-06-singleflight.md) · 键词：SingleFlight、请求合并、并发去重、缓存击穿、golang.org/x/sync/s
| 03 | `79-08-waitgroup.md` | `★★★★★` | [WaitGroup：协程同步计数器，Add/Done/Wait 机制与常见陷阱](./79-08-waitgroup.md) · 关键词：WaitGroup、counter 泄露、error propagation、errgroup
| 04 | `38-01-channel.md` | `★★★★☆` | [Channel 底层原理：hchan 结构、环形队列、sendq/recvq](./38-01-channel.md)
| 05 | `80-pool-best-practices.md` | `★★★★☆` | [sync.Pool 最佳实践与常见误区：对象池设计哲学与正确使用场景](./80-pool-best-practices.md) · 关键词：sync.Pool、New 回调、Get/Put 生命周期
| 06 | `39-02-cond.md` | `★★★☆☆` | [sync.Cond：条件变量与 Go 1.26 Breaking Change](./39-02-cond.md) · 键词：Cond、Wait、Signal、Broadcast、Go 1.26 breaking cha
| 07 | `40-02-sync.md` | `★★★☆☆` | [sync 原语：Mutex / RWMutex / Once / Pool](./40-02-sync.md)
| 08 | `41-03-atomic.md` | `★★★☆☆` | [atomic 与无锁编程：CAS、atomicValue、lock-free](./41-03-atomic.md)
| 09 | `42-05-context.md` | `★★★☆☆` | [context 原理：取消传播、超时控制与层级树](./42-05-context.md)
| 10 | `43-07-select.md` | `★★★☆☆` | [Go Select 深度解析：底层实现、源码解读与生产实践](./43-07-select.md)
| 11 | `44-08-goroutine-vs-thread.md` | `★★★☆☆` | [Goroutine vs OS 线程：核心差异与调度开销](./44-08-goroutine-vs-thread.md)
| 12 | `45-deadlock.md` | `★★★☆☆` | [死锁：原理与排查](./45-deadlock.md)
| 13 | `46-race-condition.md` | `★★★☆☆` | [Data Race 与竞争条件：原理、检测与生产实战](./46-race-condition.md)

---
_🔄 最后更新：2026-08-12 03:02_
