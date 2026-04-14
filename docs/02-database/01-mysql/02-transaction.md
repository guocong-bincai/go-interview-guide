[🏠 首页](../../../README.md) · [🗤️ 数据库](../../README.md) · [🐬 MySQL](../README.md)

---

# MySQL 事务、隔离级别与 MVCC

## 面试官考察意图

事务是后端面试必考项，且追问层次非常深。
初级只背"ACID + 四种隔离级别"，高级要能讲清楚 **MVCC 如何实现可重复读、undo log 的版本链、ReadView 的生成时机**，以及**当前读 vs 快照读的区别**，并结合生产中遇到的幻读、死锁场景给出解决方案。

---

## 核心答案（30 秒版）

InnoDB 通过 **MVCC（多版本并发控制）** 实现读写不阻塞：
- 写操作生成 **undo log 版本链**，旧版本数据不立即删除
- 读操作生成 **ReadView**，根据事务可见性规则从版本链中找到合适的版本
- 默认隔离级别 **可重复读（RR）** 下，ReadView 在事务开始后**第一次读时**生成，整个事务看到的是一个一致性快照

---

## 深度展开

### 1. ACID 特性

| 特性 | 说明 | InnoDB 实现 |
|------|------|-------------|
| **原子性 Atomicity** | 事务要么全部成功，要么全部回滚 | undo log |
| **一致性 Consistency** | 事务前后数据满足业务约束 | 原子性+隔离性+业务逻辑保证 |
| **隔离性 Isolation** | 并发事务互不干扰 | MVCC + 锁 |
| **持久性 Durability** | 提交后数据不丢失 | redo log（WAL 机制） |

### 2. 四种隔离级别与并发问题

```
并发问题：
- 脏读：读到其他事务未提交的数据
- 不可重复读：同一事务两次读同一行，值不同（其他事务已提交修改）
- 幻读：同一事务两次范围查询，行数不同（其他事务已提交插入）
```

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|----------|------|-----------|------|---------|
| READ UNCOMMITTED | ✅ 可能 | ✅ 可能 | ✅ 可能 | 无控制 |
| **READ COMMITTED** | ❌ 解决 | ✅ 可能 | ✅ 可能 | MVCC（每次读生成 ReadView）|
| **REPEATABLE READ**（默认）| ❌ 解决 | ❌ 解决 | ⚠️ 部分解决 | MVCC + 间隙锁 |
| SERIALIZABLE | ❌ 解决 | ❌ 解决 | ❌ 解决 | 读加共享锁，串行执行 |

> InnoDB 的 RR 通过**间隙锁（Gap Lock）**解决了大部分幻读场景，但**快照读无法防止"先快照读后当前读"的幻读**（见后文）。

### 3. MVCC 原理：版本链 + ReadView

#### 3.1 行记录的隐藏字段

每行数据有 3 个隐藏列：

```
┌─────────┬─────────┬──────────────┬─────────────┐
│ 用户数据 │  DB_TRX_ID  │  DB_ROLL_PTR │  DB_ROW_ID  │
│         │（最近修改的事务ID）│（undo log 指针）│（无主键时用）│
└─────────┴─────────┴──────────────┴─────────────┘
```

#### 3.2 undo log 版本链

每次修改数据，旧版本写入 undo log，新版本的 `DB_ROLL_PTR` 指向旧版本：

```
当前版本（name='Bob', trx_id=100）
    │ DB_ROLL_PTR
    ▼
旧版本1（name='Alice', trx_id=50）
    │ DB_ROLL_PTR
    ▼
旧版本2（name='Tom', trx_id=10）
    │
    ▼ NULL（最初版本）
```

#### 3.3 ReadView 结构

ReadView 记录生成时刻活跃事务的快照：

```go
// ReadView 伪结构
type ReadView struct {
    m_low_limit_id  uint64  // 高水位：生成时下一个将分配的事务ID，≥此值的事务不可见
    m_up_limit_id   uint64  // 低水位：活跃事务中最小的ID，<此值的事务已提交可见
    m_ids           []uint64 // 生成时活跃（未提交）的事务ID列表
    m_creator_trx_id uint64 // 创建此 ReadView 的事务ID
}
```

