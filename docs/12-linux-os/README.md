# 12 Linux Os 模块

> 📌 共 13 个文件（12 个主题 + 索引页），持续更新中 ｜ ✅ 已按面试频率排序（★★★★★ → ★☆☆☆）

---

## 📋 题目索引（点击直接跳转阅读）

> 按面试频率降序排列（★★★★★ → ★★★☆☆）｜ 序号 = 文件名前缀，便于对照文件

| 序号 | 📄 文件名 | 🔥 频率 | 💡 考点 & 跳转 |
|---|---|---|---|
| 09 | `09-09-cpu-cache-mesi-memory-barrier.md` | `★★★★★` | [CPU 缓存一致性（MESI）与内存屏障](./09-09-cpu-cache-mesi-memory-barrier.md) · 关键词：cache line、MESI/MESIF/MOESI、store buffer、invalidate queue、内存屏障、x86 TSO、Go 内存模型 happens-before |
| 11 | `11-11-container-network-and-overlayfs.md` | `★★★★★` | [容器网络底层与 OverlayFS](./11-11-container-network-and-overlayfs.md) · 关键词：netns、veth pair、Linux bridge、iptables NAT、conntrack 表满、VXLAN MTU、OverlayFS、copy-up、whiteout |
| 01 | `01-08-linux-fs-and-interrupts.md` | `★★★★☆` | [Linux 文件系统与中断机制](./01-08-linux-fs-and-interrupts.md) · 关键词：inode、硬链接/软链接、文件锁、VFS、tmpfs、软硬中断、NAPI、eBPF、调度策略 |
| 04 | `04-03-io-model-zero-copy.md` | `★★★★☆` | [I/O 模型与零拷贝](./04-03-io-model-zero-copy.md) · 五种 I/O 模型/Reactor/零拷贝/sendfile/io_uring/SYN Cookie/SO_REUSEPORT/Backlog |
| 05 | `05-04-cgroup-namespace.md` | `★★★★☆` | [cgroup、namespace 与容器资源隔离](./05-04-cgroup-namespace.md) · cgroup v2/OOMKilled/GOMAXPROCS/CPU Throttling |
| 06 | `06-05-linux-troubleshooting.md` | `★★★★☆` | [Linux 线上排障基础 + Core Dump](./06-05-linux-troubleshooting.md) · CPU/内存/I/O 排查、Core Dump 分析、OOM Killer 评分 |
| 07 | `07-06-linux-commands.md` | `★★★★☆` | [Linux 基础高频命令](./07-06-linux-commands.md) · top/ps/ss/lsof/strace/tcpdump/pstack/gstack/ulimit/journalctl /proc/sys/cpuset |
| 10 | `10-10-kernel-allocator-and-synchronization.md` | `★★★★☆` | [内核内存分配（Buddy/Slab）与内核同步（spinlock/mutex/RCU）](./10-10-kernel-allocator-and-synchronization.md) · 关键词：伙伴系统、外部碎片、slub、kmem_cache、/proc/buddyinfo、/proc/slabinfo、RCU 宽限期、per-CPU |
| 12 | `12-12-process-states-and-durability.md` | `★★★★☆` | [进程状态 D/Z 与 fsync 落盘](./12-12-process-states-and-durability.md) · 关键词：D 状态、僵尸/孤儿进程、subreaper、tini、page cache 回写、fsync vs fdatasync、dirty_ratio、journaling、原子替换 |
| 02 | `02-01-process-thread.md` | `★★★☆☆` | [进程、线程、协程与调度](./02-01-process-thread.md) · M/P/G 模型、上下文切换、信号、NUMA、CPU Throttling |
| 03 | `03-02-virtual-memory-page-cache.md` | `★★★☆☆` | [虚拟内存、页缓存与 mmap](./03-02-virtual-memory-page-cache.md) · 地址空间/页表/TLB/缺页中断/THP/Swap/OOM Killer/kswapd |
| 08 | `08-07-ipc.md` | `★★★☆☆` | [进程间通信](./08-07-ipc.md) · 管道/消息队列/共享内存/mmap/socket/eventfd/futex/seccomp/COW |

---
_🔄 最后更新：2026-09-22 03:00_
