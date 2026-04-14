[🏠 首页](../../../README.md) · [🗤️ 数据库](../../README.md) · [🔴 Redis](../README.md)

---

# Redis 持久化机制：RDB / AOF / 混合持久化

## 面试官考察意图

考察候选人对 Redis 数据安全的掌握深度。
初级只能说出"RDB 是快照、AOF 是日志"，高级要能讲清楚 **RDB 和 AOF 的 fork 机制与 COW、写放大问题、AOF 三种同步策略的 tradeoff、混合持久化的实现方式**，并能根据业务场景给出选型建议。

---

## 核心答案（30 秒版）

| 持久化方式 | 原理 | 性能 | 数据完整性 |
|------------|------|------|------------|
| **RDB** | fork 子进程，COW 快照全量数据 | 高（fork 后后台完成） | 丢失最近一次快照后的数据 |
| **AOF** | 写命令追加到日志文件 | 低（每条命令都要写磁盘） | 可配置（最多 1 秒） |
| **混合持久化** | RDB 格式的全量 + AOF 的增量 | 中 | 最佳 |

生产环境推荐：**AOF + 混合持久化 + RDB 定期备份**。

---

## 深度展开

### 1. RDB：定时快照

**原理：fork + Copy-On-Write**

```
主进程（Redis Server）
    │
    ├─ fork() → 子进程（BGSAVE）
    │
    ├─ 主进程：继续处理请求，COW 复制内存页给子进程
    └─ 子进程：遍历内存，生成 .rdb 文件
                    │
                    ▼ 完成后
    新 .rdb 文件覆盖旧文件（或写入新的位置）
```

**COW（Copy-On-Write）核心语义：**
- fork 时，父子进程共享同一份物理内存页
- 父进程修改某页时，内核**复制**该页到新物理地址，再修改
- 子进程看到的仍是 fork 时的内存快照（一致性保证）

**RDB 触发方式：**

```bash
# 1. 手动触发
redis-cli BGSAVE  # 后台异步，不阻塞主线程
redis-cli SAVE     # 同步阻塞（仅在运维紧急场景使用）

# 2. 自动触发（配置在 redis.conf）
save 900 1      # 900 秒内 ≥1 个 key 变化 → BGSAVE
save 300 10     # 300 秒内 ≥10 个 key 变化
save 60 10000   # 60 秒内 ≥10000 个 key 变化
```

**RDB 的问题：**

```
时间 T1: BGSAVE 完成，生成 dump.rdb
时间 T1 + 30min: Redis 宕机
结果：丢失 30 分钟内的数据
```

### 2. AOF：命令日志

**原理：每条写命令追加写入 appendonly.aof**

```bash
# 配置
appendonly yes
appendfilename "appendonly.aof"

# AOF 三种同步策略（性能 vs 安全的 tradeoff）
appendfsync always    # 每条命令都 fsync → 最安全，最慢（~1万 QPS）
appendfsync everysec  # 每秒一次 fsync → 推荐，默认值
appendfsync no        # 不主动 fsync，等 OS 决定 → 最快，最不安全
```

**AOF 重写（Rewrite）：压缩文件**

```bash
# AOF 文件会不断膨胀（SET a 1; SET a 2; SET a 3; → 最终只需 SET a 3）
# bgrewriteaof：后台重写，生成紧凑的 AOF 文件

redis-cli BGREWRITEAOF
```

```
重写流程：
1. fork 子进程
2. 子进程：遍历当前内存数据，生成 SET/KEEP/DEL 等最小命令
3. 写入新 AOF 文件
4. 父进程：继续接收新命令，同时写入 AOF 缓冲区 + 重写缓冲区
5. 完成后，新 AOF 覆盖旧 AOF
```

**AOF 的写放大问题：**

```
Redis 每秒 10 万 QPS
每条命令 ~100 字节
每秒 AOF 日志写入量 = 10万 * 100 = 10MB/s
加上 fsync 开销：
  - everysec：每秒一次 10MB 写入，SSD OK
  - always：每秒 10万次 fsync → 灾难性性能下降
```

### 3. 混合持久化（RDB + AOF）

**Redis 4.0+ 支持，5.0+ 默认开启**

```bash
aof-use-rdb-preamble yes  # 开启混合持久化
```

