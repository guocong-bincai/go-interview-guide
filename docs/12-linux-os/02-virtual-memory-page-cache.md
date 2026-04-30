[🏠 首页](../../../README.md) · [📦 Linux / 操作系统基础](../README.md)

---

# 虚拟内存、页缓存与 mmap

> 考察频率：★★★★☆  优先级：P0
> 关键词：虚拟地址空间、页表、缺页中断、Page Cache、mmap、Swap

---

## 1. 虚拟内存：进程眼中的"私有地址空间"

### 1.1 虚拟地址空间布局（64-bit Linux）

```
高地址
┌─────────────────────┐ 0xFFFFFFFFFFFFFFFF
│     内核空间        │ （只有内核能访问，进程不可见）
├─────────────────────┤ 0xFFFF800000000000 (VA_BITS = 48)
│        ↑           │
│   未使用（guard）   │
├─────────────────────┤
│   栈（向下增长）    │ ← SP 寄存器，8MB 默认大小
│        ↓           │
├─────────────────────┤
│   内存映射段 mmap   │ ← mmap、动态库、共享内存
├─────────────────────┤
│   堆（向上增长）    │ ← brk() 扩展，malloc 底层
├─────────────────────┤
│   BSS（未初始化）   │ ← global 未赋值变量，0 初始化
├─────────────────────┤
│   Data（已初始化）  │ ← global 变量，有初值
├─────────────────────┤
│   Text（代码段）    │ ← 只读，指令
├─────────────────────┤
│   保留区            │ ← 防止空指针解引用
└─────────────────────┘ 0x0000000000400000
低地址
```

**32 位 vs 64 位：**
- 32 位：4GB 虚拟地址（用户态 3GB，内核 1GB）
- 64 位：128TB 用户空间（实际使用 VA_BITS=48，仅 256TB）

### 1.2 虚拟内存的核心作用

**① 进程隔离**：每个进程有独立虚拟地址空间，A 进程 0x00400000 是代码段，B 进程 0x00400000 也是代码段，互不干扰

**② 解决碎片化**：物理内存可以分散，虚拟地址连续，应用不关心物理布局

**③ 内存保护**：页表项（PTE）记录 R/W/X 权限，越权访问触发段错误

**④ 按需分配**：虚拟页可以不映射物理页（缺页时才分配）

---

## 2. 页表与 TLB

### 2.1 多级页表（x86-64）

Linux 使用四级页表：
```
虚拟地址（VA）结构：
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ PML4     │ PDPT     │ PD       │ PT       │ offset   │
│ 9 bits   │ 9 bits   │ 9 bits   │ 9 bits   │ 12 bits  │
└──────────┴──────────┴──────────┴──────────┴──────────┘
   │          │          │          │          │
   ▼          ▼          ▼          ▼          ▼
 PML4E      DPDTE       PDE        PTE      物理页帧
（页全局目录的索引）                              （4KB 对齐）
```

**为什么需要多级页表？**
- 64 位地址空间 256TB，如果是一级页表，每个进程需要 256TB/4KB = 640亿 个页表项（不可能）
- 多级页表：只分配实际使用的页表（稀疏地址空间）
- 4 级：4KB × 512 = 2MB（最坏情况），实际使用更少

### 2.2 TLB（Translation Lookaside Buffer）

TLB 是 MMU 内部的**最近访问页表项缓存**：

```
CPU 请求虚拟地址 VA
       │
       ▼
  TLB 查找（~1 cycle）
       │
   命中 → 直接得到物理地址 PA → 访问物理内存
       │
   未命中 → MMU walk 多级页表（~100 cycles）
            → 填充 TLB → 再次查找
```

**TLB 的问题与 Go 的关系：**
- TLB 是 CPU 绑定的（每颗 CPU 一个）
- 进程切换时，TLB 必须 flush（旧实现）或 ASID-tag（新实现）
- **goroutine 切换不 flush TLB**（同一进程的虚拟地址空间不变）
- Go 1.20 引入了 **PCID + TLB shootdown 优化**，减少不必要刷新

### 2.3 缺页中断（Page Fault）

缺页中断类型：

| 类型 | 含义 | 处理方式 |
|------|------|---------|
| Soft Fault | 页在物理内存，但页表未建立映射 | 内核建立映射，用户进程继续运行 |
| Hard Fault | 页不在内存，在磁盘 Swap | 从 Swap 读入（~ms 级延迟），再映射 |
| Segfault | 访问了未映射且非 Swap 的地址 | 发送 SIGSEGV，进程崩溃 |

```bash
# 查看缺页中断次数
cat /proc/vmstat | grep pg

# 查看进程的缺页情况
cat /proc/<pid>/status | grep -i pagemap
```

---

## 3. Page Cache（页缓存）：磁盘 I/O 的加速器

### 3.1 什么是 Page Cache

Linux 的 Page Cache 是**内存中缓存磁盘数据的区域**：

```
应用程序 read(fd, buf)
       │
       ▼
  内核查找 Page Cache
       │
   命中（Cache Hit）→ 直接从内存返回（~μs 级）
       │
   未命中（Cache Miss）
       │
       ▼
  发起磁盘 I/O → 读取到 Page Cache → 返回给应用（~ms 级）
       │
       ▼
  应用读取后，Page Cache 保留（可能被后续 read 复用）
```

