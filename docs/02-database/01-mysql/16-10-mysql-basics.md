[🏠 首页](../../../README.md) · [🗺️ 数据库](../../README.md) · [🐬 MySQL](../README.md)

---

# MySQL 基础高频题：面试中容易被忽略的细节

## 面试官考察意图

MySQL 基础题看似简单，但**区分度极高**。面试官出一道 `COUNT(*)` vs `COUNT(1)` vs `COUNT(col)` 的题，初级只答"差不多"，高级要能讲清楚**全表扫描 vs 索引扫描、NULL 的特殊处理、InnoDB vs MyISAM 差异**，并指出哪些写法在生产中是坑。

这类题目考察的不是"会不会"，而是**理解深度**。

---

## 核心答案（30 秒版）

| 问题 | 正确答案 |
|------|---------|
| **CHAR vs VARCHAR** | CHAR 定长、VARCHAR 变长；CHAR 适合定长（UUID、邮编），VARCHAR 适合变长；CHAR 会自动截断空格，VARCHAR 保留末尾空格 |
| **COUNT(*/1/col)** | `COUNT(*)` = 全表扫描，`COUNT(col)` = 只扫描非 NULL 行；InnoDB 中 `COUNT(*)` 经过优化不比 `COUNT(1)` 慢 |
| **NULL 的坑** | NULL 不属于任何值，`IS NULL` / `IS NOT NULL` 才能匹配；`NOT IN (子查询)` 含 NULL 时结果全为空 |
| **InnoDB vs MyISAM** | InnoDB 支持事务/行锁/外键，支持 MVCC；MyISAM 不支持事务，只支持表锁，读多写少场景性能好 |
| **VARCHAR(100) vs VARCHAR(255)** | 都最多存 100/255 个字符，但 VARCHAR(255) 在某些版本需要额外 1 字节存长度，索引长度限制也不同（767/3072 字节）|

---

## 深度展开

### 1. CHAR vs VARCHAR：99% 面试者答不全

```sql
-- CHAR(10)：无论存入多少字符，都占 10 字节，末尾补空格
-- VARCHAR(10)：只占实际字节数 + 1~2 字节（长度前缀）

CREATE TABLE t1 (
    code CHAR(10),    -- 固定 10 字节
    name VARCHAR(10)   -- 最多 10 字节 + 1 字节长度
);
```

**存储示例**：

| 存入值 | CHAR(10) 存储 | VARCHAR(10) 存储 |
|--------|-------------|----------------|
| 'abc' | 'abc' + 7空格（10字节）| 'abc'（4字节）|
| '中国' | '中国' + 6空格（10字节，UTF8下每汉字3字节）| '中国'（7字节）|

**CHAR 的坑**：

```sql
-- 坑1：CHAR 自动截断尾部空格
INSERT INTO t1 (code) VALUES ('abc  ');  -- 'abc'（空格被截断）
SELECT code = 'abc' FROM t1;  -- TRUE！

-- 坑2：CHAR 超过长度不报错，直接截断
INSERT INTO t1 (code) VALUES ('abcdefghijk');  -- 'abcdefghij'（静默截断）
```

**生产选型原则**：
- 手机号、身份证、邮编 → **CHAR**（固定长度，避免碎片化）
- 用户名、地址、描述 → **VARCHAR**（节省空间）
- `VARCHAR(255)` 是常见默认值，但不要无脑用 255（索引长度限制见下）

**VARCHAR 索引长度限制（MySQL 5.7/InnoDB）**：
```sql
-- 767 字节前缀索引限制（InnoDB 16KB 页）
-- utf8mb4 每个字符最多 4 字节，所以：
VARCHAR(191) -- 最大安全索引长度（191 * 4 = 764 < 767）
VARCHAR(255) -- 无法直接建索引（255 * 4 > 767），需要前缀索引
ALTER TABLE t ADD KEY (name(191));  -- 前缀索引，只索引前 191 字符
```

### 2. COUNT(*) vs COUNT(1) vs COUNT(col)

```sql
SELECT COUNT(*) FROM orders;       -- 全表扫描，统计行数
SELECT COUNT(1) FROM orders;       -- 全表扫描，对每行常量表达式计数
SELECT COUNT(user_id) FROM orders;  -- 只统计 user_id 非 NULL 的行
SELECT COUNT(DISTINCT user_id) FROM orders;  -- 去重计数
```

