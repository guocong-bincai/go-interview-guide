# 12 · Linux / 操作系统

> 考察频率：★★★★★  优先级：P0
> 高级 Go 工程师必须补齐的底层基础：调度、内存、I/O、容器资源与线上排障

## 文章清单

### 基础与排障
- [✅] `01-process-thread.md` — 进程、线程、协程与调度：上下文切换、goroutine 与线程映射 + M/P/G 生命周期/CPU Throttling
- [⏳] `02-virtual-memory-page-cache.md` — 虚拟内存、页缓存与 mmap：页表、缺页中断、Page Cache
- [⏳] `03-io-model-zero-copy.md` — I/O 模型与零拷贝：epoll、sendfile、Go netpoller
- [✅] `04-cgroup-namespace.md` — cgroup、namespace 与容器资源隔离：OOMKilled、资源限制 + CPU Throttling/GODEBUG/
- [✅] `05-linux-troubleshooting.md` — Linux 线上排障基础：CPU、内存、fd、IO、TIME_WAIT + CPU Throttling 排障/镜像优化
- [✅] `06-linux-commands.md` — Linux 基础高频命令：top / ps / ss / lsof / strace / tcpdump / perf
- [✅] `07-ipc.md` — 进程间通信：管道、消息队列、共享内存、信号量、Socket + eventfd/seccomp/COW/futex（2026 大更新）
- [✅] `08-linux-fs-and-interrupts.md` — Linux 文件系统与中断机制：inode/链接/文件锁/VFS/tmpfs/软硬中断/NAPI/eBPF/调度策略
