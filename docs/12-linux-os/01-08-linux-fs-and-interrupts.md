# Linux 文件系统与中断机制

> 考察频率：★★★★☆  优先级：P0
> 关键词：inode、硬链接/软链接、文件锁、VFS、tmpfs、软硬中断、NAPI、eBPF、调度策略

---

## 1. Inode（索引节点）：文件的"身份证"

### 1.1 什么是 inode

在 Linux 文件系统中，**每个文件都有一个唯一的 inode（index node）**。文件名和文件内容之间通过 inode 间接关联：

```
磁盘布局示意：
┌───────────────┐       ┌──────────────────────────┐
│  目录区        │       │   inode 位图（bitmap）     │
│               │       │                          │
│  dir_name → iNO│────→  │   第 3 号 inode          │
│               │       │   - 文件大小: 4096        │
│  another →  iNO│────→  │   - 权限: rwxr-xr-x      │
│               │       │   - 用户: UID/GID         │
│               │       │   - 时间: atime/mtime/ctime│
│               │       │   - 数据块指针: b[0]~b[11] │
└───────────────┘       │             + 间接指针     │
                        │     -> ib[0] -> 256个指针    │
                        └──────────────────────────┘

读取文件时：先找文件名对应的 inode → 从 inode 获取数据块地址 → 读数据
删除文件时：只删除目录项（文件名→inode的映射），inode 未被引用后才释放空间
```

### 1.2 Inode 包含的关键信息

| 字段 | 含义 | Go 场景影响 |
|------|------|------------|
| `i_mode` | 文件类型+权限 | 判断是否为目录、特殊文件、Socket |
| `i_nlink` | 硬链接数 | =1 时文件可被删除；>1 说明有硬链接 |
| `i_size` | 文件大小 | 日志截断前需要检查 |
| `i_blocks` | 占用磁盘块数 | 实际物理存储，注意 512 字节为单位 |
| `i_uid/i_gid` | 文件所有者 | 容器内常见 UID 映射问题 |
| `i_atime` | 最后访问时间 | 频繁访问会影响性能 |
| `i_mtime` | 最后修改时间 | 文件内容变化时间 |
| `i_ctime` | 最后元数据变更时间 | chmod/chown 会更新此值 |
| 数据块指针 | 指向实际数据的位置 | 直接块、一级间接、二级间接、三级间接 |

### 1.3 硬链接 vs 软链接（符号链接）

```go
// Go 中如何操作链接？
import "os"

// 创建硬链接 —— 指向同一个 inode
err := os.Link("source.txt", "hardlink.txt")
// 结果：两个文件名指向同一个 inode，i_nlink 变成 2
// 删除任何一个，另一个仍然有效
// 限制：不能跨文件系统；root 也不能为目录创建硬链接

// 创建符号链接（软链接）—— 创建新 inode，内容是路径字符串
err := os.Symlink("/path/to/target", "symlink.txt")
// 结果：创建一个全新的 inode，其内容是 "/path/to/target" 这个字符串
// 删除目标文件后，软链接变成悬空（broken link）
// 可以跨文件系统，也可以链接目录
```

**高频对比：**

| 维度 | 硬链接 | 软链接 |
|------|--------|--------|
| 指向 | 同一 inode | 新 inode，存路径字符串 |
| 跨文件系统 | ❌ | ✅ |
| 链接目录 | ❌（除非 root）| ✅ |
| 删除原文件 | ✅ 不影响 | ❌ 悬空 |
| i_nlink 变化 | +1 | 不变化 |
| 统计大小 | 与原文件一致 | 只是路径字符串大小 |
| df / du 视角 | 一样 | du 显示原文件大小 |

**生产陷阱：**
```bash
# 现象：磁盘空间没满但无法创建文件
$ df -i
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
/dev/sda1        10M     10M       0   100% /data

# 原因：大量小文件（如 Redis RDB、Kafka segment、Go 服务临时文件）占满 inode
# 解决：清理小文件、调整挂载参数（mkfs.ext4 -N 更多 inode）、使用 ext4 大文件系统
```

