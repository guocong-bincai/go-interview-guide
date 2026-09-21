[🏠 首页](../../../README.md) · [📦 Linux / 操作系统基础](../README.md)

---

# 内核内存分配器（Buddy / Slab）与内核同步机制（spinlock / mutex / RCU）

> 考察频率：★★★★☆  优先级：P1
> 关键词：伙伴系统、外部碎片、slab/slub、kmem_cache、/proc/buddyinfo、/proc/slabinfo、spinlock、mutex、RCU、grace period

---

## 面试官考察意图

这道题是**"操作系统底蕴"的试金石**。
候选人如果只会说"内核用 malloc 分配内存"就露怯了。高级工程师要能讲清：**为什么内核需要两套分配器**、**用户态内存分配的完整链路（brk/mmap → 页表 → 伙伴系统 → slab）**、以及 **RCU 为什么能让读路径零开销**（并映射到 Go 的 `sync.Map` / `atomic.Pointer` COW 设计）。

---

## 1. 为什么需要两层分配器

| 层次 | 职责 | 分配单位 | 解决什么 |
|------|------|---------|---------|
| **伙伴系统（Buddy System）** | 管理**物理页** | 页（4KB）的 2^n 次幂块 | 大块连续物理内存、反碎片 |
| **Slab / Slub 分配器** | 管理**小对象** | 字节级（几十 B ~ 几 KB） | 频繁 alloc/free 的性能与内部碎片 |

```
一次 read() 的内核内存旅程：

  用户 read(fd, buf, 64KB)
        │
        ▼
  内核复制数据 → 需要缓冲区
        │
        ├── kmalloc(64KB) ──> Slab 分配器 ──> 自己要不到 ──> 向 Buddy 要页
        │                                                    （order = 4，
        │                                                     2^4 × 4KB = 64KB）
        ▼
  Buddy 从 freelist[4] 摘一个块 → 若没有，向上找 order 5 拆成两个 order 4（一用一挂）
```

**关键结论：Slab 是"批发商"，Buddy 是"厂商"。** 直接每次向 Buddy 要页做小对象分配，开销和碎片都不可接受。

---

## 2. 伙伴系统（Buddy System）

### 2.1 核心思想

内存按 **2^order 页** 分组（order 0 = 1 页 = 4KB，最大通常 order 10 = 4MB）。
分配时若目标 size 无空闲，就**向上找更大的块劈成两半**（两半互为"伙伴"）；释放时若**伙伴也空闲就合并**，向上递归。

```
order 2（16KB）→ 需要 8KB（order 1）

  [ 16KB 空闲块 ]
        │ 劈开
  ┌──────────┬──────────┐
  │ 8KB 使用 │ 8KB 空闲 │   ← 互为伙伴
  └──────────┴──────────┘
        │ 释放使用时
  ┌──────────┬──────────┐
  │ 8KB 空闲 │ 8KB 空闲 │ → 伙伴都在 → 合并回 16KB
  └──────────┴──────────┘
```

**伙伴的判定（经典面试点）**：块地址 XOR (1 << order) 得到伙伴块地址——因为伙伴块由同一个父块劈出，二进制上只有**对应位不同**。

### 2.2 碎片问题

| 碎片类型 | 含义 | 伙伴系统能否解决 |
|---------|------|----------------|
| **内部碎片** | 分配 5KB 实际给了 8KB，浪费 3KB | ❌ 由 Slab 缓解（细粒度 size class） |
| **外部碎片** | 总空闲够，但没有**连续**大块 | ⚠️ 只能靠合并缓解，长期运行必然劣化 |

```bash
# 查看各 order 的空闲块数量（从 order 0 到 max_order）
cat /proc/buddyinfo
# Node 0, zone   Normal   1234  567  89  12   3   1   0   0   0   0   0
#                          ↑order0                 ↑order5

# 判断碎片化程度：order 0 巨大但 order >= 4 全 0 → 严重外部碎片
# 后果：需要大页（THP）/ 大块 DMA 时分配失败，即使 free -h 显示内存充足
```

**真实故障形态**：服务跑了 30 天，`free` 显示还剩 20GB，但网络驱动要 order-3 的块失败，报 `order:3 allocation failure`。解决方案：重启、启用 THP、调 `vm.min_free_kbytes`、减少大块分配。

### 2.3 水位线与回收（与 03 号文件呼应）

```
zone 的三个水位：

  high ──────────────────  kswapd 停止回收
  low  ──────────────────  kswapd 被唤醒（异步回收）
  min  ──────────────────  分配被阻塞，触发 direct reclaim（同步回收，进程自己等）
                                   ↑ 这就是 P99 毛刺的经典来源
```

