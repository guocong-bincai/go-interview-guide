# MySQL 自增 ID（AUTO_INCREMENT）在高并发下的锁竞争与优化方案

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：AUTO_INCREMENT、innodb_autoinc_lock_mode、间隙锁、高并发写入、雪花ID替代

## 面试官考察意图

面试官通常从这条链提问：

1. "你们表的主键是啥类型？为什么选这个类型？" → 引出主键设计
2. "自增 ID 在高并发下会不会有性能问题？" → 引出 AUTO_INCREMENT 锁
3. "如果自增 ID 不够用了怎么办？分布式环境呢？" → 引出全局唯一 ID 方案
4. "你们现在主键怎么生成的？" → 看候选人是否有实战经验

**区分度**在于能否说清 `innodb_autoinc_lock_mode` 三种模式的差异，以及在高并发场景下如何权衡数据连续性和插入性能。

---

## 核心答案（30 秒版）

| 问题 | 一句话回答 |
|------|-----------|
| **自增 ID 高并发下有锁竞争吗？** | MySQL 5.1~5.1.27 使用"表级自增锁"，所有 INSERT 排队等锁；5.1.22+ 引入轻量级算法后大幅缓解，但极端并发仍有瓶颈 |
| **什么是 innodb_autoinc_lock_mode？** | InnoDB 控制 AUTO_INCREMENT 锁粒度的参数，有 0（传统）、1（轻量）、2（交错）三种模式 |
| **三种模式区别？** | 0：每批 INSERT 都拿表锁；1：只查当前最大值（不锁）；2：直接取 counter（最快，但不保证连续）|
| **什么场景必须用模式 0？** | 存储过程 + AUTO_INCREMENT 同时使用时（需保证顺序），或某些复制场景要求严格连续性 |
| **高并发下自增 ID 够用吗？** | 单表 BIGINT AUTO_INCREMENT 理论上限约 922 亿条，一般业务不会达到 |

---

## 深度展开

### 1. AUTO_INCREMENT 的底层实现

InnoDB 维护了一个内存计数器（auto-inc counter），每次批量插入时：

```
传统模式（innodb_autoinc_lock_mode = 0）：

INSERT INTO t VALUES(NULL)  —→  请求 AUTO_INC_LOCK
                                │
                     ┌──────────▼──────────┐
                     │      Auto Inc Lock   │  ← 表级互斥锁
                     │    (直到语句结束释放)  │
                     └──────────┬──────────┘
                                │
                        返回下一个自增值

问题：所有并发 INSERT 必须排队！每个事务都要等前一个事务完成才能获取自增值。
```

### 2. innodb_autoinc_lock_mode 三种模式

```ini
# MySQL 配置文件 / 运行时修改
SET GLOBAL innodb_autoinc_lock_mode = 1;
```

| 模式 | 名称 | 锁定行为 | 适用场景 | 连续性保证 |
|------|------|---------|---------|-----------|
| **0** | 传统模式 | 每条 INSERT/LOAD DATA 都持表级 AUTO_INC_LOCK | 兼容老版本、复制一致性要求 | ✅ 严格连续递增 |
| **1** | 连续模式（默认 ≥ 5.1.22） | 批量 INSERT 只需在查询 max_id 时加短暂共享锁 | 绝大多数 OLTP 场景 | ✅ 基本连续（批量除外）|
| **2** | 交错模式 | 完全不锁，直接用计数器值 | 不要求 ID 连续的纯写场景 | ❌ ID 可能跳跃 |

**模式 1 的核心机制：**

```
模式 1（连续模式）执行流程：

Step 1: 检查是否批量 INSERT（INSERT ... SELECT / LOAD DATA / INSERT IGNORE）
        ├─ NO（普通 INSERT）→ 走轻量路径（不锁）
        └─ YES（批量操作）→ 需要保证连续性

Step 2（普通 INSERT）：
        ① 读取 auto_inc_counter 变量（无锁，原子操作）
        ② counter++
        ③ 将 counter 赋值给该行的 AUTO_INCREMENT 列
        ④ 无需等待任何其他事务
        
Step 3（批量操作）：
        ① 申请 AUTO_INC_LOCK（共享锁 + 自增专用锁）
        ② 计算需要多少行（通过预扫描）
        ③ 依次分配连续 ID
        ④ 释放锁
```

**实际效果对比：**

```sql
-- 测试场景：每秒 10 万 INSERT，观察锁等待
-- 模式 0：平均每行等待时间 ~50μs（排队严重）
-- 模式 1：平均每行等待时间 ~5μs（提升约 10x）
-- 模式 2：平均每行等待时间 ~2μs（进一步提升，但 ID 不连续）
```

### 3. 高并发下自增 ID 的实际瓶颈分析

```
假设场景：单机 MySQL，每秒 10 万并发写入同一张表

瓶颈链路：
┌─────────────────────────────────────┐
│  网络层接收请求 → Parser → Optimizer │
│           ↓                          │
│  执行器：获取自增值 → 生成 Row       │
│           ↓                          │
│  Redo Log 写入 → Flush             │
└─────────────────────────────────────┘

锁竞争主要在「获取自增值」这一步。
当 innodb_autoinc_lock_mode = 1 时：
- 普通 INSERT：几乎无锁竞争（原子 counter 递增）
- 批量 INSERT：有短暂锁等待，但远小于模式 0
- 真正瓶颈通常是 Redo Log flush 而非自增锁
```