**InnoDB 内部实现差异**：

| 写法 | 实际行为 | 优化 |
|------|---------|------|
| `COUNT(*)` | 全表扫描，但 InnoDB 对 `*` 做了特殊处理 | **有优化**，不取值 |
| `COUNT(1)` | 全表扫描，对每行计算 `1` 的值 | 无优化（或同等优化）|
| `COUNT(col)` | 索引扫描（如果是二级索引），只统计非 NULL | 依赖是否有索引 |

```sql
-- 生产问题：COUNT(*) 慢
-- 原因：InnoDB 事务隔离级别下，每次 COUNT 需要访问所有记录
-- 解决方案：
-- 1. 异步计数：用 Redis/数据库维护计数器
-- 2. 近似计数：用 EXPLAIN 的 rows_examined
-- 3. 缓冲表：定时同步到小表
```

**MyISAM 的特殊行为**：
```sql
-- MyISAM：COUNT(*) 直接读元数据（O(1)）
-- InnoDB：必须扫描全表（O(n)）
-- 这是 MyISAM 唯一在读性能上优于 InnoDB 的场景
```

### 3. NULL 的五大生产坑

```sql
-- 坑1：NULL 不等于任何值，包括自己
SELECT * FROM users WHERE age = NULL;        -- 永远为空！
SELECT * FROM users WHERE age IS NULL;       -- 正确写法
SELECT * FROM users WHERE age IS NOT NULL;   -- 正确写法

-- 坑2：NOT IN 子查询含 NULL，结果全为空
SELECT * FROM users
WHERE id NOT IN (
    SELECT manager_id FROM departments WHERE manager_id IS NOT NULL
);  -- 必须排除 NULL，否则整个 NOT IN 返回空

-- 坑3：COUNT(col) 不统计 NULL
SELECT COUNT(age) FROM users;  -- 只统计 age IS NOT NULL 的行
SELECT COUNT(*) FROM users;    -- 统计所有行

-- 坑4：GROUP BY 分组，NULL 算作一个分组
SELECT age, COUNT(*) FROM users GROUP BY age;
-- 输出：20岁->3人, 30岁->5人, NULL->2人（NULL 单独一组）

-- 坑5：JOIN 时 NULL 不匹配
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id;  -- user_id 为 NULL 的订单匹配不到用户
```

**推荐实践：所有字段尽量 NOT NULL**

```sql
-- 原因：
-- 1. 索引更紧凑（无需 NULL bitmap）
-- 2. 避免 NULL 的各种陷阱
-- 3. 查询语义更清晰

ALTER TABLE orders ADD COLUMN remark VARCHAR(200) NOT NULL DEFAULT '';
```

### 4. InnoDB vs MyISAM 生产选型

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务 | ✅ 支持 | ❌ 不支持 |
| 行锁 | ✅ 支持（高并发写入）| ❌ 只支持表锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| MVCC | ✅ 支持（RC/RR）| ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复（redo log）| ❌ 需手动修复 |
| 全文索引 | ✅（5.6+）| ✅（原生支持）|
| COUNT(*) | O(n)（全表扫描）| O(1)（元数据）|
| 适用场景 | OLTP、写入密集型 | OLAP、读多写少、日志 |

**生产决策树**：

```
需要事务支持？
    → YES → 必须 InnoDB
    → NO → 写入量 > 1000/s？
        → YES → InnoDB（行锁并发好）
        → NO → 数据仓库/报表？
            → YES → MyISAM 或 ClickHouse
            → NO → 小型站点/缓存 → MyISAM 可选
```

### 5. VARCHAR 长度选择：不要无脑 255

```sql
-- 255 的问题：索引长度超限
-- utf8mb4：255 * 4 = 1020 字节 > 767（InnoDB 索引限制）

-- 合理选型示例：
email           VARCHAR(255)  -- 邮箱，最大长度合理
phone           CHAR(11)     -- 手机号，定长
province        CHAR(10)     -- 省/市，定长
article_title   VARCHAR(200) -- 文章标题，够用即可
user_nickname   VARCHAR(32)  -- 昵称，中文 10 字 + emoji
```