---

## 2. 文件锁：Go 服务的并发安全

### 2.1 Linux 文件锁类型

```
POSIX 文件锁（fcntl）              BSD 文件锁（flock）
┌─────────────────────┐           ┌─────────────────────┐
│ fcntl() 系统调用     │           │ flock() 系统调用    │
│                     │           │                     │
│ F_RDLCK（共享读锁）  │           │ LOCK_SH（共享锁）    │
│ F_WRLCK（排他写锁）  │    vs     │ LOCK_EX（排他锁）    │
│ F_UNLCK（解锁）      │           │ LOCK_UN（解锁）      │
│                     │           │ LOCK_NB（非阻塞）    │
│ 特点：               │           │                     │
│ - 进程退出自动释放   │           │ - 简单易用           │
│ - 可以被子进程继承   │           │ - 进程退出自动释放   │
│ - 支持记录级锁       │           │ - 只支持整文件锁     │
│   （部分字节范围）   │           └─────────────────────┘
└─────────────────────┘
```

### 2.2 Go 中的文件锁实现

```go
// 方式1：flock（最简单，整文件级别）
import "golang.org/x/sys/unix"

func lockFile(path string) error {
    fd, err := os.OpenFile(path, os.O_CREATE|os.O_RDWR, 0644)
    if err != nil {
        return err
    }
    // LOCK_EX | LOCK_NB：排他锁，非阻塞
    err = unix.Flock(int(fd.Fd()), unix.LOCK_EX|unix.LOCK_NB)
    if err == unix.EWOULDBLOCK {
        // 锁已被其他进程持有，立即返回
        return errors.New("file locked by another process")
    }
    return nil
}

// 方式2：fcntl（更精细，可以锁定文件的部分字节）
import "golang.org/x/sys/unix"

func flockWithFcntl(path string) error {
    fd, _ := os.OpenFile(path, os.O_CREATE|os.O_RDWR, 0644)
    flock := unix.Flock_t{
        Type:   int16(unix.F_WRLCK),  // 写锁
        Whence: 0,                     // SEEK_SET
        Start:  0,
        Len:    0,                     // 0 = 整个文件
        Pid:    0,                     // 0 = 当前进程
    }
    err := unix.FcntlFlock(fd.Fd(), unix.F_SETLK, &flock)
    // F_SETLK：非阻塞尝试锁
    // F_SETLKW：阻塞等待锁
    return err
}
```

### 2.3 典型使用场景

```go
// 场景1：单点部署保证（防止多实例重复消费）
type SingleInstance struct {
    lock *os.File
}

func NewSingleInstance(lockPath string) (*SingleInstance, error) {
    f, err := os.OpenFile(lockPath, os.O_CREATE, 0644)
    if err != nil {
        return nil, err
    }
    if err := unix.Flock(int(f.Fd()), unix.LOCK_EX|unix.LOCK_NB); err != nil {
        f.Close()
        return nil, errors.New("another instance is running")
    }
    return &SingleInstance{lock: f}, nil
}

func (s *SingleInstance) Close() {
    unix.Flock(int(s.lock.Fd()), unix.LOCK_UN)
    s.lock.Close()
}

// 场景2：日志追加原子性
// write 本身是原子的（小于 PIPE_BUF 的 write 是原子的，约 4KB）
// 但多个 goroutine 同时写入一个日志文件需要加锁
type SafeLogger struct {
    mu sync.Mutex
    f  *os.File
}

func (l *SafeLogger) Log(msg string) {
    l.mu.Lock()
    defer l.mu.Unlock()
    l.f.WriteString(time.Now().Format(time.RFC3339) + " " + msg + "\n")
    l.f.Sync()  // 确保落盘
}
```

---

## 3. VFS（虚拟文件系统）：Linux 的统一文件抽象

### 3.1 什么是 VFS

VFS 是 Linux 内核提供的**统一文件访问接口层**，让所有文件系统对上层（应用程序）表现为相同 API：