`vm.min_free_kbytes` 调大 → 更早异步回收，但可用内存变少。**容器内尤其危险：宿主机水位与容器 limit 是两套体系。**

---

## 3. Slab / SLUB 分配器

### 3.1 解决的问题

内核要频繁分配**同类型小对象**（`task_struct`、`dentry`、`inode`、`sk_buff`）。
每次都走伙伴系统 = 每次都要劈/合页 + 清零 + 初始化，开销巨大。

**Slab 思路：预先从 Buddy 拿一批页，切成固定大小对象放进缓存池，分配时直接摘一个，释放时放回池子。**

```
kmem_cache（如 dentry_cache，对象大小 192B）

  ┌─ slab 1 ─┐ ┌─ slab 2 ─┐ ┌─ free slab ─┐
  │ □■□■□■□  │ │ ■■■□□□□  │ │ □□□□□□□    │
  └──────────┘ └──────────┘ └────────────┘
   □ 空闲  ■ 已用
   ↑ 分配 = 摘一个空闲对象；释放 = 标回空闲，不还页给 Buddy
```

### 3.2 Slab 家族演化（2026 面试加分项）

| 实现 | 特点 | 现状 |
|------|------|------|
| **SLAB** | 原始实现，per-CPU 缓存 + 复杂 slab 管理，开销大 | 已基本淘汰 |
| **SLUB** | 简化元数据结构（元数据放 slab 外部），**默认实现** | Linux 2.6.23+ 默认 |
| **SLOB** | 极小内存嵌入式专用 | 嵌入式场景 |

**SLUB 的性能关键：per-CPU freelist（无锁快路径）**

```
分配路径：
  CPU0 要找 dentry 对象
    │
    ├─ 先看本 CPU 的 per-cpu partial slab → 命中则直接摘对象（零锁）
    │
    └─ 未命中 → 从 node 的 partial list 拿一个 slab 挂到 per-cpu（加锁，慢路径）
```

**这跟 Go 的 mcache 几乎一模一样**——见下方对比表。

### 3.3 内存回收与 shrinker

Slab 缓存的内存**不是泄漏就是缓存**，这是排查的难点：

```bash
cat /proc/slabinfo | sort -k3 -n -r | head
# dentry    1234567  1234567   192  ...   ← 第一列名, 第二三列 active/num_objs
# 若 dentry / inode 持续增长且不回落 → 大量小文件被反复访问（或泄漏）

slabtop -s c        # 按内存占用排序，实时观察
```

**shrinker 机制**：内存紧张时，kswapd 会回调各缓存注册的 shrinker，回收空闲 slab 对象（如 dentry 缓存可以丢，重新 scan 目录即可重建）。**dentry 缓存涨到几 GB 通常不是泄漏，是被 shrinker 回收前的正常缓存。**

**Go 服务关联**：容器内存 limit 统计的是 **cgroup 内存总量**，而 Slab 里属于该 cgroup 的部分计入 `memory.current`。大量小文件操作（如 Go 服务遍历日志目录）会导致 dentry/inode slab 上涨，**"我的 Go 程序才用了 200MB，为什么容器 OOM 了"** 的常见原因之一。可查 `memory.stat`：

```bash
cat /sys/fs/cgroup/memory.stat | grep -E "slab|sock|file|anon"
# slab 计入容器内存，但不可 swap，OOM 时优先被杀
```

---

## 4. 内核同步机制：spinlock / mutex / RCU

### 4.1 三者对比（必背）

| 机制 | 等待方式 | 可用上下文 | 典型场景 | 开销 |
|------|---------|-----------|---------|------|
| **spinlock（自旋锁）** | 忙等（PAUSE 指令） | 进程 / **中断上下文** | 临界区极短（< 微秒） | 无竞争时 ~20 cycle |
| **mutex / semaphore** | 睡眠（挂到等待队列） | **仅进程上下文** | 临界区可能阻塞（可能引发 I/O、分配内存） | 无竞争时 CAS 即可 |
| **RCU** | 读不加锁，写延迟释放 | 读侧任意（含中断），写侧进程上下文 | **读多写极少**（路由表、dcache） | 读侧 ≈ 0 |
| **rwlock / seqlock** | 读写分离 | rwlock 可中断上下文 | 读多写少但需阻塞语义 | 中 |
| **per-CPU 变量** | 无同步 | 任意 | 计数器（`percpu_counter`） | 最小 |

### 4.2 spinlock 的两个铁律