### 6. 存储引擎选择：如何修改默认引擎

```sql
-- MySQL 5.7 默认：InnoDB
-- 修改默认引擎（my.cnf）
[mysqld]
default-storage-engine = InnoDB

-- 已有表转换（注意：锁表！）
ALTER TABLE old_myisam ENGINE = InnoDB;
```

---

## 高频追问

### Q1：InnoDB 为什么推荐自增主键而不是 UUID？

```sql
-- 顺序主键：插入时追加到页尾，页分裂少，顺序 I/O
-- UUID：随机插入，导致页分裂、随机 I/O、索引碎片化

-- 实测（500 万条数据）：
自增主键 INSERT:  ~8500 QPS
UUID INSERT:      ~1200 QPS（慢 7 倍）
```

```go
// Go 生成有序 UUID：snowflake 或 ULID
// Snowflake: 时间戳 + 机器ID + 序列号 → 有序
// ULID: 时间戳 + 随机数 → 字典序有序，但不完全顺序
type ID string

func GenerateSnowflake() ID {
    // 使用 Sonyflake（Go 实现的 Snowflake）
    sf := sonyflake.NewSonyflake(sonyflake.Settings{})
    id, _ := sf.NextID()
    return strconv.FormatUint(id, 36)
}
```

### Q2：CHAR(10) 存 'abc' 实际占多少字节？

在 `utf8mb4` 编码下：
- 'a'、'b'、'c' 各占 1 字节（ASCII 范围）
- 不足 10 字节的部分用**空格**填充
- `'abc'` 实际存储为 `'abc       '`（7 个空格）
- **CHAR 始终占用声明的长度**，尾部空格在比较时被忽略

### Q3：生产环境为什么不要用 VARCHAR(255) 作为所有字段的默认值？

1. **索引长度问题**：`VARCHAR(255)` 在 `utf8mb4` 下无法建完整索引
2. **内存浪费**：InnoDB 内存页（16KB）中能容纳的行数更少
3. **语义不清**：255 字符 = 约 85 个中文 = 一段短文本，不适合所有场景

---

### Q4：MyISAM 还在生产用吗？

**基本不用了**。MySQL 8.0 直接**移除了 MyISAM**（`ALTER TABLE ... ENGINE=MyISAM` 在 8.0 中会报错）。

唯一保留价值：**崩溃后快速恢复**（`.MYI` 索引文件 + `.frm` 表定义文件结构简单）。但 InnoDB 的崩溃恢复已经非常可靠，MyISAM 的这个优势也消失了。

---

### Q5：utf8 vs utf8mb4 — 为什么现在一律用 utf8mb4？

MySQL 里 `utf8` 不是标准 UTF-8！它是一个 **最多只支持 3 字节** 的编码，根本存不了 Emoji。

```sql
-- MySQL 的 "utf8" 实际是 utf8mb3
SHOW CHARACTER SET LIKE 'utf%';
+---------+---------------+--------------------+--------+
| Charset | Description   | Max Len | Default |
+---------+---------------+----------+--------+
| utf8    | UTF-8 Unicode | 3        | utf8_general_ci |
| utf8mb4 | UTF-8MB4      | 4        | utf8mb4_0900_ai_ci |
+---------+---------------+----------+--------+
```

| 对比项 | utf8 (utf8mb3) | utf8mb4 |
|--------|-------------|---------|
| 最大字符长度 | 3 字节 | **4 字节** |
| Emoji 支持 | ❌ 存不进去 | ✅ 正常存储 |
| 索引最大长度 | 767 字节 | 3072 字节（4.0+）|
| 生产推荐度 | ❌ 已废弃警告 | ✅ 标准 |

**Emoji 存入失败报错示例：**
```sql
INSERT INTO users (nickname) VALUES ('🐕');
-- ERROR 1366: Incorrect string value: '\xF0\x9F\x90\x95'
-- 因为 \xF0\x9F\x90\x95 是 4 字节，utf8 只支持 3 字节
```

**正确配置（my.cnf + Go DSN）：**
```ini
# my.cnf
[mysqld]
default-character-set = utf8mb4
collation-server = utf8mb4_0900_ai_ci

[client]
default-character-set = utf8mb4
```

