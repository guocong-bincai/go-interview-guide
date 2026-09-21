[🏠 首页](../../../README.md) · [📦 Linux / 操作系统基础](../README.md)

---

# 进程状态（D/Z）、僵尸进程回收与 fsync 落盘 / 文件系统崩溃一致性

> 考察频率：★★★★☆  优先级：P1
> 关键词：D 状态、uninterruptible sleep、僵尸进程、孤儿进程、subreaper、tini、page cache 回写、fsync vs fdatasync、dirty_ratio、journaling、delayed allocation、原子替换

---

## 面试官考察意图

这一组是**"线上事故复盘"类问题的最佳素材**：
- 为什么 `kill -9` 杀不掉进程，CPU 还不忙？（D 状态）
- 为什么容器里堆了上千个僵尸进程？（Go 的 `exec.Cmd` 没 `Wait`）
- 为什么写了数据还是丢了？（`write` 成功 ≠ 落盘）
- 为什么每天整点 P99 突然飙高？（脏页阈值触发的同步回写）
- 为什么机器断电后文件变成 0 字节？（缺 `fsync` 目录项）

答得好，面试官会认为你**真的抗过线上**。

---

## 1. 进程状态全景

```bash
ps -eo pid,ppid,stat,wchan:20,comm | head
```

| 状态 | ps 显示 | 含义 | 可被 kill 唤醒？ |
|------|--------|------|----------------|
| **R** | `R` | Running / Runnable（在运行队列中） | — |
| **S** | `S` | Interruptible Sleep（可中断睡眠，等事件） | ✅ 信号能唤醒 |
| **D** | `D` | **Uninterruptible Sleep（不可中断睡眠，通常等 I/O）** | ❌ **`kill -9` 无效** |
| **T** | `T` | Stopped（`SIGSTOP` / 调试暂停） | 需 `SIGCONT` |
| **Z** | `Z` | Zombie（已退出，等待父进程 `wait`） | ❌ **杀不掉** |
| **X** | 罕见 | Dead（几乎不可见） | — |

**ps STAT 附加标志：**

| 标志 | 含义 |
|------|------|
| `<` | 高优先级 |
| `N` | 低优先级（nice） |
| `s` | 会话首进程（session leader） |
| `l` | 多线程 |
| `+` | 前台进程组 |
| `Ss` / `Ssl` / `Z` | 常见组合：`Ss`=可中断睡眠+会话首进程 |

---

## 2. D 状态：最容易被误判的"卡死"

### 2.1 什么会导致 D 状态

进程发起**无法被信号打断**的内核等待：

| 场景 | 说明 |
|------|------|
| 块设备 I/O | 磁盘故障、坏道重试、RAID 降级重建 |
| **NFS / CIFS / 网络块设备挂载点** | 服务端不响应，客户端无限重试（经典事故） |
| `sync()` 等待脏页回写 | 全局同步调用会被拖住 |
| **内存直接回收（direct reclaim）** | 内存紧张时分配内存的进程自己去做回收，等 I/O |
| 等待内核锁 | 并发极高的锁争用 |
| 冻结（freezer）| cgroup freezer 挂起容器时 |

### 2.2 为什么 `kill -9` 杀不掉

```
信号处理只发生在进程返回用户态的那一刻（signal delivery 在内核态 → 用户态边界）

  D 状态进程：一直停在内核代码里等 I/O
      │
      ▼
  内核态代码路径不检查 pending signal（除非该等待被设计为可中断）
      │
      ▼
  SIGKILL 只能标记 pending，进程永远没机会回到用户态处理它
      │
      ▼
  必须等 I/O 完成 / 硬件恢复，或重启机器
```

**面试必答的关键判断：D 状态进程长时间存在 = 一定是 I/O 或内核资源问题，不要浪费时间去 kill。**

### 2.3 排查 SOP

```bash
# ① 找出所有 D 状态进程及其等待点
ps -eo pid,ppid,stat,wchan:30,comm | awk '$3 ~ /D/'

# wchan 直接告诉你卡在哪：rpc_wait_bit_killable（NFS）、
# balance_dirty_pages（脏页限流）、__alloc_pages（内存回收）

# ② 看进程内核栈（需要 root / CAP_SYS_ADMIN）
cat /proc/<pid>/stack
# [<0>] rpc_wait_bit_killable+0x24/0x60 [nfs]
# [<0>] nfs_wait_on_request+...

# ③ 全局看：谁在等 I/O
iostat -x 1 5          # 看 %util、await、r_await（设备是否饱和）
cat /proc/<pid>/status | grep -E "State|VmRSS"

# ④ 内存回收导致的 D：看回收统计
cat /proc/vmstat | grep -E "pgscan_direct|allocstall"
# allocstall 持续增长 → 频繁 direct reclaim → Go 服务表现为分配内存卡顿

# ⑤ 网络文件系统（最常见的元凶）
mount | grep -E "nfs|cifs"     # 生产环境挂载点务必加 soft,timeo,retrans
```