#### 3.4 可见性判断规则

对版本链中每个版本的 `trx_id`，按以下规则判断：

```
if trx_id == creator_trx_id:
    → 自己修改的，可见 ✅

if trx_id < m_up_limit_id:
    → 生成 ReadView 前已提交，可见 ✅

if trx_id >= m_low_limit_id:
    → 生成 ReadView 后才开始的事务，不可见 ❌

if trx_id in m_ids:
    → 生成时还未提交的事务，不可见 ❌
else:
    → 在活跃列表范围内但已提交，可见 ✅

若不可见：沿 DB_ROLL_PTR 找上一个版本，重复判断
```

#### 3.5 RC vs RR 的关键差异

```
READ COMMITTED（RC）：
    每次 SELECT 都生成新的 ReadView
    → 能看到其他事务最新提交的数据
    → 两次读可能不同（不可重复读）

REPEATABLE READ（RR）：
    事务内第一次 SELECT 生成 ReadView，之后复用
    → 整个事务看到的是同一个快照
    → 两次读结果相同（可重复读）
```

### 4. 当前读 vs 快照读

```
快照读（Snapshot Read）：普通 SELECT
    → 走 MVCC，读历史版本，不加锁

当前读（Current Read）：读最新版本，加锁
    SELECT ... FOR UPDATE          → 加排他锁（X锁）
    SELECT ... LOCK IN SHARE MODE  → 加共享锁（S锁）
    INSERT / UPDATE / DELETE        → 加排他锁
```

**RR 隔离级别下，MVCC 并不能完全防止幻读：**

```sql
-- 事务A（RR 隔离级别）
BEGIN;
SELECT * FROM t WHERE id > 10;   -- 快照读，返回0行，生成 ReadView

-- 事务B 此时 INSERT INTO t VALUES(11, ...); COMMIT;

-- 事务A 再次快照读：还是0行（ReadView 快照，看不到 B 的插入）✅
SELECT * FROM t WHERE id > 10;

-- 但当前读：看到了 B 插入的数据！出现幻读 ⚠️
SELECT * FROM t WHERE id > 10 FOR UPDATE;  -- 返回1行
```

**解决方案：间隙锁（Gap Lock）**

```sql
-- 加了 FOR UPDATE 后，InnoDB 会加间隙锁，阻止其他事务在 id>10 范围内插入
SELECT * FROM t WHERE id > 10 FOR UPDATE;
-- 此时事务B的 INSERT 会被阻塞，直到事务A提交
```

### 5. 锁机制

#### 5.1 行锁类型

| 锁类型 | 说明 | 使用场景 |
|--------|------|---------|
| **Record Lock** | 锁定单行记录 | 等值查询命中索引 |
| **Gap Lock** | 锁定索引间隙，不锁记录本身 | 防止幻读，RR 级别 |
| **Next-Key Lock** | Record Lock + Gap Lock（左开右闭区间） | InnoDB RR 默认的行锁 |
| **Insert Intention Lock** | 插入意向锁，Gap 锁的特殊形式 | INSERT 时申请 |

```
例：表中 id 有 1, 5, 10
Next-Key Lock 的区间：(-∞,1], (1,5], (5,10], (10,+∞)

WHERE id = 5 FOR UPDATE：
  命中记录 → Record Lock(5) + Gap Lock(1,5)
  即 Next-Key Lock (1,5]
  防止其他事务在 1~5 之间插入
```

#### 5.2 死锁案例与处理

```sql
-- 经典死锁：两个事务反向加锁

-- 事务A：
UPDATE t SET v=1 WHERE id=1;  -- 加 X 锁在 id=1
UPDATE t SET v=1 WHERE id=2;  -- 等待 id=2 的 X 锁

-- 事务B（同时执行）：
UPDATE t SET v=1 WHERE id=2;  -- 加 X 锁在 id=2
UPDATE t SET v=1 WHERE id=1;  -- 等待 id=1 的 X 锁 → 死锁！
```