```
应用程序                         VFS 层                  具体文件系统
┌───────────┐                 ┌───────────┐            ┌─────────────┐
│ open("f") │                │ vfs_open()│           │ ext4_open()  │
│ read()    │                │ vfs_read()│           │ xfs_read()   │
│ write()   │                │ vfs_write()│          │ btrfs_write()│
│ stat()    │                │ vfs_stat()│           │ ...          │
│ close()   │                │ vfs_close()│          └─────────────┘
└───────────┘                 └───────────┘
                                    │
                            ┌───────┴───────┐
                            │  dentry缓存   │  ← 目录项缓存（name → inode）
                            │  inode缓存   │  ← 文件元数据缓存
                            │  buffer head │  ← 页缓存管理
                            └───────────────┘
```

### 3.2 VFS 核心数据结构

```c
// Linux 内核中 VFS 的核心结构体：

// super_block：文件系统超级块（每个已挂载文件系统一个）
struct super_block {
    dev_t s_dev;              // 设备号
    unsigned long s_magic;    // 文件系统魔数（EXT4_MAGIC等）
    const struct file_system_type *s_type;  // 文件系统类型
    struct super_operations s_op;             // 超级块操作函数
};

// inode：文件/目录的元数据（全局唯一，不管有多少名字链接到它）
struct inode {
    umode_t i_mode;      // 权限
    uid_t i_uid;         // 所有者
    gid_t i_gid;         // 所属组
    loff_t i_size;       // 文件大小
    struct timespec i_atime/i_mtime/i_ctime;  // 三个时间戳
    u64 i_ino;           // inode 号（文件系统内唯一）
    struct address_space *i_mapping;  // 关联的 page cache
    const struct inode_operations *i_op;  // inode 操作函数表
};

// dentry：目录项（文件名到 inode 的映射）
struct dentry {
    unsigned long d_time;
    struct qstr d_name;           // 文件名
    struct inode *d_inode;        // 指向的 inode（NULL = 不存在）
    struct hlist_node d_hash;     // hash 链表
    struct list_head d_lru;       // LRU 链表（缓存管理）
    struct dentry *d_parent;      // 父目录 dentry
    struct inode *d_sb;           // 所属文件系统
};
```

### 3.3 VFS 的缓存机制：为什么第二次读快

```
第一次读取文件 /app/data.log：
    open → VFS_lookup → 查 dentry 缓存 ❌ → 查硬盘找 dentry
    → 建立 dentry（内存缓存）
    → 找到 inode → 查 inode 缓存 ❌ → 从磁盘加载 inode
    → 建立 inode（内存缓存）
    → 读数据块 → Page Cache miss → 磁盘 IO
    → 数据存入 Page Cache

第二次读取同样的文件：
    open → VFS_lookup → 查 dentry 缓存 ✅ → 命中！
    → 找到 inode → 查 inode 缓存 ✅ → 命中！
    → 读数据块 → Page Cache hit → 直接从内存返回
    → ~μs 级延迟（vs 上次 ~ms 级）
```

**Go 服务的优化启示：**
- 频繁打开的文件，VFS 会自动缓存 dentry 和 inode
- 频繁读取的小文件，Page Cache 能加速 100 倍以上
- 不需要手动做太多优化，Go 标准库 `os.Open()` 已经是高效路径

---

## 4. tmpfs vs ramfs：内存文件系统

### 4.1 区别

```bash
# tmpfs（现代 Linux 推荐）
$ mount -t tmpfs tmpfs /dev/shm  # /dev/shm 就是 tmpfs，默认 50% RAM
$ cat /proc/filesystems | grep tmpfs
tmpfs

# 特性：
# ✅ 基于页面内存，可以用 Swap
# ✅ 有容量限制（mount 时指定 size=XX）
# ✅ 支持标准的文件系统语义（truncate, fallocate 等）
# ❌ 重启后数据丢失

# ramfs（旧版，逐渐被淘汰）
# 特性：
# ✅ 纯内存文件系统
# ❌ 无容量上限 → 可能导致 OOM！
# ❌ 不会 swap-out → 用完即涨
# ❌ 生产环境不建议使用
```

