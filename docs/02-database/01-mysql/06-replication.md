# MySQL 主从复制原理与生产问题排查

> 面试频率：★★★★☆  考察角度：主从复制原理、半同步复制、延迟处理、GTID

---

## 1. 主从复制原理

### 1.1 复制架构

```
┌─────────┐  binlog   ┌─────────┐  relay-log  ┌─────────┐
│ Master  │ ────────▶ │ Slave IO │ ─────────▶ │Slave SQL│
│  MySQL  │  dump线程  │  Thread  │  relay-log  │ Thread  │
└─────────┘           └─────────┘             └─────────┘
```

### 1.2 复制三步骤

**Step 1：Binlog Dump（Master 侧）**
Master 的 Binlog Dump 线程监听 binlog 变化，当 Slave 连接并请求数据时，Master 将 binlog 内容发送给 Slave。

```sql
-- 查看 Master binlog 信息
SHOW MASTER STATUS;
-- Binlog File: mysql-bin.000123
-- Binlog Position: 4567
```

**Step 2：IO Thread 拉取（Slave 侧）**
Slave 的 IO Thread 连接 Master，请求从指定 position 开始的 binlog 事件。

```
CHANGE MASTER TO 
  MASTER_HOST='master-host',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='xxx',
  MASTER_LOG_FILE='mysql-bin.000123',
  MASTER_LOG_POS=4567;
```

IO Thread 将收到的 binlog 写入本地 **relay-log** 文件。

```sql
-- 查看 Slave 复制状态
SHOW SLAVE STATUS\G

Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: 0   -- 复制延迟，0 表示无延迟
Read_Master_Log_Pos: 4567
```

**Step 3：SQL Thread 重放（Slave 侧）**
SQL Thread 读取 relay-log，按顺序在 Slave 本地重放 SQL。

### 1.3 三种复制模式

| 模式 | 数据安全 | 性能 | 网络要求 |
|------|---------|------|---------|
| **异步复制** | Master 宕机可能丢数据 | 最高 | 低 |
| **半同步复制**（Semi-sync） | Master 宕机最多丢 1 个事务 | 中 | 高 |
| **全同步复制** | 不丢数据 | 最低 | 极高 |

**半同步复制原理**：

```sql
-- Master 侧
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = ON;

-- Slave 侧
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
```

```
Master: 提交事务 → 等待至少一个 Slave ACK → 返回客户端
（等待超时后自动降级为异步）
```

---

## 2. GTID 复制模式

### 2.1 GTID 是什么

GTID（Global Transaction Identifier）= `server_uuid:transaction_id`

```sql
-- 示例
3E11FA47-71CA-11E1-9E33-C80AA9429562:23
  ↑ 服务器唯一 ID              ↑ 事务序号
```

**解决的问题**：
- 传统模式 `MASTER_LOG_FILE + POS` 难以管理，容易出错
- GTID 自动记录已执行的事务，不依赖文件名和位置

### 2.2 GTID 配置

```sql
-- Master 和 Slave 都要开启 GTID
gtid_mode = ON
enforce_gtid_consistency = ON
log_slave_updates = ON

-- Slave 配置
CHANGE MASTER TO 
  MASTER_AUTO_POSITION = 1   -- 不再指定 POS，自动找 GTID 位置
```

### 2.3 GTID + 半同步

```sql
-- 最佳实践：GTID + 半同步
rpl_semi_sync_master_wait_for_slave_count = 1  -- 至少 1 个 Slave ACK
rpl_semi_sync_master_timeout = 1000            -- 1000ms 超时降级
```

---

## 3. 生产常见问题

### 3.1 主从延迟

**原因分析**：

```
1. Master 写入太密集，Slave 重放跟不上（单线程 SQL Thread）
2. 大事务提交（大事务 binlog 大，Slave 重放慢）
3. 网络抖动导致 Slave 拉取 binlog 慢
4. Slave 机器配置低，IO 差
```

**解决方案**：

```sql
-- 1. 大事务拆小事务
-- ❌ 大事务
BEGIN;
INSERT INTO orders (...) VALUES (...); -- 10万条
COMMIT;

-- ✅ 小事务 + 批量
for batch in chunks(1000) {
    INSERT INTO orders (...) VALUES (batch);
}
```