InnoDB 死锁检测（wait-for graph）：

```
InnoDB 自动检测死锁，选择代价最小的事务回滚（error 1213）
业务层需要捕获此错误并重试

// Go 示例：检测死锁并重试
func execWithRetry(db *sql.DB, fn func(*sql.Tx) error) error {
    const maxRetry = 3
    for i := 0; i < maxRetry; i++ {
        tx, _ := db.Begin()
        err := fn(tx)
        if err == nil {
            return tx.Commit()
        }
        tx.Rollback()
        // 1213 = ER_LOCK_DEADLOCK
        if mysqlErr, ok := err.(*mysql.MySQLError); ok && mysqlErr.Number == 1213 {
            time.Sleep(time.Millisecond * time.Duration(10*(i+1)))
            continue
        }
        return err
    }
    return errors.New("max retry exceeded")
}
```

**避免死锁的实践：**
1. 所有事务按**相同顺序**加锁（如按主键从小到大）
2. 事务尽量短，持锁时间要短
3. 大事务拆成小事务
4. 尽量用索引查询，避免全表扫描导致表锁

### 6. redo log 与 wal 机制

```
写数据的流程（WAL：Write-Ahead Logging）：

修改内存中的数据页（Buffer Pool）
    │
    ▼
写 redo log（顺序写，极快）
    │
    ▼
事务提交（redo log 落盘，innodb_flush_log_at_trx_commit 控制时机）
    │
    ▼  （后台异步）
脏页刷回磁盘

宕机恢复：redo log 保证已提交事务不丢，undo log 回滚未提交事务
```

```
innodb_flush_log_at_trx_commit：
  0 = 每秒刷一次（最快，宕机丢最多1秒数据）
  1 = 每次提交都刷（最安全，默认值）
  2 = 写到 OS cache，每秒刷（折中）
```

---

## 高频追问

**Q：MVCC 能解决幻读吗？**

快照读（普通 SELECT）能解决幻读。但如果事务中混用快照读和当前读，仍可能出现幻读。完全解决幻读需要使用当前读（`SELECT FOR UPDATE`）配合间隙锁，或使用 SERIALIZABLE 隔离级别。

**Q：binlog 和 redo log 的区别？**

| | redo log | binlog |
|---|---|---|
| 归属 | InnoDB 引擎层 | MySQL Server 层 |
| 格式 | 物理日志（数据页的修改） | 逻辑日志（SQL 或行变更） |
| 大小 | 固定大小，循环写 | 追加写，可归档 |
| 用途 | 崩溃恢复 | 主从复制、数据恢复 |

**Q：长事务有什么危害？**

1. **undo log 无法清理**：其他事务的修改对它不可见，版本链越来越长
2. **持锁时间长**：阻塞其他事务，导致锁等待超时
3. **回滚成本高**：执行时间越长，回滚代价越大

```sql
-- 查找长事务
SELECT * FROM information_schema.innodb_trx
WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60
ORDER BY trx_started;
```

**Q：select count(*) 为什么慢？**

InnoDB 不存储表的行数（因为 MVCC，不同事务看到的行数可能不同），每次 `count(*)` 都需要扫描。
优化方案：
- 小表用 `count(*)`，通常走最小的二级索引（不用回表）
- 大表维护独立的计数器（Redis 或单独的 count 表）

---

## 延伸阅读

- [InnoDB MVCC 源码分析](https://github.com/mysql/mysql-server/blob/trunk/storage/innobase/read/read0read.cc)
- [《MySQL 技术内幕：InnoDB 存储引擎》第 6、7 章]
- [MySQL 官方文档：InnoDB Transaction Model](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html)

---

**[← 上一篇：MySQL 索引原理与优化](./01-index.md)**

---

## 更新记录

- v1.3（2026-04-14）：修复 README 状态标记，修正 transaction.md 导航链接。标记以下内容为已完成：锁机制深度、OOM 排查、CPU 飙升排查、goroutine 泄漏（原有文件内容与 README 不同步）。版本 v1.2 → v1.3。
