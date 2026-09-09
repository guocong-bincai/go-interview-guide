# MySQL 乐观锁与悲观锁：CAS 实现、ABA 问题与 Go 实战选型

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：乐观锁 CAS、版本号 version、retry 机制、悲观锁 FOR UPDATE、并发冲突

## 面试官考察意图

面试官通常从这条链提问：

1. "你们更新库存/余额时怎么保证并发安全？" → 引出并发控制
2. "MySQL 锁有悲观和乐观之分，什么场景用哪种？" → 看架构思考深度
3. "乐观锁失败了怎么办？重试吗？" → 看实战经验
4. "乐观锁会有 ABA 问题吗？" → 区分度题

**区分度**在于能否说清两种方案的适用场景（读多写少 vs 写多读少）、CAS 重试策略的合理上限，以及误判率分析。

---

## 核心答案（30 秒版）

| 问题 | 一句话回答 |
|------|-----------|
| **乐观锁是什么？** | 不加排他锁，靠版本号（version）或时间戳做 CAS 比较来判断数据未被修改 |
| **悲观锁是什么？** | `SELECT ... FOR UPDATE` 加排他行锁，拿到锁后才能执行更新 |
| **什么时候选乐观锁？** | 读多写少、冲突率低、短事务（如秒杀扣减前的资格检查） |
| **什么时候选悲观锁？** | 写多读少、高冲突场景（如转账操作、库存强一致性扣减） |
| **乐观锁重试多少次合适？** | 一般 3~5 次，配合指数退避；超过阈值返回错误由业务层处理 |

---

## 深度展开

### 1. 乐观锁实现（Version 字段 + CAS）

```sql
-- 表结构
CREATE TABLE product (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    stock INT DEFAULT 0,
    version INT DEFAULT 0  -- 乐观锁版本号
);

-- 更新操作（原子 CAS）
UPDATE product 
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = ?;  -- 传入查询时获取的旧版本号

-- 检查 affected rows
-- affected_rows = 1 → 更新成功
-- affected_rows = 0 → 版本冲突，需要重试
```

**Go 完整实现：**

```go
type ProductDAO struct {
    db *sql.DB
}

const maxRetries = 3

func (d *ProductDAO) DeductStock(productID int64, amount int) error {
    for attempt := 0; attempt < maxRetries; attempt++ {
        // Step 1: 先读取当前状态（包括版本号）
        var stock, version int
        err := d.db.QueryRowContext(context.Background(),
            "SELECT stock, version FROM product WHERE id = ?", productID).Scan(&stock, &version)
        if err != nil {
            return err
        }
        
        // Step 2: 业务校验
        if stock < amount {
            return fmt.Errorf("insufficient stock: have %d, need %d", stock, amount)
        }
        
        // Step 3: CAS 尝试更新
        result, err := d.db.ExecContext(context.Background(),
            `UPDATE product SET stock = stock - ?, version = version + 1 
             WHERE id = ? AND version = ?`,
            amount, productID, version)
        if err != nil {
            return err
        }
        
        // Step 4: 检查是否命中（affected rows = 0 表示版本号不匹配）
        affected, _ := result.RowsAffected()
        if affected == 1 {
            return nil  // 成功
        }
        
        // retry：短暂 sleep 后重试（指数退避）
        if attempt < maxRetries-1 {
            time.Sleep(time.Duration(attempt+1) * 100 * time.Millisecond)
            continue
        }
        
        return fmt.Errorf("concurrent modification, exhausted retries")
    }
    return nil
}
```

### 2. 悲观锁实现（FOR UPDATE）

```sql
START TRANSACTION;

-- 加排他锁并更新
SELECT stock FROM product WHERE id = 1 FOR UPDATE;
-- ← 此时其他事务的 FOR UPDATE 会被阻塞

UPDATE product SET stock = stock - 1 WHERE id = 1;

COMMIT;  -- 释放锁
```

**Go 实现：**

