# MySQL 主从复制：原理、配置与生产问题排查

## 面试官考察意图

主从复制是中高级 MySQL 面试必考项。
初级只知道"主库写、从库读"，高级要能讲清楚 **binlog 的三种格式及区别、主从复制的完整数据流、半同步复发的原理、复制延迟的原因及解决方案**，并能结合生产问题给出可操作的排查思路。

---

## 核心答案（30 秒版）

MySQL 主从复制基于 **binlog** 实现：
- 主库将数据变更记录到 binlog
- 从库通过 **IO Thread** 拉取主库 binlog，写入 **relay log**
- 从库通过 **SQL Thread** 重放 relay log，实现数据同步

复制方式有三种：异步复制（可能丢数据）、半同步复制（主库等至少一个从库确认）、全同步复制（性能差）。

---

## 深度展开

### 1. binlog 的三种格式

| 格式 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **ROW** | 记录每一行数据的变化 | 精确恢复，不丢数据；MySQL 5.7+默认 | binlog 体积大，高并发时主库压力大 | 生产环境推荐 |
| **STATEMENT** | 记录执行的 SQL 语句 | binlog 体积小，可审计 | 某些函数（NOW()、UUID()）结果不一致 | 低并发、需审计场景 |
| **MIXED** | 混用 ROW 和 STATEMENT | 平衡安全性和体积 | 边界情况仍可能出问题 | MySQL 5.1 过渡方案 |

```sql
-- 查看当前 binlog 格式
SHOW VARIABLES LIKE 'binlog_format';

-- 设置 binlog 格式（需重启或重新连接）
SET GLOBAL binlog_format = 'ROW';
```

> **面试追问**：为什么 MySQL 5.7+ 默认用 ROW？
> 答：STATEMENT 格式在主从复制时，从库执行相同 SQL 可能产生不同结果（如 UUID()、NOW()、Rand() 等非确定性函数）。ROW 格式精确记录数据变化，虽然 binlog 大，但可靠性高。

### 2. 主从复制完整数据流

```
主库:
  ① 事务提交 → 写 redo log（prepare）
  ② 写 binlog（顺序追加，性能高）
  ③ 事务提交 → 写 redo log（commit）
  ④ 通知存储引擎可以提交

从库:
  ① IO Thread：连接主库，拉取新的 binlog 事件，写入 relay log
  ② SQL Thread：读取 relay log，在从库重放（replay）事件
```

关键线程：

| 线程 | 位置 | 职责 |
|------|------|------|
| Binlog Dump | 主库 | 监听 binlog 变化，发送给从库 |
| IO Thread | 从库 | 拉取主库 binlog，写 relay log |
| SQL Thread | 从库 | 重放 relay log |

```sql
-- 在从库查看复制状态
SHOW SLAVE STATUS\G
-- 关键字段：
-- Slave_IO_Running: IO Thread 是否运行
-- Slave_SQL_Running: SQL Thread 是否运行
-- Seconds_Behind_Master: 复制延迟秒数
-- Exec_Master_Log_Pos: 已重放的位置
-- Relay_Log_Space: relay log 总大小
```

### 3. 复制方式对比

#### 3.1 异步复制（默认）

```
主库事务提交 → 写 binlog → 返回客户端成功
               ↓
          （不等从库确认）
```

**问题**：主库崩溃时，已提交的数据可能未复制到从库，导致数据丢失。

#### 3.2 半同步复制（Semi-sync）

```
主库事务提交 → 写 binlog → 等待至少一个从库写入 relay log → 返回成功
```

```sql
-- 安装半同步插件（主库+从库都要装）
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

-- 启用半同步
SET GLOBAL rpl_semi_sync_master_enabled = ON;
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
```

- **等待超时**：如果从库超过 `rpl_semi_sync_master_timeout`（默认 10s）没确认，自动降级为异步复制
- **性能影响**：主库事务提交需多一次网络 RTT，高并发场景延迟明显

#### 3.3 全同步复制

主库等待**所有从库**都重放完成后才返回。性能太差，实际很少用。

### 4. GTID 复制模式

MySQL 5.6+ 支持 **GTID（Global Transaction Identifier）**，每个事务有唯一 ID，复制更可靠。

```sql
-- 开启 GTID
gtid_mode = ON
enforce_gtid_consistency = ON
```

**传统 position 复制的痛点**：
- 主库 DDL 或重启后 binlog 文件名/位置可能变化
- 手工指定位置容易出错