### 2.4 预防设计（Go 服务）

- 挂载点不要用**硬挂载**（默认 `hard`），改 `soft,timeo=30,retrans=2,intr`——否则 NFS 挂了整个服务 D 住
- 容器里避免挂载外部网络存储作为**数据目录**（至少要有本地 fallback）
- 内存 limit 留足余量，避免 direct reclaim；Go 侧配合 `GOMEMLIMIT`（见 [虚拟内存与页缓存](./03-02-virtual-memory-page-cache.md)）
- **健康检查要能识别 D 状态**：进程还活着但完全不响应 → 探针超时 → 触发重启（注意别让重启风暴掩盖真因）

---

## 3. 僵尸进程（Z）与孤儿进程

### 3.1 三态转换

```
父进程 fork() ──> 子进程 exit()
                      │
                      │ 内核释放：内存、fd、大部分内核结构
                      │ 保留：task_struct 中的退出码 + 进程号（PID）
                      ▼
                 状态 = Z（僵尸）
                      │
     父进程 wait()/waitpid() 收尸
                      │
                      ▼
                 task_struct 释放，PID 回收

⚠️ 若父进程不 wait，僵尸永久占用 PID 表项：
   /proc/sys/kernel/pid_max（默认 32768 或 4194304）
   → 僵尸堆积到 pid_max → 无法再创建任何进程（fork: Cannot allocate memory）
```

**孤儿进程**：父进程**先**退出 → 子进程被 `init`(PID 1) 收养 → `init` 会自动 wait，**不会变僵尸**。
（Linux 3.4+ 可用 `PR_SET_CHILD_SUBREAPER` 指定"次级收养者"，systemd/容器运行时常用。）

### 3.2 Go 里的僵尸进程（超高频面试点）

```go
// ❌ 错误写法一：启动了不 Wait 也不 Release
func badStart() {
    cmd := exec.Command("sh", "-c", "sleep 3600")
    cmd.Start()          // 只 Start 不 Wait
    // 函数返回后：cmd 被 GC，但子进程尚未退出
    // → 子进程退出后必然变僵尸（Go 运行时不会替你 wait）
}

// ❌ 错误写法二：以为 Release 能回收
func badRelease() {
    cmd := exec.Command("sleep", "1")
    cmd.Start()
    cmd.Process.Release()  // ⚠️ Release 只是"放弃管理"，不是 wait！
    // 文档明确：Release 后进程资源不会被回收，僵尸依旧
}

// ✅ 正确写法：Wait 是唯一收尸方式
func good() error {
    cmd := exec.CommandContext(ctx, "sh", "-c", "sleep 1")
    cmd.Start()
    return cmd.Wait()      // 收尸 + 释放 fd + 拿到退出码
}

// ✅ 后台长任务：起一个 goroutine 专门收尸
func startBackground(ctx context.Context) {
    cmd := exec.CommandContext(ctx, "worker")
    if err := cmd.Start(); err != nil {
        log.Fatal(err)
    }
    go func() {
        if err := cmd.Wait(); err != nil {   // 必须有！
            log.Printf("worker exited: %v", err)
        }
    }()
}
```

**验证方式（面试可直接说）：**

```go
// 单测里断言没有僵尸：跑完后检查子进程状态
cmd := exec.Command("true")
_ = cmd.Run()   // Run = Start + Wait，不会留僵尸
```

```bash
# 线上检测
ps -eo stat,ppid,pid,comm | awk '$1 ~ /Z/'
ps -eo stat,ppid,pid,comm | awk '$1 ~ /Z/ {print $2}' | sort | uniq -c | sort -rn
# 输出：父进程 PID + 僵尸数量 → 定位是"哪个进程没有 wait"
```

### 3.3 容器中的特殊问题