**混合持久化文件结构：**

```
混合文件：
┌──────────────┐
│    RDB header│  ← 开局一个 RDB 全量快照（快速加载）
│   (binary)   │
├──────────────┤
│   AOF tail   │  ← RDB 之后是 AOF 格式的增量命令
│ (RDB+AOF)    │    set a 1; set b 2; del c;
└──────────────┘
```

**加载流程：**
```
Redis 启动
    │
    ├─ 有混合文件 → 先加载 RDB 部分（快速）
    │              再应用 AOF tail 的增量命令
    │
    └─ 纯 AOF 文件 → 逐行解析 AOF 命令（慢）
```

### 4. RDB vs AOF 选型决策树

```
数据敏感程度？
    │
    ├─ 极高（如支付数据）：AOF everysec + 主从复制（从机备份）
    │
    ├─ 中等（如会话缓存）：AOF everysec 或 混合持久化
    │
    └─ 可丢失（如排行榜）：RDB 定期备份

QPS 要求？
    │
    ├─ >10 万：混合持久化（RDB 为主，AOF 只记录关键操作）
    │
    └─ 一般：混合持久化
```

### 5. 生产经验：主从 + 持久化组合

```
生产最佳实践：
- 主库：开启 AOF everysec + 混合持久化
- 从库：开启 RDB（用于快速故障恢复）+ AOF
- 同时：每日凌晨业务低峰期备份 RDB 文件到 OSS

故障恢复演练：
1. 模拟主库宕机
2. 确认从库数据完整性（LASTSAVE vs 主库 latest save time）
3. 提升从库为主（redis-cli promote）
4. 验证应用重连
```

```bash
# 生产监控脚本：检查 AOF 文件大小异常
AOF_SIZE=$(redis-cli INFO persistence | grep aof_base_size | cut -d: -f2)
AOF_CURRENT=$(redis-cli INFO persistence | grep aof_current_size | cut -d: -f2)
RATIO=$(echo "scale=2; $AOF_CURRENT / $AOF_SIZE" | bc)
if (( $(echo "$RATIO > 10" | bc -l) )); then
    echo "ALERT: AOF 文件异常膨胀，当前/基准=$RATIO"
fi
```

---

## 高频追问

**Q：fork 的子进程会阻塞 Redis 主线程吗？**

不会。fork 是**内核级别的操作**，只复制页表（几 KB），瞬间完成。COW 的实际复制是后续各页面被修改时才发生的，且只复制被改动的页面。但 fork 期间如果 Redis 内存很大（数十 GB），fork 耗时可能达秒级（Linux copy_page_range 遍历所有页表）。

**Q：AOF 文件损坏了怎么办？**

```bash
# 1. 使用 redis-check-aof 修复
redis-check-aof --fix appendonly.aof

# 2. 修复策略：找到最后一个正确的命令，截断之后的内容
#   （可能丢失最后一个命令的部分内容）

# 3. 如果无法修复：
#   - 从从库恢复
#   - 使用 redis-cli --pipe 逐行导入备份
```

**Q：Redis 进程 OOM 时，AOF 重写会怎样？**

Linux OOM Killer 会优先杀掉内存占用最大的进程。Redis AOF 重写期间内存可能膨胀到原来的 2 倍（父进程 + 子进程同时持有 COW 页面）。解决方案：限制 Redis 最大内存（`maxmemory`），或使用 cgroup 限制内存上限。

**Q：主从复制和持久化的关系？**

- **主从复制**：保证**高可用**（机器级别的故障切换）
- **持久化**：保证**数据安全**（进程级别的数据恢复）

两者是互补关系，不是替代关系。没有持久化的主从复制，主库数据仍会在切换后丢失。

---

## 延伸阅读

- [Redis Persistence](https://redis.io/docs/management/persistence/)（官方文档）
- [Redis AOF 重写机制详解](https://segmentfault.com/a/1190000015368468)
- [Copy-On-Write 机制与 Redis fork](https://www.cnblogs.com/zwbg/p/14018317.html)
- [Redis 混合持久化原理](https://juejin.cn/post/6844903869072695309)

---

**[← 上一篇：Redis 数据结构底层实现](./01-data-structures.md)** · **[下一篇：缓存穿透/击穿/雪崩 →](./03-cache-problems.md)**