### 4.2 tmpfs 在生产中的应用

```yaml
# K8s 中常见 tmpfs 挂载（Pod Spec）
volumes:
- name: tmp
  emptyDir:
    medium: Memory          # 使用 tmpfs 而不是磁盘
    sizeLimit: 1Gi          # 限制大小，防止 OOM
---
containers:
  volumeMounts:
  - name: tmp
    mountPath: /tmp/app
```

```go
// Go 中利用 tmpfs 做高性能缓存
package main

import (
    "io/ioutil"
    "os"
    "path/filepath"
)

// 场景：Go 服务用文件作为临时数据交换
// 放在 tmpfs 上比磁盘快 10-100 倍（内存 vs 磁盘）
func setupTmpCache(mountPath string) error {
    // 检查是否 mounted tmpfs
    info, err := os.Stat(mountPath)
    if err != nil {
        return err
    }
    // 验证是否为 tmpfs
    fi, err := info.Info()
    if fi.Mode()&os.ModeIrregular != 0 {
        // 可以进一步用 syscall.Statfs 验证 fstype == "tmpfs"
    }
    
    // 在 tmpfs 上做临时文件操作，性能远高于普通磁盘
    tmpFile := filepath.Join(mountPath, "cache.bin")
    data := make([]byte, 1024*1024) // 1MB
    return ioutil.WriteFile(tmpFile, data, 0600)
}
```

**tmpfs 监控：**
```bash
# 查看 tmpfs 使用情况
df -hT | grep tmpfs
# tmpfs           16G  256M   16G   2% /dev/shm

# 查看哪些容器在使用 tmpfs
docker inspect $(docker ps -q) | grep -A5 MountPoints | grep tmpfs
```

---

## 5. 硬中断 vs 软中断 vs 系统调用

### 5.1 三种进入内核的方式

```
┌─────────────────────────────────────────────────────────────────┐
│                    进入内核的三种途径                             │
│                                                                 │
│  ┌──────────────┐                                              │
│  │  硬中断      │   ← 外部硬件触发                              │
│  │  (Hard IRQ)  │   网卡收到包、键盘按键、磁盘写完数据            │
│  │              │                                              │
│  │  流程：                                      │
│  │  1. 外设发信号给 APIC/IOAPIC                       │
│  │  2. CPU 保存现场，跳转到中断向量表                   │
│  │  3. 执行中断处理程序（上半部：快速处理）            │
│  │  4. 标记软中断，恢复现场                           │
│  │  5. 时钟中断或软中断时，执行下半部（延迟处理）    │
│  └──────────────┘                                              │
│                                                                 │
│  ┌──────────────┐                                              │
│  │  软中断      │   ← 内核自己触发                              │
│  │  (Soft IRQ)  │   tasklet、NET_RX_SOFTIRQ、BLOCK_IOP_SOFTIRQ│
│  │              │                                              │
│  │  流程：                                               │
│  │  1. 中断上半部标记待处理                              │
│  │  2. ksoftirqd 内核线程或下一轮时钟中断处理         │
│  │  3. 执行实际的延迟工作（网络包处理、磁盘提交）       │
│  │  特点：可以在多个 CPU 上并行运行                     │
│  │         不能有睡眠（中断上下文）                      │
│  └──────────────┘                                              │
│                                                                 │
│  ┌──────────────┐                                              │
│  │  系统调用     │   ← 用户态主动请求                          │
│  │  (syscall)   │   read()/write()/open()/close()             │
│  │              │                                              │
│  │  流程：                                                   │
│  │  1. 用户态调用 glibc wrapper                            │
│  │  2. 触发 trap 指令（x86: syscall/sysenter）            │
│  │  3. CPU 切换到内核态，跳转到 sys_call_table              │
│  │  4. 内核执行对应函数                                     │
│  │  5. 返回用户态，恢复寄存器                                │
│  │  代价：~100-200 cycles（约几十纳秒）                   │
│  └──────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 关键区别

| 维度 | 硬中断 | 软中断 | 系统调用 |
|------|--------|--------|---------|
| 触发源 | 硬件（网卡/磁盘/定时器）| 内核自身 | 用户态进程 |
| 方向 | 外设→CPU | 内核内部 | 用户→内核 |
| 上下文 | 中断上下文 | 中断上下文 | 进程上下文 |
| 可睡眠 | ❌ | ❌ | ✅ |
| 可抢占 | ❌ | ❌ | ✅ |
| 并行性 | 每个 CPU 独立 | 多 CPU 并行 | 串行（当前线程） |
| Go 相关 | NAPI 处理网络包 | 网络/磁盘软中断 | goroutine 的系统调用 |

### 5.3 Go 工程师要关注的

```go
// ① Go 的网络收发经过硬中断 → NAPI（中断下半部）→ poll（epoll）→ goroutine