```
容器里 Go 程序作为 PID 1：

  PID 1 在内核中语义特殊：
  （1）未注册 handler 的信号，其默认动作被忽略（SIGTERM 不会杀进程！）
  （2）PID 1 必须负责 reap 所有"被收养的孤儿"子进程

  Go 程序默认不调用 wait 处理孤儿 → 容器里僵尸堆积

✅ 解决方案：
  A. 用 tini：docker run --init  /  K8s 里加 initContainer + shareProcessNamespace
  B. Go 自己处理：signal.Notify + syscall.Wait4(-1, ...) 循环回收
  C. 不在 PID 1 位置跑：entrypoint 包一层 shell（不推荐，shell 也会忽略 SIGTERM）
```

```go
// 方案 B：Go 自己当"init"，回收所有孤儿进程（适用于必须做 PID 1 的场景）
func reapZombies() {
    for {
        var ws syscall.WaitStatus
        _, err := syscall.Wait4(-1, &ws, syscall.WNOHANG, nil)
        if err == nil || err == syscall.ECHILD {
            time.Sleep(100 * time.Millisecond)
            continue
        }
        time.Sleep(10 * time.Millisecond)
    }
}
```

```bash
# 容器内验证
docker exec <c> ps -eo stat,pid,comm | grep Z
kubectl exec <pod> -- ps -eo stat,pid,ppid,comm
```

---

## 4. 僵尸进程 / D 状态 高频追问

**Q1：僵尸进程占内存吗？**
只占 **PID 表项 + 一个很小的 `task_struct`**（约 1~2KB，不可换出），几乎不占内存。真正的危害是**耗尽 PID 号**（`pid_max`）以及暴露父进程逻辑 bug。**别把"僵尸导致内存高"当成结论。**

**Q2：能 `kill -9` 干掉僵尸吗？**
不能。僵尸已经死了，没有可执行的代码。只能 `kill` 它的**父进程**（父进程死后僵尸被 init 收养并回收），或者让父进程正确 `wait`。

**Q3：为什么容器里的 PID 1 会忽略 SIGTERM？**
内核为 PID 1 特殊处理：只对**显式注册了 handler** 的信号执行默认动作。所以 Go 程序若不 `signal.Notify(SIGTERM)`，收到 `SIGTERM` 会被忽略 → K8s 只能等 `terminationGracePeriodSeconds` 后 `SIGKILL`（表现为"优雅退出不生效/每次都要等 30 秒"）。

**Q4：`WaitGroup` 和"回收子进程"是一回事吗？**
不是。`WaitGroup` 是 goroutine 同步；子进程收尸必须靠 `syscall.wait4`/`os/exec.Cmd.Wait`。**goroutine 结束不代表子进程被回收**——这正是 `Cmd.Start()` 后不 `Wait()` 的坑。

**Q5：D 状态进程会推高 load average 吗？**
**会。** Linux 的 load average 统计 **R + D** 两种状态（这与 Unix 传统定义不同），所以大量 I/O 等待会让 load 飙高而 CPU 很闲。这就是"load 高但 CPU 不高"的经典解释。

---

## 5. 写入落盘：write / fsync / fdatasync

### 5.1 写路径全景（P99 毛刺的源头）

```
应用 write(fd, buf, n)
    │
    ▼ ① 只写进 Page Cache（内存），立即返回成功！
   Page Cache（脏页）
    │
    ▼ ② 后台回写（异步）
    │     flusher 线程按 vm.dirty_writeback_centisecs 周期扫描
    │     或脏页比例超过 vm.dirty_background_ratio 时启动
    ▼
   磁盘设备队列（可能带自身 write cache）
    │
    ▼ ③ 设备真正落盘（需 FUA/flush 才保证持久）
   持久介质
```

**核心结论：`write()` 返回成功只代表"数据进了内存"，机器断电就丢。**

```bash
# 关键参数
sysctl vm.dirty_ratio                 # 默认 20：脏页占比超此值 → 应用 write 被同步阻塞（P99 毛刺！）
sysctl vm.dirty_background_ratio      # 默认 10：后台开始异步回写
sysctl vm.dirty_expire_centisecs      # 默认 3000（30s）：脏页最长存活时间
sysctl vm.dirty_writeback_centisecs   # 默认 500（5s）：回写线程唤醒周期
cat /proc/meminfo | grep -i dirty
# Dirty:           123456 kB     ← 当前脏页
# Writeback:            0 kB     ← 正在回写
```

**为什么"整点 P99 飙高"？** 日志/报表类服务每秒写几千条，脏页慢慢累积到 `dirty_ratio`，触发的瞬间所有写线程被同步回写阻塞 → 几百 ms 到数秒的毛刺，且**周期性出现**。

**调优方向：**