```c
// 1) 临界区必须极短 —— 自旋期间 CPU 100% 空转，且关抢占
spin_lock(&lock);
count++;                 // ✅ 几十条指令
spin_unlock(&lock);

// ❌ 绝不能在 spinlock 里做这些：
// kmalloc(GFP_KERNEL)  → 可能睡眠/触发回收 → 内核直接 panic（"scheduling while atomic"）
// copy_from_user()     → 可能缺页 → 睡眠
// msleep() / mutex_lock() → 睡眠

// 2) 中断上下文与进程上下文共享数据时，必须关中断 + 关抢占
unsigned long flags;
spin_lock_irqsave(&lock, flags);   // 保存中断状态并关本地中断
/* ... */
spin_unlock_irqrestore(&lock, flags);
// 原因：若进程持有锁时来了中断，中断处理程序也要同一把锁 → 死锁（中断不会睡，永远等）
```

**Go 的对应物**：Go 运行时的 `runtime.lock`（`sched.lock`）就是 spinlock；Go 用户态没有 spinlock，因为**抢占式调度下自旋会拖垮整个 P**——需要"短临界区"时用 `atomic`，需要"等待"时用 `Mutex`。

### 4.3 RCU：读多写少场景的终极武器（2026 高频）

**核心思想：读者永远不加锁、不写共享数据，写者"复制-修改-原子发布"，旧版本等所有读者离开后再释放。**

```
RCU 三个步骤：

① 读者（零开销）
   rcu_read_lock();       // 只是 preempt_disable()，没有原子指令！
   p = rcu_dereference(gp);  // 拿指针
   use(p);                // 用对象
   rcu_read_unlock();     // preempt_enable()

② 写者（copy → update → publish）
   new = kmalloc(...);          // 复制一份
   *new = *old;  new->field = x; // 修改副本
   rcu_assign_pointer(gp, new);  // 原子发布（带 release 屏障）
   synchronize_rcu();            // 等待所有已存在的读者读完 → 宽限期（grace period）
   kfree(old);                   // 现在才能释放

③ 宽限期判定：所有 CPU 都经历过一次"静止态"（quiescent state）
   —— 上下文切换、用户态执行、idle 都算
```

| 维度 | RCU | 读写锁 rwlock |
|------|-----|--------------|
| 读开销 | **几乎为零**（无原子指令、无 cache line 争用） | 每次都要原子操作改计数器 |
| 读扩展性 | **线性扩展**到几百核 | 读侧也有 cache line 争用 |
| 内存开销 | 读写并发期间存在两个版本 | 单版本 |
| 写开销 | 高（等待宽限期，可能毫秒级） | 低 |
| 适用 | **读:写 > 100:1** | 读多写少但仍需阻塞语义 |

**RCU 广为人知的应用**：内核路由表、`dentry` 哈希表、`epoll` 的 wait queue（部分）、eBPF map。
**用户态 RCU 库**：`liburcu`。**Go 里没有正式 RCU**，因为 GC 天然提供了"延迟回收"能力。

### 4.4 Go 中的等价物：为什么 Go 不需要 RCU

```go
// Go 版"RCU-lite"：atomic.Pointer + 不可变对象整体替换
type Table struct {
    m map[string]string   // 只读，发布后永不修改
}

var table atomic.Pointer[Table]

// 读者：全无锁，等价 rcu_read_lock + rcu_dereference
func Lookup(k string) (string, bool) {
    t := table.Load()      // 一次原子读
    v, ok := t.m[k]
    return v, ok
}

// 写者：复制-修改-发布，等价 RCU 写路径
func Update(k, v string) {
    old := table.Load()
    next := make(map[string]string, len(old.m)+1)
    for kk, vv := range old.m {
        next[kk] = vv
    }
    next[k] = v
    table.Store(&Table{m: next})   // 原子发布；旧对象由 GC 回收
}
```

**对比要点（面试高价值回答）：**
- RCU 的 `synchronize_rcu` + `kfree` ↔ Go 的 GC（延迟、并发、自动判定"没有引用"）
- **但 GC 不保证"读者不再持有指针时立即释放"**，所以 Go 里做长生命周期的内存敏感服务时，COW 全量复制 map 的开销是真实成本（大表要分片）
- `sync.Map` 是另一条路：`read` 字段（`atomic.Pointer[readOnly]`）就是 RCU 式快照，**写频繁或元素多时会退化成互斥锁**

```go
// sync.Map 内部结构（简化）
type Map struct {
    mu     Mutex
    read   atomic.Pointer[readOnly]  // 无锁读快照（≈ RCU 的读副本）
    dirty  map[any]*entry            // 写入的新数据
    misses int
}
// 读：先查 read（无锁）；miss 次数超过 dirty 长度 → 加锁把 dirty 提升为 read
// 结论：读多写少（读:写 > 10:1）才划算，否则用 Mutex + map
```