// 当网卡收到数据包时：
// 1. 网卡 DMA 把数据写到内存缓冲区
// 2. 网卡发硬中断 → 驱动上半部（ack + 标记软中断）→ 很快返回
// 3. 软中断（NET_RX_SOFTIRQ）在下半部处理：
//    - 解封装协议头
//    - 放入 socket 接收队列
// 4. Go netpoller 发现 FD 可读 → goready → 唤醒 goroutine 读数据

// 如果软中断耗时过长（%si 高）→ 网络处理能力不足 → P99 延迟升高

// ② Go 系统调用过多会导致 %sy 高
// 每调用一次 syscall（read/write/sendto），都要经历用户↔内核切换
// Go 的 netpoller 减少系统调用的原理：批量 epolldetect，一次 epoll_wait 处理多个连接

// ③ 排查方法：
// top 看到 %si（softirq）很高
// → 看是哪个软中断导致
// $ cat /proc/softirqs
#      HI  TIMER NET_TX NET_RX SCHED HRTIMER  TRAFFIC  BLOCK  IRDMA  ISR
# cpu0   0   100     50   5000     0       0        0      0      0    0
#  ↑  NET_RX 高 → 网络包接收处理压力大
```

---

## 6. NAPI：现代 Linux 网络栈的性能革命

### 6.1 传统中断驱动的缺陷

```
传统中断驱动模式：
    收到 1000 个数据包
         │
         ▼
    产生 1000 次中断！（可怕）
         │
         ▼
    每次中断：save state → handler → restore state
    ≈ 1000 × 100μs = 100ms（纯中断开销）
         │
         ▼
    CPU 几乎全花在处理中断上，没时间处理业务逻辑

→ 这就是"中断风暴"，万兆网卡轻松触发
```

### 6.2 NAPI 的工作原理

```
NAPI（New API）模式：
    收到数据包
         │
         ▼
    产生一次中断（仅一次！）
         │
         ▼
    中断上半部：禁用中断，注册 polling
         │
         ▼
    退出中断上下文 → 进入进程上下文
         │
         ▼
    调度 NAPI poll 函数（在 softirq 上下文中执行）
         │
         ▼
    poll 函数循环收取数据包（一次性收最多 weight=64 个）
         │
         ▼
    全部收完 → 重新开启中断
         │
         ▼
    如果还有未处理完的包 → 继续 poll
    否则 → 关闭 poll，等待下次中断

结果：无论来多少包，都只需要 1 次中断 + 1 次 polling
```

### 6.3 Go 中 NAPI 的影响

```bash
# 查看当前 NAPI 配置
cat /sys/class/net/eth0/queues/rx-0/rps_cpus
cat /proc/net/softnet_stat

# 第一个字段的十进制值 = 这段时间处理的 NAPI 报文数
# 第二字段 = RX  dropped（丢包数）
# 第三字段 = frame too short（错帧数）