```bash
# ① 用绝对值而非比例（大内存机器上 20% 可能是几十 GB 脏页！）
sysctl -w vm.dirty_background_bytes=268435456   # 256MB
sysctl -w vm.dirty_bytes=1073741824             # 1GB
# 注意：设置 *_bytes 后，*_ratio 自动归零，二者互斥

# ② 让 Go 侧主动、均匀地落盘，而不是等内核憋大招
```

### 5.2 fsync vs fdatasync

| 调用 | 同步内容 | 额外代价 | 适用 |
|------|---------|---------|------|
| `write()` | 无（只到 page cache） | — | 可容忍少量丢失 |
| **`fdatasync()`** | 数据 + **必需的**元数据（如文件 size 变大） | 1 次日志提交 | 写日志/追加写，性能优先 |
| **`fsync()`** | 数据 + **全部**元数据（mtime、ctime、权限…） | 元数据变更都要提交日志 | 强一致，需保证属性也落盘 |
| `sync()` | **全系统**所有脏页 | 极慢，阻塞全局 | 关机/维护脚本 |
| `syncfs(fd)` | 指定文件系统所有脏页 | 慢 | 批量场景 |

```go
// Go 中：File.Sync() = fsync（不是 fdatasync）
f, _ := os.OpenFile("wal.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
f.Write(record)
f.Sync()   // ← fsync 语义：数据 + 元数据

// 想用 fdatasync 需要系统调用（如 golang.org/x/sys/unix.Fdatasync）
unix.Fdatasync(int(f.Fd()))
```

**面试要答的关键差异：** `fdatasync` 省掉**不需要的元数据提交**。如果是追加写一个已有文件，`fsync` 与 `fdatasync` 差别不大（因为 size 变化本身就是必需元数据）；但如果文件频繁改 mtime 而无实际数据变化，`fsync` 会更贵。

### 5.3 原子替换文件：8 步缺一不可（必考）

```go
// 目标：配置文件/索引文件更新时，任何时刻读方看到的都是完整内容
// 且机器掉电不会留下 0 字节文件

func writeAtomic(path string, data []byte) error {
    dir := filepath.Dir(path)

    // 1) 临时文件放在同目录（必须同文件系统，否则 rename 会 EXDEV 失败）
    tmp, err := os.CreateTemp(dir, ".tmp-*")
    if err != nil {
        return err
    }
    tmpName := tmp.Name()
    defer os.Remove(tmpName) // 失败路径兜底

    // 2) 写入数据
    if _, err := tmp.Write(data); err != nil {
        tmp.Close()
        return err
    }

    // 3) 数据落盘（否则可能 rename 生效但内容为空 → 经典的 0 字节文件）
    if err := tmp.Sync(); err != nil {
        tmp.Close()
        return err
    }
    if err := tmp.Close(); err != nil {
        return err
    }

    // 4) rename 原子替换（POSIX 保证同文件系统 rename 原子）
    if err := os.Rename(tmpName, path); err != nil {
        return err
    }

    // 5) ⭐ fsync 父目录：保证"目录项变更"本身也落盘
    //    否则掉电后可能仍是旧文件名/旧 inode → 文件"凭空消失"
    d, err := os.Open(dir)
    if err != nil {
        return err
    }
    defer d.Close()
    return d.Sync()
}
```

**为什么第 5 步必做？** 常见误区是只 `fsync` 文件。`rename` 修改的是**目录**的内容（目录项），元数据日志未提交时掉电 → 目录项丢失。**这是"配置更新后重启变回旧版本"的真实根因。**

### 5.4 文件系统日志（journaling）与崩溃一致性

| ext4 模式 | 行为 | 一致性 | 性能 |
|-----------|------|-------|------|
| `data=journal` | 数据 + 元数据都进日志 | 最强 | 最慢（数据写两遍） |
| **`data=ordered`（默认）** | 只记元数据日志，但**数据先落盘再提交元数据** | 元数据一致 + 不暴露未初始化数据块 | 平衡 |
| `data=writeback` | 只记元数据日志，数据顺序不限 | 崩溃后可能看到**陈旧数据**（安全风险） | 最快 |

```bash
mount | grep -o "data=[a-z]*"      # 确认当前模式
dumpe2fs -h /dev/sda1 | grep -i "filesystem features"
dmesg | grep -i "EXT4-fs.*recovery"   # 是否发生过日志重放
```