### 4. 自增 ID 何时需要考虑替换？

| 条件 | 说明 |
|------|------|
| **单表超过 1 亿行** | BIGINT AUTO_INCREMENT 上限约 922 亿，一般不会超，但分库分表后单表仍需谨慎 |
| **需要全局唯一 ID** | 分布式环境下，多个节点的自增 ID 会冲突，需用 Snowflake / UUID / DB 自增号段 |
| **ID 暴露给外部且担心枚举攻击** | 淘宝商品 ID 被恶意遍历的案例，应使用雪花 ID 或随机 ID |

```go
// Go 中使用雪花算法（Snowflake）生成全局唯一 ID
func Snowflake(nodeID int64) func() int64 {
    var sequence int64
    var lastTime int64
    mu := &sync.Mutex{}
    
    return func() int64 {
        mu.Lock()
        defer mu.Unlock()
        
        now := time.Now().UnixNano() / 1_000_000 // 毫秒
        
        if now == lastTime {
            sequence++
            if sequence > 4095 {
                // 同一毫秒内序列用完，等下一毫秒
                for now <= lastTime {
                    now = time.Now().UnixNano() / 1_000_000
                }
            }
        } else {
            sequence = 0
            lastTime = now
        }
        
        return (now << 22) | (int64(nodeID) << 12) | sequence
        // 41位时间 + 10位节点ID + 12位序列 = 63位
    }
}
```

---

## 高频追问

### Q1：什么时候 AUTO_INCREMENT 会出现跳跃？

常见原因：
1. **INSERT 失败回滚**（如违反 UNIQUE 约束）：该次分配的自增值不会被回收
2. **切换 innodb_autoinc_lock_mode = 2**：计数器策略改变导致跳跃
3. **批量 INSERT（INSERT ... SELECT）**：预分配一批连续 ID，如果最终插入行数 < 预分配数，中间会有空档
4. **事务回滚**：事务中的自增值不回退

**面试话术**：
> "自增 ID 出现跳跃是正常的数据库行为，不是 bug。生产上除非业务强依赖连续 ID（极少见），否则不应纠结连续性。如果需要全局唯一且不重复的 ID，应该用雪花算法或 UUID，而不是依赖自增。"

### Q2：分库分表后怎么解决自增 ID 冲突？

主流方案：

```sql
-- 方案一：DB 自增号段模式（推荐）
-- 每个 DB 实例独立自增，步长 = 实例总数
-- 例如 10 个实例，每个步长 10
-- Instance 0: 1, 11, 21, 31, ...
-- Instance 1: 2, 12, 22, 32, ...
-- Instance 9: 10, 20, 30, 40, ...

CREATE TABLE t (
    id BIGINT PRIMARY KEY,
    -- 其余字段
);

-- 通过自定义自增起始值和步长
ALTER TABLE t AUTO_INCREMENT = N;

-- 实际中通过 ShardingSphere 或 MyCat 自动管理
```

```
方案二：Snowflake（雪花算法）
├── 优点：本地生成，零网络开销，按时间有序
└── 缺点：时钟回拨问题，需要服务注册中心协调 nodeID

方案三：UUID（v4 随机型）
├── 优点：绝对全局唯一
└── 缺点：无序导致索引页分裂，插入性能差

方案四：Redis INCR
├── 优点：简单可靠
└── 缺点：依赖 Redis 可用性，存在单点风险
```

### Q3：innodb_autoinc_lock_mode = 2 有什么风险？

主要风险：**MySQL-Master 复制一致性**。

```
场景：MASTER 上执行存储过程中的 AUTO_INCREMENT 查询

Master (mode=2):
1. INSERT INTO t VALUES(NULL) → 取 counter=100
2. Binlog: INSERT INTO t VALUES(100)

Slave (mode=0, 老版本):
1. Binlog 回放 INSERT t VALUES(100)
2. Slave 的 self-inc counter = 101（因为 auto_increment_offset）
3. 再次 INSERT → 得到 101

结果：Master 和 Slave 的后续自增值不同步 → 数据不一致！

解决方案：
- Master 和 Slave 设置相同 innodb_autoinc_lock_mode
- 或者统一升级到支持 mode=2 的版本
- 如果使用了存储过程 + AUTO_INCREMENT，建议保持 mode=0 或 mode=1
```

---

## 延伸阅读

- [MySQL Reference - Using AUTO_INCREMENT](https://dev.mysql.com/doc/refman/8.0/en/innoDB-auto-increment.html)
- [Oracle Blog - How InnoDB Handles Auto-Increment Values](https://blogs.oracle.com/mySQLRelatability/post/how-innodb-handles-auto-increment-values)
- [MySQL Internals Manual - Innodb Lock Modes](https://dev.mysql.com/doc/internals/en/auto-increment.html)