---

## 5. 完整链路：一次 malloc 到物理内存

```
Go:   make([]byte, 1024)
        │
        ▼
   Go 分配器（mcache → mcentral → mheap）
        │  需要新页时
        ▼
   syscall: mmap(MAP_ANONYMOUS) / brk
        │
        ▼  ──────────── 用户态/内核态分界 ────────────
        ▼
   内核：建立 VMA（vm_area_struct）
        │
        ▼
   首次访问该虚拟地址 → 缺页异常（Page Fault）
        │
        ▼
   内核：分配物理页 = Buddy System（order 0，单页）
        │  （若该页用于内核小对象，则来自 Slab 缓存）
        ▼
   建立页表映射（PTE）→ 更新 TLB → 返回用户态重试指令
        │
        ▼
   进程拿到可用的物理内存
```

**对比表（面试可直接说）**

| 维度 | Linux 内核 | Go 运行时 |
|------|-----------|-----------|
| 页大小 | 4KB（+ Huge Page） | 8KB span 粒度 |
| 小对象分配 | Slab/SLUB（per-CPU freelist） | mcache（per-P 无锁） |
| 中央缓存 | node partial list / kmem_cache | mcentral（按 size class 分锁） |
| 大块来源 | Buddy System | mheap（arena + page allocator） |
| 归还策略 | shrinker 按需回收 | GC 后归还 OS（`scavenger`，`MADV_FREE`/`MADV_DONTNEED`） |
| 碎片治理 | 合并伙伴块 | size class + span 复用 |
| 首次触达 | 缺页异常按需分配 | 同样走缺页 |

**"Go 程序 RSS 不降" 的答案就在最后一行**：Go 用 `MADV_FREE` 归还内存，RSS 下降依赖 OS 压力；`GODEBUG=madvdontneed=1` 可改成立即归还。

---

## 6. 高频追问

**Q1：为什么内核不用 malloc/free？**
内核没有 libc，也不能假设有堆；内核栈只有 8~16KB，无法承受递归分配；且内核需要在**中断上下文**分配内存（不能睡眠），需要 `GFP_ATOMIC` 这类专用路径。`kmalloc` 是 Slab 之上的封装，`vmalloc` 则用于分配**虚拟连续但物理可不连续**的大块。

**Q2：`kmalloc` 和 `vmalloc` 的区别？**
`kmalloc` 物理连续，适合 DMA，但大块易失败（碎片）；`vmalloc` 虚拟连续 + 物理离散，几乎不会因碎片失败，但**每次访问都可能跨页触发 TLB miss**，性能差。故 `vmalloc` 只用于大块、非性能敏感场景（模块加载）。

**Q3：Slab 会不会内存泄漏？**
会。最常见是 `sk_buff`（网络）、`dentry` / `inode` 泄漏。判定方法：`slabtop` 观察某缓存对象的 `num_objs` 持续上涨且 shrinker 无法回收。**注意区分"缓存"与"泄漏"**：dentry 可以被回收，所以涨到几 GB 也未必是泄漏；`task_struct` 只增不减就是真泄漏。

**Q4：RCU 的宽限期为什么不能让写者立即释放旧对象？**
因为可能还有读者正持有旧指针在遍历（它们没有加锁，内核无从得知）。必须等到所有 CPU 都经历静止态（上下文切换/idle/用户态），才能确认"没有读者持有旧指针"。

**Q5：`sync.Map` 比 `Mutex + map` 快在哪？为什么官方说"大多数场景应该用 Mutex"？**
快路径是 `atomic.Pointer` 读只读快照，读侧无锁无 cache line 争用。但**写入必须复制 entry 并最终加锁**，且每次 miss 都要遍历 dirty；元素 > 几千或写占比高时，性能明显低于 `RWMutex + map`。官方结论：只有当 key 稳定、读远多于写、且各 goroutine 操作不相交集合时才划算。

**Q6：per-CPU 计数器为什么能避免伪共享？**
数据物理上分散在每核私有内存，各核只改自己的副本，无 cache line 争用；读总数时遍历所有副本求和（`percpu_counter_sum`）。代价是**读慢、值可能短暂不准**——这正好是内核 `percpu_counter` 的取舍（如 `vm_stat`）。

---

## 延伸阅读

- [Linux Kernel 文档：Memory Management](https://docs.kernel.org/mm/index.html)
- [What is RCU, Fundamentally? (LWN)](https://lwn.net/Articles/262464/)
- [SLUB 分配器设计文档](https://docs.kernel.org/mm/slub.html)
- [Go runtime 内存分配器设计](https://go.dev/doc/go1.22)
