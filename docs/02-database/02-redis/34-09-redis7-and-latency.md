# Redis 7 新特性与阻塞卡顿排查实战

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：Redis Function、Multi-Part AOF、Sharded Pub/Sub、Client Eviction、slowlog、latency monitor、fork COW、AOF fsync

## 面试官考察意图

面试官想区分「只会用 SET/GET」和「真在生产上背过 Redis 锅」的候选人：

1. "你们 Redis 什么版本？7.0 有什么变化？" → 引出 Function / Multi-Part AOF / 分片发布订阅
2. "Redis 突然变慢（P99 抖动）你怎么查？" → 引出阻塞根因定位方法论
3. "Redis 单线程，为什么会卡？" → 引出慢命令、大 key、AOF fsync、fork + COW、Swap 五大元凶

**区分度**在于是否有完整的排查链路：`MONITOR`/`SLOWLOG`/`LATENCY` → 定位到命令 → 再定位到系统层（fork/swap/页缓存）。

---

## 核心答案（30 秒版）

| 问题 | 一句话回答 |
|------|-----------|
| Redis 7.0 最重要的变化？ | ① 用 **Function**（`FUNCTION LOAD`）替代散落的 EVAL 脚本；② **Multi-Part AOF**（base + incr 分离，重写不再全量重放）；③ **Sharded Pub/Sub** 让订阅随集群分片扩展 |
| Redis 为什么卡？ | 单线程执行，任何一条 O(n) 命令、大 key 删除、AOF fsync、fork 阻塞、Swap 抖动都会直接卡住所有请求 |
| 怎么定位慢命令？ | `SLOWLOG GET` 找慢命令 + `INFO commandstats` 看单命令耗时 + `MONITOR` 短时抓包 |
| 怎么定位系统层卡顿？ | `LATENCY DOCTOR` / `LATENCY HISTORY` + 看 `fork` 耗时、`mem_fragmentation_ratio`、Swap 使用 |
| 大 key 怎么安全删？ | 用 `UNLINK`（后台异步释放）替代 `DEL`；并开启 `lazyfree-lazy-*` 系列配置 |

---

## 深度展开

### 1. Redis 7.0 新特性

#### 1.1 Function：脚本的服务端「模块化」

```bash
# 7.0 之前：EVAL 脚本每次都要把整段 Lua 发给服务端，无法复用、无法版本管理
EVAL "return redis.call('GET', KEYS[1])" 1 mykey

# 7.0 之后：先加载一个库（函数），再按名字调用
FUNCTION LOAD "#!lua name=mylib
redis.register_function('my_get', function(keys, args)
  return redis.call('GET', keys[1])
end)"

FCALL my_get 1 mykey
```

**Function 相对 EVAL 的优势：**

| 维度 | EVAL | FUNCTION |
|------|------|----------|
| 脚本存储 | 客户端每次传输全文 | 服务端持久化，`FUNCTION LOAD` 一次 |
| 复用性 | 复制粘贴 | 命名调用 `FCALL` |
| 集群支持 | 需 `EVALSHA` + 脚本缓存，节点间不同步 | 加载后随 `FUNCTION DUMP/RESTORE` 复制 |
| 版本管理 | 无 | 库名 + 版本，可 `FUNCTION LIST` 查看 |

**注意**：Function 里**不能执行随机写命令**（如 `TIME` 之外的非确定性命令受限于 `no-writes`/`allow-oom` 标志），且默认 `allow-oom` 为 false——OOM 时会直接拒绝而非驱逐。

#### 1.2 Multi-Part AOF：重写不再「越写越大」

```
7.0 之前（单文件 AOF）：
  appendonly.aof  ← 重写时要 fork 出子进程把当前数据集**全量**写成新文件，
                    数据量越大重写越慢，且写放大严重。

7.0 之后（Multi-Part AOF）：
  appendonlydir/
    ├── appendonly.aof.1.base.rdb     # 基线（RDB 格式，快照）
    ├── appendonly.aof.1.incr.aof     # 增量命令
    └── appendonly.aof.manifest       # 清单文件，记录各部件与顺序
```

好处：基线用 RDB 格式（重写更快更小），增量单独追加；重写只替换 base 部分，不再需要把整个历史命令重放一遍。

```bash
# 开启（需先开启 AOF）
appendonly yes
aof-use-rdb-preamble yes
aof-timestamp-enabled yes     # 7.0+：AOF 里带时间戳，便于按时间点恢复
```

#### 1.3 Sharded Pub/Sub：让订阅能水平扩展

```
普通 Pub/Sub：消息广播到**所有**节点（集群模式下客户端可连任意节点订阅）
  → 消息量随节点数放大，无法水平扩展

Sharded Pub/Sub（7.0）：
  SSUBSCRIBE / SPUBLISH
  → 频道按 slot 归属到某个分片，消息只在该分片内广播
  → 订阅关系随分片扩展，适合大规模频道场景
```

#### 1.4 Client Eviction：防止「慢客户端」拖垮整个实例