```go
// Go 连接字符串中指定 charset
dsn := "user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
```

---

### Q6：COLLATION 是什么？utf8mb4_general_ci vs utf8mb4_unicode_ci vs utf8mb4_0900_ai_ci 怎么选？

COLLATION（排序规则）定义了字符集的**比较和排序规则**。相同字符集可以有多种 Collation。

```sql
-- 查看 utf8mb4 的所有排序规则
SHOW COLLATION WHERE Charset = 'utf8mb4' LIMIT 10;
```

三大常见 Collation 对比：

| Collation | 版本 | 对齐方式 | 排序准确性 | 性能 |
|-----------|------|---------|-----------|------|
| `utf8mb4_general_ci` | 老版本 | 简单逐字符比较 | ❌ 有偏差（如 ß ≠ ss）| ⚡ 最快 |
| `utf8mb4_unicode_ci` | 4.1+ | Unicode 标准对齐 | ✅ 准确 | 🐢 中等 |
| `utf8mb4_0900_ai_ci` | 8.0+ | ICU Unicode 9.0 对齐 | ✅✅ 最准 | 🐢 稍慢 |

**核心区别示例：**
```sql
-- 德语：ß 在 general_ci 下等于 s，但在 unicode_ci 下不等于
SELECT 'ß' = 'ss' COLLATE utf8mb4_general_ci;   -- 1（错误匹配）
SELECT 'ß' = 'ss' COLLATE utf8mb4_unicode_ci;    -- 0（正确）

-- 中文拼音排序：general_ci 完全靠不住
SELECT '张' > '李' COLLATE utf8mb4_general_ci;   -- 无意义（按码位比）
SELECT '张' > '李' COLLATE utf8mb4_zh_0900_as_cs; -- 按拼音比
```

**⚠️ Collation 不一致是线上最常见隐性问题！**

```sql
-- 场景：两表 JOIN 时 Collation 不同导致全表扫描
CREATE TABLE t1 (name VARCHAR(50)) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE TABLE t2 (name VARCHAR(50)) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 这两个表的 name 列 COLLATION 不同！
-- 做 JOIN 时无法使用索引 → Extra: Using where; Using join buffer
SELECT * FROM t1 JOIN t2 ON t1.name = t2.name;
```

**解决原则**：建库建表时统一指定 `COLLATE`。

**面试建议**：生产环境用 `utf8mb4_0900_ai_ci`（MySQL 8.0 默认），如需区分大小写再加 `_cs` 后缀。---

### Q7：事务回滚到底是怎么做到的？UNDO LOG 的作用

面试官问完隔离级别和 MVCC 之后，经常追问一个深层问题：**「你说 RR 隔离级别能看到快照数据，那 UNDO LOG 具体是怎么配合 MVCC 实现快照读的？」**

```sql
START TRANSACTION;
UPDATE orders SET status = 'paid' WHERE id = 1;  -- version 2
ROLLBACK;  -- 回到 version 1 的状态
```

**UNDO LOG 是 InnoDB 保证原子性的关键——没有它，事务就只是"看起来能提交"。**

核心机制：

```
执行 UPDATE 之前，先把旧值写入 UNDO LOG：

原始数据（page）：id=1, status='pending', trx_id=100, version=1

1. BEGIN → trx_id=100
2. UPDATE status='paid'：
   ├─ 将旧值 {status='pending'} 写入 UNDO LOG
   ├─ 修改页内数据为 {status='paid'}, trx_id=100, version=2
   └─ TRX_ID=100 记录此事务

3. COMMIT → 标记 tx_commit=100，UNDO LOG 标记为可清理
4. 或 ROLLBACK → 读取 UNDO LOG 中的旧值覆盖当前数据
```

**关键概念**：

| 概念 | 说明 |
|------|------|
| **Undo Log** | 逆操作日志：记录数据被修改前的值，用于回滚 |
| **Read View** | MVCC 的核心：创建事务时的可见性判断依据 |
| **Hidden Columns** | 每行数据有两个隐藏字段：`trx_id`(最后修改事务ID)、`roll_pointer`(指向 undo log 的指针) |

