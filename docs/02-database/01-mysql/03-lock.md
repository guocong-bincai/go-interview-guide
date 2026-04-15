# MySQL 锁机制与死锁排查

> 面试频率：★★★★☆  考察角度：锁的类型、行锁 vs 表锁、死锁场景与预防

---

## 1. 锁的类型概览

### 1.1 按锁粒度划分

| 粒度 | InnoDB | MyISAM |
|------|--------|--------|
| 表级锁 | ✅（意向锁） | ✅（表级读写锁） |
| 行级锁 | ✅（Record Lock） | ❌ |
| 间隙锁 | ✅（Gap Lock） | ❌ |
| 临键锁 | ✅（Next-Key Lock） | ❌ |

**InnoDB 行锁实现**：InnoDB 的行锁是通过在索引记录上加锁实现的。如果 UPDATE 语句没有命中索引，会升级为**表锁**。

```sql
-- 命中索引：行锁
UPDATE users SET name='Tom' WHERE id = 1;

-- 未命中索引：表锁
UPDATE users SET name='Tom' WHERE name='Tom';  -- 全表扫描
```

### 1.2 按锁模式划分

```sql
-- 共享锁（S锁）：允许事务读取一行
SELECT ... LOCK IN SHARE MODE;

-- 排他锁（X锁）：允许事务更新或删除一行
SELECT ... FOR UPDATE;
INSERT / UPDATE / DELETE 自动加 X 锁;
```

### 1.3 意向锁（Intention Lock）

InnoDB 自动维护**意向锁**，避免遍历检查表是否有行锁：

- **IS 意向共享锁**：事务将要获取某些行的 S 锁
- **IX 意向排他锁**：事务将要获取某些行的 X 锁

```sql
-- 加行锁前，数据库自动加 IX 锁
SELECT * FROM orders WHERE id = 100 FOR UPDATE;  -- IX + X on record
```

**兼容矩阵**：

| | IS | IX | S | X |
|--|----|----|---|---|
| IS | ✅ | ✅ | ✅ | ❌ |
| IX | ✅ | ✅ | ❌ | ❌ |
| S | ✅ | ❌ | ✅ | ❌ |
| X | ❌ | ❌ | ❌ | ❌ |

---

## 2. 记录锁（Record Lock）

锁定**索引记录**本身。

```sql
-- 锁住 id = 5 的记录
SELECT * FROM users WHERE id = 5 FOR UPDATE;
```

即使 id 字段没有显式建立索引，InnoDB 会自动使用隐式主键索引。

---

## 3. 间隙锁（Gap Lock）

锁定**索引记录之间的间隙**，防止其他事务在间隙中插入数据。

**什么情况下触发间隙锁？**

- 使用 `REPEATABLE READ` 隔离级别
- 范围查询时：`WHERE id > 10 AND id < 20`

```sql
-- 锁定 (10, 20) 这个间隙，其他事务无法插入 id=15 的记录
SELECT * FROM users WHERE id > 10 AND id < 20 FOR UPDATE;
```

**间隙锁的作用**：防止**幻读**（Phantom Read）。

### 3.1 间隙锁的 bug 场景

**生产问题**：使用范围 DELETE 可能锁住大量间隙，导致严重阻塞。

```sql
-- 危险：大批量范围删除
DELETE FROM orders WHERE create_time < '2024-01-01';
-- 可能锁住几个月的数据间隙，线上卡死
```

**正确做法**：分批删除，控制每批影响行数。

```go
func BatchDeleteOrders(db *sql.DB, before time.Time, batchSize int) error {
    for {
        result, err := db.Exec(
            "DELETE FROM orders WHERE create_time < ? LIMIT 1000", before)
        if err != nil {
            return err
        }
        affected, _ := result.RowsAffected()
        if affected == 0 {
            break
        }
        time.Sleep(10 * time.Millisecond) // 让出锁给其他事务
    }
    return nil
}
```

---

## 4. 临键锁（Next-Key Lock）

**Next-Key Lock = Record Lock + Gap Lock**，是 RR 隔离级别下 InnoDB 的默认加锁方式。

锁定**记录本身 + 记录前的间隙**：

```sql
SELECT * FROM users WHERE id = 10 FOR UPDATE;
-- 锁定：(-∞, 10] 闭区间，以及 (10, 20) 间隙
```

**唯一索引的优化**：对于唯一索引的等值查询，Next-Key Lock 退化为 Record Lock。

```sql
-- id 是唯一索引
SELECT * FROM users WHERE id = 10 FOR UPDATE;
-- 退化为：只锁 id=10 这条记录，不锁间隙
```

---

## 5. 死锁（Dead Lock）

### 5.1 死锁的四个必要条件

1. **互斥**：资源只能被一个事务占用
2. **占有并等待**：事务持有锁，同时等待其他锁
3. **不抢占**：已分配的锁不能被强制夺取
4. **循环等待**：事务之间形成循环锁等待

### 5.2 典型死锁场景

**场景 1：双向顺序依赖**

```sql
-- 事务 A
UPDATE orders SET status = 'paid' WHERE id = 1;  -- 锁住 id=1
UPDATE orders SET status = 'paid' WHERE id = 2;  -- 等待 id=2

-- 事务 B（并发执行）
UPDATE orders SET status = 'paid' WHERE id = 2;  -- 锁住 id=2
UPDATE orders SET status = 'paid' WHERE id = 1;  -- 等待 id=1 → 死锁！
```

