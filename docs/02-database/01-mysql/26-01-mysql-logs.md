[🏠 首页](../../../README.md) · [🗤️ 数据库](../../README.md) · [🐬 MySQL](../README.md)

---

# MySQL 三大日志：binlog / redo log / undo log 与两阶段提交

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：binlog、redo log、undo log、两阶段提交、WAL、crash-safe、XID

## 面试官考察意图

三大日志是 MySQL 面试"深水区"的第一站，几乎每个大厂必问。面试官想确认：

1. 能否说清三种日志各自的作用、写入时机、所属层级（Server 层 vs 引擎层）
2. 能否讲透 **redo log 与 binlog 的区别**（物理日志 vs 逻辑日志、循环写 vs 追加写）
3. 能否解释 **两阶段提交** 为什么能保证 crash-safe
4. 能否结合崩溃恢复场景说明 prepare/commit 的判断逻辑

---

## 核心答案（30 秒版）

MySQL 有三大日志：

| 日志 | 层级 | 类型 | 作用 | 写入方式 |
|------|------|------|------|----------|
| **binlog** | Server 层 | 逻辑日志 | 主从复制、数据恢复 | 追加写（归档） |
| **redo log** | InnoDB 引擎层 | 物理日志 | 崩溃恢复（crash-safe） | 循环写（固定大小） |
| **undo log** | InnoDB 引擎层 | 逻辑日志 | 事务回滚、MVCC 版本链 | 追加写 |

**两阶段提交**：事务提交时，redo log 先写 **prepare** 状态 → 再写 binlog → 最后把 redo log 改为 **commit** 状态。崩溃恢复时，如果 redo log 是 prepare 状态，就去 binlog 里查有没有对应的 XID：有则提交，没有则回滚。这样保证 redo log 和 binlog 逻辑一致，主从复制不会丢数据。

---

## 深度展开

### 1. 三种日志逐一拆解

#### 1.1 redo log（重做日志）：崩溃恢复的"账本"

- **属于 InnoDB 引擎层**，记录的是"数据页被改成了什么样"（物理变更），比如"页号 5 偏移量 100 处写入 1"
- 为什么需要它？**WAL（Write-Ahead Logging）**：先写日志、后写数据。因为直接改数据页是随机 IO，慢；而日志是顺序写，快
- 数据更新流程：改 Buffer Pool 中的内存页 → 标记脏页 → 写 redo log（顺序写）→ 后台线程择机把脏页刷盘
- **固定大小、循环写**：`innodb_log_file_size` 控制，写满后从头覆盖（会触发 checkpoint）
- 关键参数：`innodb_flush_log_at_trx_commit`
  - `0`：每秒刷盘，性能最好但可能丢 1 秒数据
  - `1`：每次提交都刷盘，最安全（默认）
  - `2`：每次提交写 OS cache，每秒刷盘，MySQL 挂了不丢、OS 挂了丢 1 秒

```
redo log 循环写示意：

[checkpoint] → [写入位置 write pos]
     ↓              ↓
┌────┬────┬────┬────┬────┬────┬────┐
│ 已 │ 已 │ 可 │ 可 │ 新 │ 新 │ 已 │
│ 刷 │ 刷 │ 覆 │ 覆 │ 写 │ 写 │ 刷 │
│ 盘 │ 盘 │ 盖 │ 盖 │ 入 │ 入 │ 盘 │
└────┴────┴────┴────┴────┴────┴────┘
     └─────── 可覆盖区 ──────┘
```

#### 1.2 binlog（归档日志）：复制的"剧本"

- **属于 MySQL Server 层**（所有引擎共用），记录的是"执行了哪条语句/哪行数据变了"（逻辑变更）
- 三种格式：
  - **STATEMENT**：记录原始 SQL，体积小但主从可能不一致（如 `NOW()`）
  - **ROW**：记录每行变更前后值，最准确，MySQL 8.0 默认，体积大
  - **MIXED**：混合模式，自动判断
- **追加写、可归档**：完整记录所有历史变更，用于主从复制 + 基于时间点的数据恢复
- 关键参数：`sync_binlog`
  - `0`：不主动刷盘，依赖 OS
  - `1`：每次事务提交刷盘（最安全，配合 redo log 的 1 才能双保险）

#### 1.3 undo log（回滚日志）：后悔药 + MVCC 版本链

- **属于 InnoDB 引擎层**，记录"怎么改回去"（逆操作）
- 两大作用：
  1. **事务回滚**：事务失败时，用 undo log 把数据还原到修改前
  2. **MVCC 版本链**：每条记录的隐藏字段 `roll_pointer` 指向 undo log 中的上一个版本，构成版本链，配合 ReadView 实现快照读
- 数据删除时不会立即清掉 undo log，要等没有事务引用它才由 purge 线程清理

### 2. 两阶段提交（Two-Phase Commit）

#### 2.1 为什么需要？