**Page Cache 的回写策略：**
```bash
# 默认：pdflush 延迟回写（dirty_ratio 触发）
# /proc/sys/vm/dirty_ratio = 20%  # 内存 20% 为脏页时触发回写
# /proc/sys/vm/dirty_expire_centisecs = 3000  # 30秒后变为可回写
# /proc/sys/vm/dirty_writeback_centisecs = 500  # 5秒检查一次
```

### 3.2 mmap：将文件映射到内存

```go
// Go 中使用 mmap 做高性能文件读取
package main

import (
    "fmt"
    "os"
    "syscall"
)

func main() {
    // 打开文件
    fd, err := syscall.Open("bigfile.dat", os.O_RDONLY, 0)
    if err != nil {
        panic(err)
    }
    defer syscall.Close(fd)

    // 获取文件大小
    fi, err := os.Stat("bigfile.dat")
    size := fi.Size()

    // mmap 映射到内存
    data, err := syscall.Mmap(fd, 0, int(size), syscall.PROT_READ, syscall.MAP_PRIVATE)
    if err != nil {
        panic(err)
    }
    defer syscall.Munmap(data)

    // 直接像访问数组一样读取文件
    fmt.Println(string(data[:100]))
}
```

**mmap vs read/write 对比：**

| 维度 | read/write | mmap |
|------|-----------|------|
| 数据拷贝次数 | 2次（磁盘→Page Cache→用户）| 1次（磁盘→Page Cache）直接映射 |
| 内存占用 | 用户缓冲区 + Page Cache | 只用 Page Cache |
| 延迟 | 固定 2次拷贝 | 按需 page fault，首次访问有 ms 级延迟 |
| 适合场景 | 小文件、随机访问 | 大文件顺序读、共享内存、动态库加载 |

**mmap 的生产陷阱：**
```go
// ⚠️ mmap 文件增长时的坑
// 如果 mmap 一个不断增长的文件（append only log）：
data, _ := syscall.Mmap(fd, 0, currentSize, PROT_READ, MAP_PRIVATE)
// 当文件增长时，必须重新 mmap，否则读不到新数据
// 生产中 append 型日志不要用 mmap，改用 seek + read

// ⚠️ MAP_PRIVATE vs MAP_SHARED
// MAP_PRIVATE：写时复制，不影响原文件（适合读）
// MAP_SHARED：写操作直接落盘（适合共享内存）
```

### 3.3 Go 的内存分配与 Page Cache

Go 的内存分配器基于 tcmalloc，分层结构：

```
mcache（每个 P 一个）  ← 无锁，极快
     │
     ▼
mcentral（每个 sizeclass 一个）← 加锁
     │
     ▼
mheap（全局）           ← 加锁，最慢
     │
     ▼
     OS（mmap匿名映射） ← 每次 64KB 对齐
```

**Go 的 Page Cache 优化：**
- mspan（Go 的内存页）：2~32 个连续操作系统页（4KB~）
- mcache 持有 67 种 sizeclass 的 mspan，无锁分配
- 分配路径：P 的 mcache → mcentral → mheap → OS（按需 64KB 增长）

---

## 4. 高频追问

### Q1：Go 服务内存占用高（RSS 很大）是什么原因？

**原因分析：**
1. **Go 堆内存**：正常，Go 运行时会持有未归还 OS 的内存（`memory released to OS` 是延迟的）
2. **mmap 区域**：mmap 映射的文件、动态库
3. **Page Cache**：Go 服务读取的磁盘文件被 OS 缓存
4. **goroutine 栈**：百万 goroutine × 2KB = 2GB（可控）

**验证方法：**
```bash
# 查看详细内存分布
cat /proc/$(pidof your-app)/smaps_rollup | head -30

# 对比 RSS 和 Heap
go runtime: GOGC=100 时，堆 live 100MB → RSS 可能 300MB
# 差值 = goroutine栈 + mmap + Page Cache（正常）
```

### Q2：内存分配不释放（memory leak）vs OS 内存回收

Go 的 `runtime.FreeOSMemory` 是延迟的，不会立即归还内存给 OS。
使用 `debug.FreeOSMemory()` 可以主动归还：
```go
import "runtime/debug"
// 定期调用（生产中一般不必要）
debug.FreeOSMemory()
```

### Q3：Swap 对 Go 服务的影响

**Swap 是性能杀手**：Go 的 GC 会在很短时间内扫描整个堆，如果堆被 Swap 到磁盘，GC 可能导致秒级停顿。

**生产建议：**
```bash
# 关闭 Swap（生产服务器通常这么做）
swapoff -a

# 或者设置 swappiness 很低
sysctl vm.swappiness=1  # 内存不足时才用 Swap
```

---

## 延伸阅读

- [Linux 内存管理文档](https://www.kernel.org/doc/html/latest/vm/)
- [Go 内存分配器 tcmalloc 原理](https://github.com/golang/go/blob/master/src/runtime/mheap.go)
- [mmap 性能分析](https://www.flamingspork.com/blog/2022/01/04/mmap-vs-read/)