**delayed allocation（延迟分配，ext4 默认）：**
`write()` 时**不立即分配磁盘块**，只在 page cache 标记脏；等回写时才分配。好处是能做出连续块分配决策（减少碎片），坏处是：
- 大量写后 `fsync` 时会**集中分配**，表现为一次长停顿
- `write` 返回成功但**磁盘空间尚未占用**（`df` 不涨）

**`EXT4-fs error` 与 remount-ro：** ext4 检测到元数据不一致时会**只读重挂载**，所有写操作报 `EROFS`。这时**不要直接 `fsck` 运行中的文件系统**——先卸载或用快照。

### 5.5 生产实践（Go 服务）

```go
// 场景 1：WAL / 写前日志（如 etcd、RocksDB、Kafka 自身）
// 每条记录 fsync → 每条都等磁盘（可能是几 ms）→ QPS 只有几百
// ✅ 正确做法：
//    a) Group Commit：攒一小批（如 1ms 或 100 条）一起 fsync，一次提交服务所有等待者
//    b) 用 O_DSYNC + 批量写，把 fsync 次数降到与磁盘能力匹配
//    c) 落盘只保证"顺序日志"，页数据靠 checkpoint 批量刷

// 场景 2：日志文件
// ❌ 每条 log 都 fsync → 吞吐崩掉
// ✅ 交给内核回写（可容忍丢最后几秒日志），只对"审计级别"日志显式 Sync
//    log 库内部有 bufio 缓冲，进程崩溃丢缓冲，机器崩溃丢 page cache

// 场景 3：云盘 / NAS
// 注意：云盘 fsync 触发服务端 flush，单次可能 5~20ms
//       → 对 fsync 频率要有预算（每秒最多几次），否则 P99 直接烂掉
```

```bash
# 观测 fsync 真实代价
iostat -x 1        # 关注 w_await、%util、aqu-sz
# 或直接 strace 看耗时
strace -T -e trace=fsync,fdatasync -p <pid> -f 2>&1 | grep -E "fsync.*<[0-9.]+>"
# 输出示例：fsync(7) = 0 <0.012345>  ← 12ms！每条都这样就是灾难
```

---

## 6. fsync / 持久化 高频追问

**Q1：`write` 返回成功，数据一定不丢吗？**
不一定。数据只在 page cache，进程崩溃不丢、**机器断电会丢**。要强持久必须 `fsync` + `fsync` 父目录。

**Q2：`Close()` 会 fsync 吗？**
不会（除非用 `O_SYNC`/`O_DSYNC` 打开）。`close` 只释放 fd。**这是"程序正常退出但文件内容丢失"的常见误判。**

**Q3：为什么 `O_DIRECT` 常用于数据库？**
绕过 page cache，直接 DMA 到用户缓冲，避免"双份内存 + 回写不可控"。代价：必须按块对齐（通常 512B/4KB），且小随机写会变慢（丧失预读与合并）。

**Q4：`fsync` 之后能保证 SSD 内部落盘吗？**
`fsync` 会把 FUA/flush 传给设备，但**廉价 SSD/U 盘的写缓存可能不诚实**（报告成功但仍在 DRAM 缓冲）。企业级盘 + 掉电保护电容才能保证。这也是"文件系统一致性没问题，但数据损坏"的硬件层原因。

**Q5：ext4 的 `data=ordered` 和 `fsync` 是什么关系？**
`data=ordered` 保证**元数据不会指向未写入的数据**（不会读到脏数据块），但**它不保证你的数据在某个时刻已落盘**——你仍需 `fsync` 来获得"这一刻已持久"的语义。二者层级不同：一个是崩溃一致性，一个是持久性承诺。

**Q6：容器里 `fsync` 会走 overlayfs，有什么坑？**
overlayfs 上的 `fsync` 会下发到**底层文件系统**（upperdir 所在 FS）。写触发 copy-up 时，`fsync` 之前先要复制整个文件 → **一次 `fsync` 变成"复制 1GB + 落盘"，耗时秒级**。数据目录必须挂 volume（见 [容器网络与 OverlayFS](./11-11-container-network-and-overlayfs.md)）。

---

## 延伸阅读

- [Linux man：fsync(2) / fdatasync(2) / sync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html)
- [ext4 与 delayed allocation（LWN）](https://lwn.net/Articles/224829/)
- [POSIX rename 原子性与崩溃一致文件更新（Dan Kegel 论文）](https://www.kernel.org/doc/Documentation/filesystems/)
- [Go os/exec 文档：Cmd.Wait 与进程回收](https://pkg.go.dev/os/exec#Cmd.Wait)
- [tini：容器 PID 1 最小实现](https://github.com/krallin/tini)