**场景 2：不同事务对同一批资源的不同顺序加锁**

```go
// Go 代码：转账场景
func Transfer(db *sql.DB, from, to int64, amount float64) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // ❌ 危险：先锁 from 再锁 to，多个事务并发可能死锁
    _, err = tx.Exec("UPDATE accounts SET balance = balance - ? WHERE id = ?", amount, from)
    if err != nil {
        return err
    }
    _, err = tx.Exec("UPDATE accounts SET balance = balance + ? WHERE id = ?", amount, to)
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

**解决方案：按 ID 大小顺序加锁**

```go
func TransferFixed(db *sql.DB, from, to int64, amount float64) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // ✅ 按 ID 大小顺序加锁，避免循环等待
    first, second := from, to
    if from > to {
        first, second = to, from
    }

    _, err = tx.Exec("UPDATE accounts SET balance = balance - ? WHERE id = ?", amount, first)
    if err != nil {
        return err
    }
    _, err = tx.Exec("UPDATE accounts SET balance = balance + ? WHERE id = ?", amount, second)
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

### 5.3 死锁检测与处理

InnoDB 有**等待图**（Wait-For Graph）算法自动检测死锁，默认 50ms 检测一次。

```sql
-- 查看死锁日志（最近一次死锁的详细信息）
SHOW ENGINE INNOODB STATUS;
```

典型死锁日志：

```
LATEST DETECTED DEADLOCK
------------------------
2024-06-01 10:23:45 0x7f8a12345678
Transaction A: 
  INSERT INTO orders VALUES (1, 'A')  -- 锁住 X lock on (1)
Transaction B:
  INSERT INTO orders VALUES (2, 'B')  -- 锁住 X lock on (2)
Transaction A: 
  INSERT INTO orders VALUES (2, 'A')  -- 等待 B 的 X lock
Transaction B:
  INSERT INTO orders VALUES (1, 'B')  -- 等待 A 的 X lock → DEADLOCK
*** WE ROLL BACK TRANSACTION (B)  -- InnoDB 选择回滚 undo log 较少的事务
```

### 5.4 生产环境死锁预防策略

| 策略 | 说明 |
|------|------|
| 按固定顺序访问资源 | 全局约定加锁顺序（如 always 小 ID → 大 ID） |
| 减少锁持有时间 | 业务逻辑尽量放在锁外，SQL 化繁为简 |
| 降低隔离级别 | 读已提交（RC）可以减少 Gap Lock |
| 合理使用索引 | 避免无索引升级为表锁，减少锁范围 |
| 加锁超时 | `innodb_lock_wait_timeout = 5`（秒），超时自动放弃 |
| 监控告警 | `Table_locks_immediate`/`Table_locks_waited` 监控 |

---

## 6. Go + MySQL 锁的最佳实践

### 6.1 使用 `FOR UPDATE` 的正确姿势

```go
func GetAndUpdateOrder(db *sql.DB, orderID int64) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer func() {
        if err != nil {
            tx.Rollback()
        }
    }()

    var order Order
    // FOR UPDATE 加行锁，防止并发更新
    err = tx.QueryRowContext(ctx,
        "SELECT id, status, amount FROM orders WHERE id = ? FOR UPDATE", orderID).
        Scan(&order.ID, &order.Status, &order.Amount)
    if err != nil {
        return err
    }

    if order.Status == "paid" {
        return errors.New("order already paid")
    }

    _, err = tx.Exec("UPDATE orders SET status = 'paid', paid_at = NOW() WHERE id = ?", orderID)
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

### 6.2 乐观锁（无锁方案）

适合读多写少场景，避免数据库锁竞争：

```go
// 用 version 字段实现乐观锁
func UpdateWithOptimisticLock(db *sql.DB, id int64, newName string) error {
    for retries := 3; retries > 0; retries-- {
        var currentVersion int
        var name string
        err := db.QueryRow("SELECT version, name FROM users WHERE id = ?", id).
            Scan(&currentVersion, &name)
        if err != nil {
            return err
        }

        // UPDATE 返回 affected_rows，0 表示版本冲突
        result, err := db.Exec(
            "UPDATE users SET name = ?, version = version + 1 WHERE id = ? AND version = ?",
            newName, id, currentVersion)
        if err != nil {
            return err
        }

        affected, _ := result.RowsAffected()
        if affected == 1 {
            return nil // 更新成功
        }
        // else: 版本冲突，重试
    }
    return errors.New("optimistic lock failed after 3 retries")
}
```

---

## 7. 面试高频追问

**Q：InnoDB 和 MyISAM 的锁区别？**
> InnoDB 支持行锁+间隙锁，MyISAM 只有表锁。InnoDB 的锁是索引记录锁，MyISAM 是整表锁，并发性能差距巨大。

**Q：RR 级别下如何解决幻读？**
> Next-Key Lock（Record Lock + Gap Lock）锁住查询条件覆盖的范围，其他事务无法在范围内插入新记录。

**Q：死锁后 InnoDB 如何处理？**
> 等待图检测到环后，回滚 undo log 较小（最近修改数据较少）的事务，释放锁资源，让另一个事务继续执行。

**Q：线上发现大量锁等待，怎么排查？**
> 1. `SHOW PROCESSLIST` 看哪些 SQL 在等待；2. `SHOW ENGINE INNODB STATUS` 看锁等待链；3. 检查是否有长事务或大批量范围删除；4. 确认 SQL 是否走了正确索引。