```go
func (d *ProductDAO) DeductStockWithLock(productID int64, amount int) error {
    tx, err := d.db.BeginTx(context.Background(), &sql.TxOptions{Isolation: sql.LevelRepeatableRead})
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    var stock int
    err = tx.QueryRowContext(context.Background(),
        "SELECT stock FROM product WHERE id = ? FOR UPDATE", productID).Scan(&stock)
    if err != nil {
        return err
    }
    
    if stock < amount {
        return fmt.Errorf("insufficient stock: %d", stock)
    }
    
    _, err = tx.ExecContext(context.Background(),
        "UPDATE product SET stock = stock - 1 WHERE id = ?", productID)
    if err != nil {
        return err
    }
    
    return tx.Commit()
}
```

### 3. 两种方案对比决策表

| 维度 | 乐观锁（Version CAS） | 悲观锁（FOR UPDATE） |
|------|---------------------|---------------------|
| **适用场景** | 读多写少，冲突率低（<10%） | 写多读少，冲突率高（>30%） |
| **CPU 开销** | 低（无锁等待，重试在应用层） | 中（数据库内部维护锁） |
| **内存开销** | 低（不需要额外存储锁信息） | 中（需要维护锁表和锁等待队列） |
| **死锁风险** | ❌ 几乎不会 | ✅ 有（多个资源交叉持有时） |
| **吞吐量** | 高（大部分请求直接通过） | 中（被锁的请求必须排队） |
| **延迟分布** | P50 快，P99 可能慢（重试时） | 整体稳定，但极端情况会排队 |
| **实现复杂度** | 简单 | 中等（需处理死锁检测） |

### 4. ABA 问题讨论

```
ABA 问题在乐观锁中的表现：

T1: 读到 value = 100, version = 1
T2: CAS 更新 value = 100 → 101, version = 1 → 2
T3: CAS 更新 value = 101 → 100, version = 2 → 3
T1: 带 version=1 做 CAS → 失败（因为现在是 version=3），正确拦截

结论：引入 version 计数器后，ABA 问题自然避免。
因为即使值回到原来的数，version 已经不同了。
```

**注意**：ABA 真正危险的地方是**无版本号的纯值比较**，而实际工程中几乎没人这么做。Redis 的 CAS 操作也有类似问题（因此 Redis 用 Lua 脚本保证原子性来规避）。

---

## 高频追问

### Q1：乐观锁重试次数不设上限会有什么问题？

会导致 **活锁（livelock）**：两个事务互相发现对方在改，谁也不肯放，无限重试。

```go
// ❌ 错误示范：永远重试
for {
    // CAS 逻辑
    if failed { continue }
}

// ✅ 正确做法：设上限 + 退避
for i := 0; i < maxRetries; i++ {
    time.Sleep(backoff(i))
    // CAS...
}
```

### Q2：悲观锁会引发死锁，怎么解决？

```sql
-- MySQL 自带死锁检测（innodb_deadlock_detect = ON 默认开启）
-- 检测到死锁后会回滚其中一个事务

-- 主动预防策略：
1. 所有事务按相同顺序获取锁（如始终先锁表 A 再锁表 B）
2. 使用 LOCK WAIT 超时（innodb_lock_wait_timeout，默认 50s）
3. 在应用层使用 Try Lock 模式（SELECT ... FOR UPDATE NOWAIT）
```

### Q3：秒杀场景下选乐观锁还是悲观锁？

**秒杀场景推荐乐观锁。** 原因：
- 秒杀是读远多于写的场景（大量用户读库存 + 少量人下单）
- 乐观锁对未争抢的情况零性能损耗
- 真正抢到的人 CAS 才会失败重试（只有极小概率冲突）
- 如果库存紧张到频繁冲突，说明超卖风险大——应该结合分布式锁做最终确认

---

## 延伸阅读

- [MySQL Internals Manual - Optimistic Locking](https://dev.mysql.com/doc/refman/8.0/en/optimistic-locking.html)
- [Percona Blog - InnoDB Lock Modes](https://www.percona.com/blog/innodb-lock-modes/)