```sql
-- 演示 MVCC + Undo Log 协同工作
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;

-- 快照读：此时读到的是 version=1 的数据（通过 undo log 链回溯）
SELECT * FROM orders WHERE id = 1;  -- 返回 pending

UPDATE orders SET status = 'paid' WHERE id = 1;  -- 生成 v2，undo 记录 v1

-- 同一事务内再次查询，依然看到 v1（RR 隔离级别的快照一致性）
SELECT * FROM orders WHERE id = 1;  -- 仍然返回 pending！

COMMIT;  -- 不可见回滚了
```

**面试官想要的深度回答**：
1. UNDO Log 有两种：Insert Undo（仅 CREATE 时产生，提交后可立即删）和 Update Undo（UPDATE/DELETE 产生，需等到无事务引用后才清理）
2. Undo Log 存在 Undo Space（逻辑上的段空间），不是物理磁盘文件，可以通过 `innodb_undo_directory` / `innodb_undo_tablespaces` 配置
3. ROLLBACK 的本质：沿着 undo chain 从最新版本往回遍历，把每个版本的旧值重新写回数据页
4. 大批量 ROLLBACK 会很慢：因为是逐条反向执行 undo log 里的操作

---

### Q8：如何强制 MySQL 使用某个索引？OPTIMIZER HINT 实战

优化器有时会选错索引，这时候需要人为干预。MySQL 8.0 引入了 **Optimizer Hint**，可以精确告诉优化器「请用这个索引」。

```sql
-- 传统方式：FORCE INDEX（有效但粗糙）
SELECT * FROM orders FORCE INDEX(idx_user_id)
WHERE user_id = 100 AND status = 'active';

-- MySQL 8.0+ 推荐：INDEX HINT（更精准）
SELECT * FROM orders USE INDEX FOR SCAN (idx_user_id)
WHERE user_id = 100 AND status = 'active';
```

三种 Hint 用法：

```sql
-- 1. 强制用某索引
SELECT /*+ INDEX(t idx_user_id) */ * FROM orders t 
WHERE user_id = 100;

-- 2. 忽略某索引
SELECT /*+ IGNORE_INDEX(t idx_status) */ * FROM orders t 
WHERE status = 'active';

-- 3. 设置优化目标
SELECT /*+ OPT_PARAM('max_execution_time','1000') */ *
FROM orders WHERE created_at > '2024-01-01';
```

**但更好的做法不是加 Hint，而是排查优化器为什么选错**：

```sql
-- 用 OPTIMIZER_TRACE 看优化器的决策过程
SET optimizer_trace="enabled=on";
SELECT * FROM orders WHERE user_id = 100 AND status = 'active';
SELECT * FROM information_schema.OPTIMIZER_TRACE;
SET optimizer_trace="enabled=off";
```

典型 trace 输出展示优化器考虑了多少个候选索引、每个索引的成本评估，以及最终为何选择 A 而不是 B。

**生产最佳实践**：
1. 优先用 `ANALYZE TABLE` 更新统计信息（optimizer 靠它评估成本）
2. 用 OPTIMIZER_TRACE 定位优化器误判原因
3. Hint 作为最后手段，不建议日常使用
4. 联合索引顺序很重要：等值查询在前，范围查询在后（左前缀原则）

---

## 延伸阅读

- [MySQL 官方文档：InnoDB vs MyISAM](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)
- [MySQL VARCHAR 索引长度限制](https://dev.mysql.com/doc/refman/8.0/en/innodb-limits.html)
- [COUNT(*) vs COUNT(1) 性能测试](https://www.percona.com/blog/count-vs-count1/)
- [NULL 的五个坑（官方文档）](https://dev.mysql.com/doc/refman/8.0/en/working-with-null.html)
- [MySQL Character Sets Documentation](https://dev.mysql.com/doc/refman/8.0/en/charset-unicode.html)
- [MySQL Collations Overview](https://dev.mysql.com/doc/refman/8.0/en/charset-mysql.html)
- [InnoDB Rollback Mechanism](https://dev.mysql.com/doc/refman/8.0/en/rollback.html)
- [Optimizer Trace Feature](https://dev.mysql.com/doc/refman/8.0/en/optimizer-tracing.html)
- [MySQL Optimizer Hints](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html)