redo log 和 binlog 是**两份独立的日志**，如果写入时机不对，崩溃后数据就不一致：

```
场景：执行 UPDATE t SET c=1 WHERE id=2

情况 A：先写 redo log，再写 binlog
  写完 redo log 后崩溃 → binlog 没有这条记录
  → 主库用 redo log 恢复：c=1
  → 从库用 binlog 恢复：c=0（丢了更新！）❌

情况 B：先写 binlog，再写 redo log
  写完 binlog 后崩溃 → redo log 没有这条记录
  → 主库认为事务没提交：c=0
  → 从库用 binlog 恢复：c=1（多了更新！）❌
```

两阶段提交就是为了解决这个矛盾。

#### 2.2 两阶段提交的完整流程

```
事务提交时（以 UPDATE 为例）：

阶段一（prepare）：
  InnoDB 把 redo log 写入磁盘，状态置为 prepare，
  同时记录一个 XID（事务 ID，用于和 binlog 关联）

阶段二（commit）：
  ① 把 XID 写入 binlog，并刷盘（sync_binlog=1）
  ② 调用 InnoDB 提交事务接口，把 redo log 状态改为 commit
     （此步无需落盘，只要 binlog 写成功即可）

崩溃恢复时的判断：
  redo log 是 commit 状态 → 事务成功，直接提交
  redo log 是 prepare 状态 → 去 binlog 找 XID：
      binlog 有对应 XID → 事务成功，提交（binlog 完整）
      binlog 没有 XID   → 事务失败，回滚
```

**核心记忆点：以 binlog 写成功为事务提交成功的标志**。因为 binlog 写成功了，redo log 里一定能查到对应的 XID，两日志就一致了。

#### 2.3 组提交（Group Commit）

高并发下每个事务都刷盘太慢，MySQL 引入 **组提交**：多个事务的 prepare 阶段完成后，可以**一次性刷盘 binlog**，减少 fsync 次数，提升吞吐。这也是为什么 binlog 刷盘时机实际上是在"提交时刻批量进行"。

### 3. 一条 UPDATE 语句的完整日志流转

```
UPDATE t SET c=1 WHERE id=2;

① Server 层解析 SQL，找到对应行
② InnoDB 把 id=2 这行读入 Buffer Pool（若不在内存）
③ 在内存中修改 c=1，标记为脏页
④ 记录 undo log（保存修改前的值，用于回滚/MVCC）
⑤ 写 redo log（物理变更，prepare 阶段在提交时落盘）
⑥ Server 层记录 binlog（先缓存在 binlog cache）
⑦ 事务提交：
    - redo log prepare 落盘
    - binlog 落盘
    - redo log 改为 commit
⑧ 后台线程择机把脏页刷到磁盘（此时才真正改数据文件）
```

注意：**刷脏页的时机和事务提交无关**，是后台异步进行的，这正是 WAL 的核心思想——把随机写变成顺序写。

### 4. 高频追问

**Q1：redo log 和 binlog 有什么区别？**

| 维度 | redo log | binlog |
|------|----------|--------|
| 所属层级 | InnoDB 引擎层 | Server 层 |
| 日志类型 | 物理日志（页级变更） | 逻辑日志（语句/行级变更） |
| 写入方式 | 循环写，固定大小 | 追加写，可归档 |
| 作用 | 崩溃恢复（crash-safe） | 主从复制、数据恢复 |
| 记录内容 | 数据页的具体修改 | SQL 语句或行数据前后值 |

**Q2：MySQL 怎么知道 binlog 是完整的？**
- STATEMENT 格式：最后有 COMMIT 标记
- ROW 格式：最后有 XID event
崩溃恢复时检查 binlog 文件末尾是否完整，不完整的事务会回滚。

**Q3：undo log 什么时候被清理？**
由后台 purge 线程清理，只有当 undo log 对应的版本不再被任何活跃事务/ReadView 引用时才删除。

---

## 面试话术

- **初级**："redo log 是 InnoDB 的崩溃恢复日志，binlog 是 Server 层的归档日志，undo log 用于回滚和 MVCC。"
- **中级**："两阶段提交就是 redo log 先 prepare，再写 binlog，最后 commit。崩溃恢复时看 binlog 里有没有对应 XID，有就提交没有就回滚，保证两份日志一致。"
- **高级**："redo log 是物理日志循环写、binlog 是逻辑日志追加写，两者作用不同。两阶段提交的本质是拿 binlog 作为提交成功的唯一标准。生产上 `innodb_flush_log_at_trx_commit=1` 和 `sync_binlog=1` 双 1 配置才能保证不丢数据，但性能会下降，可以用组提交来缓解。"

---

## 关联阅读

- [事务、隔离级别与 MVCC](./11-02-transaction.md)
- [Change Buffer：二级索引写优化](./17-11-change-buffer.md)
- [主从复制原理](./07-06-replication.md)
- [一条 SQL 的执行流程](./27-02-sql-execution.md)