# 如果第二个数字持续增长：
# → 网络包处理不过来，发生了丢包
# → Go 服务的 recvfrom/sendmsg 可能开始失败
# → EAGAIN/EWOULDBLOCK 增多
```

---

## 7. eBPF：现代 Linux 的可观测性基础设施

### 7.1 eBPF 是什么

eBPF（extended Berkeley Packet Filter）是 Linux 内核的一个**可编程框架**，允许用户在**不修改内核源码、不加载模块**的情况下，在内核空间中运行安全程序：

```
应用代码                          eBPF 虚拟机              内核
┌───────────────┐               ┌───────────┐             ┌───────────┐
│ BCC/bpftrace  │   compile     │ LLVM      │  verify     │  eBPF VM  │
│ Libbpf        │── code gen ──▶ │ compiler  │── check ──▶ │ (sandbox) │
│ Cilium/krping │               │           │             │           │
└───────────────┘               │ maps      │◀─ data ─────│           │
                                │ hooks     │             │ 内核子系统│
                                │ (tracepoint│── exec ──▶ │ (network │
                                 │   kprobe │             │  fs    ) │
                                  │  kretprobe│           └───────────┘
                                   │ uprobe)  │
                                   └───────────┘
```

### 7.2 eBPF 的典型应用场景

```bash
# 场景1：TCP 重传诊断（无需改代码）
$ sudo bpftrace -e 'kprobe:tcp_retransmit_skb { printf("RETRANSMIT pid=%d\n", args->skb->sk->__sk_common.skc_daddr); }'

# 场景2：系统调用耗时追踪
$ sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @start[tid] = nsecs; } tracepoint:syscalls:sys_exit_openat /@start[tid]/ { @latency[pid] = hist(nsecs - @start[tid]); delete(@start[tid]); }'

# 场景3：网络包跟踪
$ sudo bpftrace -e 'kprobe:__inet_listen_skb { printf("LISTEN %pI4:%d\n", arg1, arg2); }'

# 场景4：Go 服务专用：追踪 goroutine 相关的系统调用延迟
# 结合 go tool trace + eBPF，可以完美覆盖用户态 + 内核态
```

### 7.3 eBPF 对 Go 工程师的价值

```
传统排障工具链：
  pprof → 用户态热点（知道哪个函数慢）
  strace → 系统调用列表（知道做了哪些 syscall）
  perf → 内核函数热点（知道内核哪慢）
  → 都是事后分析，覆盖面有限

eBPF 增强：
  bpftrace/bcc → 实时追踪，无侵入
  → 可以看到：TCP 连接状态变化、DNS 解析耗时、文件读写延迟
  → 可以和 pprof 联动：pprof 定位代码热点 → eBPF 看底层 I/O 原因
  → 生产环境零开销长期采集（ebpf programs 编译成 JIT 机器码）
```

---

## 8. Linux 进程调度策略

### 8.1 调度器演进

```
2.6.x ~ 3.x：O(1) 调度器
    - 两个就绪队列（活跃 + 过期）
    - O(1) 复杂度选择下一个进程

3.x ~ 4.x：CFS（Completely Fair Scheduler）
    - CFS 是默认调度器
    - 基于红黑树维护 vruntime（虚拟运行时间）
    - 最小化进程的等待时间

5.15+：EEVDF（Earliest Eligible Virtual Deadline First）
    - 替代 CFS，进一步优化延迟公平性
    - Go 服务受益于更小的调度抖动
```

### 8.2 调度策略（sched_setscheduler）

```c
// Linux 支持的调度策略：

SCHED_OTHER      // 默认：完全公平调度（CFS/EEVDF）
SCHED_FIFO       // 先进先出：实时优先级，一旦运行直到自愿放弃或更高优先级抢占
SCHED_RR         // 轮转法：类似 FIFO，但有时间片轮转
SCHED_BATCH      // 批处理：适合计算密集型，牺牲延迟换取吞吐
SCHED_IDLE       // 空闲：仅在系统空闲时运行
SCHED_DEADLINE   // 截止期优先：指定 deadline/period/runtime（实时的实时）
```

### 8.3 Go 与调度

```go
// Go 调度器绑定到 M（OS 线程），M 使用系统的 SCHED_OTHER
// Go Runtime 有自己的调度逻辑（work-stealing GMP 模型）