**GTID 的优势**：
- 事务有唯一 ID，无需关心 binlog 文件名
- 从库自动找主库对应位置
- 切换主从时更简单：`CHANGE MASTER TO MASTER_AUTO_POSITION = 1`

```sql
-- GTID 复制下切换主库
CHANGE MASTER TO
  MASTER_HOST = 'new_master',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'password',
  MASTER_AUTO_POSITION = 1;
```

### 5. 复制延迟原因与解决方案

**复制延迟**是最常见的生产问题，表现为 `Seconds_Behind_Master` 不为 0。

#### 原因分析

| 原因 | 表现 | 排查方法 |
|------|------|---------|
| 从库并发能力弱 | SQL Thread 单线程重放 | 开启**并行复制** |
| 大事务 | 主库一次变更太多数据 | 拆分为小事务 |
| 从库服务器负载高 | 硬件资源不足 | 升级硬件/优化负载 |
| 网络延迟 | 主从网络不稳定 | 检查网络质量 |
| 从库执行太慢 | 缺少索引或 SQL 写的有问题 | EXPLAIN 分析 |

#### 并行复制（多线程）

MySQL 5.6+ 支持多线程重放，根据**库名**或**事务组**并行回放。

```sql
-- MySQL 5.7 多线程复制（基于库）
slave_parallel_type = 'DATABASE'   -- 按库并行
slave_parallel_workers = 8         -- 8 个 worker 线程

-- MySQL 5.7.22+ WRITESET 并行（按行并行，效果更好）
slave_parallel_type = 'LOGICAL_CLOCK'
slave_parallel_workers = 16
```

> **面试追问**：并行复制可能有什么问题？
> 答：如果主库并发很高，从库可能出现乱序。如果业务对执行顺序敏感，需要仔细评估并行度。MySQL 8.0 的 `slave_preserve_commit_order = ON` 可以保证提交顺序。

### 6. 生产问题与解决方案

#### Q1：主从数据不一致怎么办？

1. 确认不一致的范围（哪些表）
2. 常用修复方案：
   - 从库 `SET GLOBAL read_only = ON`，重建从库（mysqldump + CHANGE MASTER TO）
   - 使用 pt-table-checksum 定位不一致的表
   - 使用 pt-table-sync 修复不一致数据

```bash
# 使用 pt-table-checksum 检查一致性
pt-table-checksum h=master_host,u=root,p=password --databases=myapp
```

#### Q2：主库 binlog 太大，占满磁盘怎么办？

1. 定期清理：`PURGE BINARY LOGS BEFORE '2024-01-01 00:00:00'`
2. 设置 expire_logs_days 自动清理
3. 临时扩大 binlog 空间，同时分析哪些大表适合做归档

```sql
-- 自动清理 7 天前的 binlog
SET GLOBAL expire_logs_days = 7;
```

#### Q3：如何在线平滑切换主库？

1. 关闭所有应用写主库
2. 确认从库全部追上主库（`SHOW SLAVE STATUS`，`Seconds_Behind_Master = 0`）
3. 从库 `STOP SLAVE` + `RESET SLAVE`
4. 从库关闭只读 `SET GLOBAL read_only = OFF`
5. 应用切换连接新的主库

> **推荐工具**：MySQL 5.7+ 的 MGR（Group Replication）或 Orchestrator 可自动处理主从切换。

---

## 高频追问

**Q：主从复制和读写分离有什么坑？**
A：主要坑是**复制延迟**导致的数据不一致。比如用户下单后立刻查订单，可能查到从库没有的订单。解决方案：强一致性场景走主库，弱一致性场景从库读+延迟告警。

**Q：binlog 和 redo log 的区别？**
A：redo log 是 InnoDB 引擎的物理日志，保证事务持久性（崩溃恢复）；binlog 是 Server 层的逻辑日志，用于主从复制和数据恢复。写入顺序：事务提交 → 写 binlog → 写 redo log（commit）。

**Q：如何监控主从复制状态？**
A：用 `SHOW SLAVE STATUS` 配合 Prometheus MySQL Exporter，上报到 Grafana 监控 `Seconds_Behind_Master`、IO/SQL Thread 状态、relay log 空间。

---

## 延伸阅读

- [MySQL 官方文档 - Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [MySQL 官方文档 - Binary Logging](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)
- 《高性能 MySQL》第 4 版，第 10 章