```
问题：输出缓冲区（output buffer）无上限时，一个网络慢的客户端订阅了大流量频道，
      Redis 会为它堆积响应数据，最终把内存吃光。
7.0 新增：maxmemory-clients，按客户端内存用量做驱逐（默认 0=不限制，建议设为 maxmemory 的 1%~5%）。
```

### 2. Redis 阻塞卡顿排查方法论

**第一层：确认是 Redis 卡还是网络/客户端卡**

```bash
redis-cli --latency                 # 本机到 Redis 的往返延迟
redis-cli --latency-history -i 5    # 观察延迟随时间变化
redis-cli --intrinsic-latency 100   # 测本机内核调度引入的固有延迟（Redis 无关）
```

`--intrinsic-latency` 若就很高（>1ms），说明问题在操作系统/机器本身（CPU 争抢、虚拟化），不是 Redis。

**第二层：找慢命令**

```bash
# 慢日志（默认阈值 10ms，只记录执行耗时，不含排队时间）
CONFIG SET slowlog-log-slower-than 5000   # 5ms
SLOWLOG GET 10
SLOWLOG LEN

# 单命令累计耗时排行（定位"哪类命令最耗 CPU"）
INFO commandstats
# cmdstat_keys:calls=... ,usec=...        ← KEYS 是典型的 O(N) 杀手
```

**高频慢命令黑名单：**

| 命令 | 复杂度 | 替代方案 |
|------|--------|----------|
| `KEYS *` | O(N) | `SCAN` 游标分批 |
| `FLUSHALL` / `FLUSHDB` | O(N) | `FLUSHALL ASYNC` / `FLUSHDB ASYNC` |
| `HGETALL` 大 hash | O(N) | `HSCAN` 分批 |
| `DEL` 大 key | O(N) 同步释放 | `UNLINK` 异步释放 |
| `SORT` / `ZRANGEBYLEX` 大集合 | O(N log N) | 建好索引/分页 |
| Lua 脚本里 `KEYS`/大批量循环 | O(N) | 拆小、限长 |

**第三层：系统层元凶**

```bash
INFO persistence
# rdb_last_bgsave_status / aof_last_bgrewrite_status
# rdb_last_cow_size  ← COW 拷贝量，过大说明 fork 期间写多、内存翻倍

INFO memory
# used_memory vs used_memory_rss → mem_fragmentation_ratio
# ratio > 1.5 碎片偏高；< 1 说明在用 Swap（危险！）

INFO stats
# latest_fork_usec  ← 最近一次 fork 耗时，>1s 基本可判定阻塞来源
```

**五大阻塞元凶与对策：**

1. **慢命令**：`SLOWLOG` 定位 → 改写命令 / 拆分批次。
2. **大 key 删除/过期**：`UNLINK` + `lazyfree-lazy-expire/eviction/server-del yes`。
3. **fork + COW**：`vm.overcommit_memory=1`，避免 CPU 超配，控制单实例内存 ≤ 10~16G（fork 耗时随内存增长）。
4. **AOF fsync**：`appendfsync everysec`（默认，每秒一次，最多丢 1s）；always 会严重拖慢；磁盘 IO 打满时考虑换 SSD 或用 `no-appendfsync-on-rewrite yes`。
5. **Swap**：Redis 一旦被换出，延迟从微秒级飙到毫秒级。生产必须 `vm.swappiness=0` 或给容器设置 `memory.limit = maxmemory * 1.5` 留足余量。

---

## 高频追问

### Q1：Redis 6.0 的多线程和 7.0 的变化是同一回事吗？

不是。6.0 引入的 I/O 线程只处理**网络读写**（读请求、写响应），命令执行仍单线程。7.0 的 Sharded Pub/Sub、Function、Multi-Part AOF 是功能/持久化层面的演进。面试若被问「Redis 到底是不是单线程」，标准答法：**命令执行是单线程，网络 I/O 从 6.0 起可多线程**。

### Q2：为什么 Redis 单线程不怕 CPU 瓶颈，反而会被 fork 拖慢？

命令执行本身极快（纯内存 O(1)），单线程足够。但 `fork()` 是一个需要**复制页表**的系统调用，耗时与**进程内存大小成正比**（内存越大页表越大，fork 越慢），且 fork 期间主线程阻塞。BGSAVE/AOF 重写都会 fork，所以「内存特别大的 Redis 实例」反而是最容易卡顿的。

### Q3：`used_memory` 和 `used_memory_rss` 差很多正常吗？

正常但要盯。差值就是内存碎片（jemalloc 分配 + 删除后不归还 OS）。`mem_fragmentation_ratio = rss/used`，1.0~1.5 正常；>1.5 碎片严重，可开 `activedefrag yes`；<1 基本说明发生了 Swap，必须立即处理。

---

## 延伸阅读

- [Redis 7.0 release notes](https://raw.githubusercontent.com/redis/redis/7.0/00-RELEASENOTES)
- [Redis Docs - Redis functions](https://redis.io/docs/manual/programmability/functions/)
- [Redis Docs - Diagnosing latency issues](https://redis.io/docs/management/optimization/latency/)