// 但如果需要更细粒度的控制：

// 1. LockOSThread() 绑定 goroutine 到特定 OS 线程
// 之后可以用 syscall.SchedSetattr 设置调度属性
import "syscall"

func bindToCPU(cpu int) error {
    var cpuset syscall.CPUSet
    cpuset.Set(cpu)
    return syscall.SchedSetaffinity(syscall.Getpid(), &cpuset)
}

// 2. Go 1.25 GOMAXPROCS 自动感知容器 CPU 配额（见 04-cgroup-namespace.md）

// 3. runtime.GOMAXPROCS 决定了最大并行 goroutine 数
//    一般设置为 CPU 核数即可，不要设太高
runtime.GOMAXPROCS(runtime.NumCPU())
```

---

## 9. 高频追问

### Q1：df 显示磁盘没满，但 mktemp 报"No space left on device"，怎么回事？

A：大概率是 **inode 用完了**。执行 `df -i` 检查 inode 使用率。大量小文件（如 Go 服务的临时文件、Redis dump、Kafka segment）会消耗大量 inode。格式化时 inode 总数固定，ext4 默认每 16KB 分配一个 inode。解决方法：清理小文件或用更大块大小的格式化。

### Q2：hard link 和 symbolic link 各有什么优缺点？生产怎么选？

A：硬链接节省空间且不受路径变动影响，但不能跨分区、不能链接目录。软链接灵活但路径变了会断。生产环境**绝大多数情况用软链接**（版本回滚目录软链是最经典的用法）。只有极特殊的场景才需要硬链接。

### Q3：Go 服务启动慢可能是文件系统的原因吗？

A：可能是。几个关键点：
1. **selinux/AppArmor**：如果开启了 SELinux，每次文件访问都需要检查安全上下文，启动慢 2-5 秒很常见
2. **NFS/远程文件系统**：Go 二进制如果在 NFS 上执行，首次加载非常慢
3. **tmpfs 不够大**：Go 编译产物放 tmpfs 但空间不足导致降级
4. **inode 碎片化**：ext4 在小文件密集环境下 inode 分配变慢

### Q4：%si（软中断）高的时候怎么定位和处理？

A：首先确认是哪个软中断导致的：
```bash
watch -n1 'grep softirq /proc/stat; grep Net_RX /proc/softirqs'
```
如果是 `NET_RX` 高 → 网卡收包处理不过来 → 检查 ethtool irq coalesce 设置、开启 RSS/RPS 分发到多核。如果是 `BLOCK` 高 → 磁盘 I/O 瓶颈。

### Q5：eBPF 和普通 tracing（strace/perf）有什么区别？

A：
- **strace**：追踪单个进程的所有系统调用，开销大（每次 syscall 都打点），只能追正在运行的进程
- **perf**：CPU 采样分析，需要调试符号，关注的是 CPU 热点函数
- **eBPF**：在内核层面可编程，可以追踪任意内核事件（网络/文件系统/调度），开销极低（JIT 编译成机器码），可以长期驻留生产，无需 restart 进程
- **组合拳**：pprof 定位代码 → strace/eBPF 追踪内核行为 → perf 分析内核热点

---

## 延伸阅读

- [Linux Inode 详解](https://www.kawabangga.com/posts/3561)
- [eBPF 入门教程](https://colobu.com/2023/09/11/tracing-a-packet-journey-using-linux-tracepoints-perf-ebpf/)
- [Linux 内核中断处理](https://github.com/0voice/linux_kernel_wiki/blob/main/%E9%9D%A2%E8%AF%95%E9%A2%98/%E9%9D%A2%E8%AF%95%E9%A2%98%E4%B8%80.md)
- [NAPI 工作原理](https://www.kernel.org/doc/html/latest/networking/napi.html)