```go
// 2. 并行复制（Parallel Replication）
// 在 Slave 侧开启多线程重放
// MySQL 5.7+ 支持基于组提交的并行复制

SET GLOBAL slave_parallel_workers = 8;          // 8 个 worker 线程
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK'; // 按 Logical Clock 并行
```

```sql
-- 3. 延迟复制（用于数据恢复）
-- Slave 比 Master 慢 1 小时，万一删库可以救命
CHANGE MASTER TO MASTER_DELAY = 3600;
```

### 3.2 从库数据不一致

```sql
-- 排查：从库是否有差异
-- pt-table-checksum 工具对比主从数据
pt-table-checksum h=master-host,u=root,P=3306 --database=mydb

-- 修复：pt-table-sync 同步差异
pt-table-sync --sync-to-master h=slave-host,u=root --database=mydb --table=orders
```

### 3.3 主库宕机，切换从库

```
1. 确认从库追上主库（Show Slave Status，Seconds_Behind_Master = 0）
2. 停止从库复制（STOP SLAVE）
3. 从库设为只读（SET GLOBAL read_only = ON）
4. 修改应用数据库连接指向新主库
5. 通知 DBA 记录新旧主库信息
```

```go
// Go 应用层：支持主从自动切换
func NewDBCluster(masterAddr string, slaveAddrs []string) *gorm.DB {
    // 读操作路由到从库
    // 写操作路由到主库
    // 健康检查：定期 ping 从库，延迟 > 阈值自动摘除
}
```

### 3.4 binlog 刷盘策略导致的丢数据

```sql
-- binlog_sync = 0（默认，性能最高，可能丢事务）
-- binlog_sync = 1（每次事务提交刷盘，最安全）
sync_binlog = 1   -- 配合双一设置，保证不丢数据
innodb_flush_log_at_trx_commit = 1  -- innodb 日志也要刷盘
```

---

## 4. Go 读写分离最佳实践

### 4.1 简单实现：自动路由

```go
type DBCluster struct {
    master *sql.DB
    slaves []*sql.DB
}

func (c *DBCluster) QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
    // 读操作走从库
    slave := c.pickSlave()
    return slave.QueryContext(ctx, query, args...)
}

func (c *DBCluster) ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error) {
    // 写操作走主库
    return c.master.ExecContext(ctx, query, args...)
}

func (c *DBCluster) pickSlave() *sql.DB {
    // 简单轮询，也可以根据延迟选择
    idx := atomic.AddInt32(&c.roundRobin, 1) % int32(len(c.slaves))
    return c.slaves[idx]
}
```

### 4.2 延迟感知读写分离

```go
// 根据 Show Slave Status 的延迟信息选择从库
type SlaveInfo struct {
    addr   string
    delay  time.Duration
}

func (c *DBCluster) pickLeastDelayedSlave() *sql.DB {
    var best *SlaveInfo
    for _, s := range c.monitor.Slaves() {
        if best == nil || s.delay < best.delay {
            best = s
        }
    }
    // 延迟超过阈值，写请求降级到主库
    if best != nil && best.delay > 500*time.Millisecond {
        return c.master  // 降级到主库
    }
    return best.db
}
```

---

## 5. 面试高频追问

**Q：主从延迟的原因有哪些？**
> 大事务（单线程 SQL Thread 重放）、从库机器性能差、网络抖动、从库拉取 binlog 慢。最常见的是大事务导致，建议把大事务拆成小批量。

**Q：如何保证主从数据一致性？**
> 半同步复制保证至少一个从库收到数据；GTID 避免位点错误；读写分离场景接受最终一致性；强一致场景用分布式事务。

**Q：主从切换时如何保证不丢数据？**
> 开启半同步复制（至少 1 个从库 ACK）+ binlog 和 redo log 双刷盘（sync_binlog=1, innodb_flush_log_at_trx_commit=1）。

**Q：从库出现延迟很大怎么处理？**
> 1. SHOW PROCESSLIST 排查哪个 SQL 重放慢；2. 开启并行复制（slave_parallel_workers）；3. 升级从库硬件；4. 延迟复制做兜底。

**Q：GTID 相比传统 POS 有什么优势？**
> GTID 全局唯一，自动定位，不依赖 binlog 文件名和位置；切换主从时不需要找准确的 POS；更容易做主从拓扑管理。
