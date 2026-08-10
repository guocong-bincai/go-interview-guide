[🏠 首页](../../../README.md) · [📦 数据库](../README.md) · [🐬 MySQL](./README.md)

---

# MySQL 8.0 新特性与高频面试题

> 考察频率：★★★★☆  优先级：P1  
> 关键词：窗口函数、CTE、SKIP LOCKED、NOWAIT、Instant DDL、JSON_TABLE、递归查询

---

## 面试官考察意图

这道题考察候选人对 **MySQL 版本演进**的关注程度，以及能否将新特性与实际业务场景结合。
5 年以上的工程师不仅要会用新特性，还要理解**为什么 MySQL 8.0 要做这些改进**，以及在迁云场景下（阿里云RDS、腾讯云CDB）这些特性的实际表现。

常见追问：
- 窗口函数能解决什么问题？和子查询相比优势在哪？
- 递归 CTE 的典型场景是什么？
- SKIP LOCKED 和 NOWAIT 在 Go 并发队列中怎么用？

---

## 核心答案（30 秒版）

MySQL 8.0 四大核心改进：

| 分类 | 特性 | 业务价值 |
|------|------|---------|
| **SQL 能力** | 窗口函数、CTE、递归查询 | 告别复杂自连接，SQL 可读性大幅提升 |
| **并发控制** | SKIP LOCKED、NOWAIT | 无锁跳过已锁定行，高并发队列场景必备 |
| **DDL 效率** | Instant ADD COLUMN | 秒级加列，不用等 DDL 锁表 |
| **JSON 处理** | JSON_TABLE、JSON_VALUE | 把 JSON 当表来查，告别大量 JSON_FUNCTION |

---

## 深度展开

### 1. 窗口函数（Window Functions）

#### 1.1 出现背景

MySQL 8.0 之前，排名、分组聚合等操作需要**自连接或子查询**，SQL 冗长且性能差：

```sql
-- 老写法：求每个部门的工资排名（子查询）
SELECT d.name, e.salary,
  (SELECT COUNT(*) FROM employees e2
   WHERE e2.dept_id = e.dept_id AND e2.salary > e.salary) + 1 AS rank
FROM employees e
JOIN departments d ON e.dept_id = d.id;
```

#### 1.2 窗口函数写法

```sql
-- 窗口函数：一行代码搞定，MySQL 自动优化
SELECT d.name, e.salary,
  ROW_NUMBER() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rank,
  RANK()       OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rank_with_gap,
  DENSE_RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS dense_rank,
  SUM(e.salary) OVER (PARTITION BY e.dept_id) AS dept_total,
  AVG(e.salary) OVER (PARTITION BY e.dept_id ORDER BY e.salary ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg
FROM employees e
JOIN departments d ON e.dept_id = d.id;
```

**三类窗口函数的区别：**

| 函数 | 重复值处理 | 示例输出 |
|------|-----------|---------|
| `ROW_NUMBER()` | 不重复，总是从 1 开始 | 1, 2, 3, 4 |
| `RANK()` | 有重复，跳过后续排名 | 1, 2, 2, 4（并列后跳到 4） |
| `DENSE_RANK()` | 有重复，不跳过排名 | 1, 2, 2, 3 |

#### 1.3 滑动窗口（Sliding Window）

```sql
-- 计算每行及其前后各 1 行的滚动平均值
SELECT date, sales,
  AVG(sales) OVER (
    ORDER BY date 
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
  ) AS moving_avg_3
FROM daily_sales;

-- 累计求和（从开始到当前行）
SELECT date, sales,
  SUM(sales) OVER (ORDER BY date) AS cumulative_sales
FROM daily_sales;
```

#### 1.4 典型业务场景

**场景一：排行榜（Leaderboard）**

```sql
-- 实时排行榜：每分钟更新一次
SELECT 
  user_id,
  score,
  ROW_NUMBER() OVER (ORDER BY score DESC) AS world_rank,
  ROW_NUMBER() OVER (PARTITION BY region ORDER BY score DESC) AS region_rank
FROM rankings;
```

**场景二：计算环比增长率**

```sql
-- 每月销售额及环比增长率
SELECT 
  month,
  revenue,
  LAG(revenue) OVER (ORDER BY month) AS prev_month,
  ROUND((revenue - LAG(revenue) OVER (ORDER BY month)) / LAG(revenue) OVER (ORDER BY month) * 100, 2) AS growth_rate
FROM monthly_revenue;
```

**场景三：去重留最大（替代 DISTINCT ON）**

```sql
-- MySQL 没有 DISTINCT ON，用窗口函数实现
-- 找出每个部门薪资最高的员工
SELECT * FROM (
  SELECT e.*, ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
  FROM employees e
) t WHERE rn = 1;
```

---

### 2. CTE（公用表表达式）

#### 2.1 WITH 子句：简化复杂查询

```sql
-- 找出销售额超过平均值的月份
WITH monthly_stats AS (
  SELECT 
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(amount) AS total
  FROM orders
  GROUP BY DATE_FORMAT(order_date, '%Y-%m')
),
avg_total AS (
  SELECT AVG(total) AS avg_sales FROM monthly_stats
)
SELECT ms.month, ms.total
FROM monthly_stats ms, avg_total a
WHERE ms.total > a.avg_sales
ORDER BY ms.total DESC;
```

**对比嵌套子查询：**

```sql
-- 嵌套写法：阅读性差，嵌套层数有限制
SELECT * FROM (
  SELECT DATE_FORMAT(order_date, '%Y-%m') AS month, SUM(amount) AS total
  FROM orders GROUP BY month
) t1
WHERE total > (
  SELECT AVG(total) FROM (
    SELECT SUM(amount) AS total FROM orders GROUP BY month
  ) t2
);
```

#### 2.2 递归 CTE：树形结构查询

**典型场景：组织架构、分类树、商品类目**

```sql
-- 递归查询：找出所有下级部门（包括子部门的子部门）
WITH RECURSIVE dept_tree AS (
  -- 锚点：顶级部门（没有父部门的）
  SELECT id, name, parent_id, 0 AS level
  FROM departments
  WHERE parent_id IS NULL
  
  UNION ALL
  
  -- 递归：找子部门
  SELECT d.id, d.name, d.parent_id, dt.level + 1
  FROM departments d
  INNER JOIN dept_tree dt ON d.parent_id = dt.id
)
SELECT * FROM dept_tree ORDER BY level, name;
```

**Go 应用场景：解析电商多级类目树**

```go
// Go 中将递归 CTE 结果映射到树结构
type DeptNode struct {
    ID       int
    Name     string
    ParentID *int
    Level    int
    Children []*DeptNode
}

// 递归构建树（将扁平结果转为树）
func BuildTree(nodes []*DeptNode) []*DeptNode {
    nodeMap := make(map[int]*DeptNode)
    for _, n := range nodes {
        nodeMap[n.ID] = &DeptNode{ID: n.ID, Name: n.Name, ParentID: n.ParentID, Level: n.Level}
    }
    var roots []*DeptNode
    for _, n := range nodes {
        if n.ParentID == nil {
            roots = append(roots, nodeMap[n.ID])
        } else {
            if parent, ok := nodeMap[*n.ParentID]; ok {
                parent.Children = append(parent.Children, nodeMap[n.ID])
            }
        }
    }
    return roots
}
```

**递归 CTE 执行过程（MySQL 内部）：**

```
Step 1: 锚点查询 → 取出根节点（parent_id IS NULL）
Step 2: 用根节点找子节点 → 加入结果集
Step 3: 用子节点继续找子子节点 → 循环直到没有新行
Step 4: UNION ALL 合并所有层级
```

**递归终止条件：**
- 没有匹配的行（WHERE 条件过滤完）
- 达到 `cte_max_recursion_depth` 限制（默认 1000，可通过 SQL hint 修改）
- 防止死循环：`UNION DISTINCT` 自动去重

---

### 3. SKIP LOCKED 与 NOWAIT：无锁并发控制

#### 3.1 出现背景

在 **Go 高并发队列消费** 场景中，传统做法是：

```go
// 传统方式：SELECT FOR UPDATE 阻塞等待
for {
    row := db.QueryRow("SELECT * FROM jobs WHERE status='pending' ORDER BY id LIMIT 1 FOR UPDATE")
    // 问题：所有 worker 都在等同一把锁，只有1个能抢到，其他都白等
}
```

MySQL 8.0 提供两个新语法：**跳过已锁定的行**，让 worker 自己找下一个可处理的行。

#### 3.2 SKIP LOCKED：跳过已锁定行

```sql
-- 尝试加锁，如果行已被锁定，跳过它，继续找下一行
SELECT * FROM jobs 
WHERE status = 'pending' 
ORDER BY id 
LIMIT 1 
FOR UPDATE SKIP LOCKED;
```

**Go 并发队列实现（Worker Pool 模式）：**

```go
func worker(ctx context.Context, id int, db *sql.DB, jobs chan<- Job) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
        }

        var job Job
        // SKIP LOCKED：其他 worker 在处理时，这边不会阻塞等待
        err := db.QueryRowContext(ctx,
            `SELECT id, payload FROM jobs 
             WHERE status='pending' 
             ORDER BY id LIMIT 1 
             FOR UPDATE SKIP LOCKED`).Scan(&job.ID, &job.Payload)

        if err == sql.ErrNoRows {
            time.Sleep(100 * time.Millisecond) // 没有任务，短暂休眠
            continue
        }
        if err != nil {
            log.Printf("worker %d query error: %v", id, err)
            continue
        }

        // 处理任务
        process(job)
        
        // 更新状态
        db.ExecContext(ctx, "UPDATE jobs SET status='done', worker_id=? WHERE id=?", id, job.ID)
    }
}

func startWorkerPool(ctx context.Context, db *sql.DB, workerCount int) {
    jobs := make(chan Job, 100)
    var wg sync.WaitGroup
    for i := 0; i < workerCount; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            worker(ctx, id, db, jobs)
        }(i)
    }
    // ... 等待 worker 结束
}
```

**SKIP LOCKED 的优势：**
- 多个 worker 完全并发，不会互相阻塞
- 任务分配自动均衡（谁先拿到谁处理）
- 适合"任务量大、worker 数固定"的秒杀/导入场景

#### 3.3 NOWAIT：立即报错

```sql
-- 尝试加锁，如果行已被锁定，直接报错（不等待）
SELECT * FROM jobs WHERE id = 123 FOR UPDATE NOWAIT;

-- ERROR 3572: Lock wait timeout exceeded; try restarting transaction
```

**使用场景：**

```go
// 希望立刻知道"有人在处理"，不做任何等待
func tryClaimJob(db *sql.DB, jobID int64) (bool, error) {
    result, err := db.Exec(
        "SELECT * FROM jobs WHERE id = ? FOR UPDATE NOWAIT", jobID)
    
    if err != nil {
        if strings.Contains(err.Error(), "Lock wait timeout") {
            return false, nil // 已被认领
        }
        return false, err
    }
    // 认领成功
    return true, nil
}
```

**SKIP LOCKED vs NOWAIT 对比：**

| 特性 | SKIP LOCKED | NOWAIT |
|------|------------|--------|
| 遇到锁 | 跳过，找下一行 | 直接报错 |
| 适用场景 | 并发任务队列 | 抢单/认领 |
| 代码逻辑 | 循环直到拿到或没任务 | 单次尝试，失败即放弃 |

---

### 4. Instant ADD COLUMN（秒级加列）

#### 4.1 背景

传统 `ALTER TABLE ... ADD COLUMN` 需要重建表（ALGORITHM=COPY）：
- 数据量大的表可能需要几小时
- 期间表被锁住，无法写入
- 主从复制延迟飙升

#### 4.2 MySQL 8.0.12+ Instant 算法

```sql
-- 末尾添加带 DEFAULT 的列，秒级完成
ALTER TABLE orders ADD COLUMN remark VARCHAR(255) DEFAULT '' NOT NULL, ALGORITHM=INSTANT;

-- 查看是否使用 INSTANT
SHOW ALTER TABLE PROCEDURES WHERE Field like '%algorithm%';
```

**限制：**
- 只能添加在末尾（不能插入中间列）
- 不能与其他 DDL 操作一起执行（如同时加索引）
- 不支持含全文索引或 spatial 索引的表

#### 4.3 Go + 灰度发布场景

```go
// Go 应用：在不同时刻发布不同版本，通过 remark 字段实现灰度
func addColumnIfNotExists(db *sql.DB) error {
    // 先检查列是否存在
    var count int
    err := db.QueryRow(`
        SELECT COUNT(*) FROM INFORMATION_SCHEMA.COLUMNS 
        WHERE TABLE_SCHEMA = DATABASE() 
        AND TABLE_NAME = 'orders' 
        AND COLUMN_NAME = 'remark'
    `).Scan(&count)
    
    if count > 0 {
        return nil // 列已存在，跳过
    }
    
    // 添加列（INSTANT 算法，不锁表）
    _, err = db.Exec("ALTER TABLE orders ADD COLUMN remark VARCHAR(255) DEFAULT '' NOT NULL, ALGORITHM=INSTANT")
    return err
}
```

---

### 5. JSON 处理增强

#### 5.1 JSON_TABLE：把 JSON 当表查

```sql
-- 从 JSON 数组中提取数据，模拟行列结构
SELECT jt.*
FROM orders,
JSON_TABLE(
    items,  -- JSON 字段名
    '$[*]'  -- 路径：JSON 数组的所有元素
    COLUMNS (
        product_id VARCHAR(50) PATH '$.product_id',
        quantity   INT         PATH '$.quantity',
        price      DECIMAL(10,2) PATH '$.price'
    )
) AS jt;

-- 对 JSON 数组做聚合：计算订单总金额
SELECT 
    o.id,
    o.total,
    SUM(jt.price * jt.quantity) AS json_total
FROM orders o,
JSON_TABLE(
    items,
    '$[*]'
    COLUMNS (
        price INT PATH '$.price',
        quantity INT PATH '$.quantity'
    )
) AS jt
GROUP BY o.id;
```

**Go + JSON_TABLE 场景：**

```go
// 订单系统：items 字段存储 JSON 数组，反查商品信息
rows, err := db.QueryContext(ctx, `
    SELECT o.id, o.total, jt.*
    FROM orders o,
    JSON_TABLE(
        items,
        '$[*]'
        COLUMNS (
            product_id VARCHAR(50) PATH '$.product_id',
            qty INT PATH '$.quantity'
        )
    ) AS jt
    WHERE o.id = ?
`, orderID)
```

#### 5.2 JSON_VALUE：提取标量值

```sql
-- 提取 JSON 中指定路径的值（返回标量，而非 JSON 对象）
SELECT 
    o.id,
    JSON_VALUE(meta, '$.source') AS source,
    JSON_VALUE(meta, '$.campaign_id') AS campaign
FROM orders o;

-- 在 WHERE 子句中使用
SELECT * FROM orders 
WHERE JSON_VALUE(meta, '$.source') = 'APP';
```

---

## 高频追问

**Q：窗口函数和子查询的性能对比？**

> MySQL 8.0 对窗口函数做了专门优化，使用**流式聚合**（streaming aggregation），不需要先完整计算子查询再外层过滤。实测中，窗口函数在 TOP-N 查询（如"每个部门薪资前3名"）上比自连接快 3-5 倍。

**Q：递归 CTE 会死循环吗？怎么防护？**

> 有可能。MySQL 通过 `cte_max_recursion_depth`（默认 1000）限制递归深度。可以通过 SQL hint 调整：`SELECT ... OPTION (max_recursion_depth = 10000)`。设计时要确保递归有终止条件（如 parent_id = NULL 或 level < 10）。

**Q：Go 项目中如何判断 MySQL 版本特性是否可用？**

```go
// 运行时检测 MySQL 版本
var version string
db.QueryRow("SELECT VERSION()").Scan(&version)
// version like "8.0.x"

// 条件性使用特性
if strings.HasPrefix(version, "8.0") {
    // 使用窗口函数、SKIP LOCKED 等
} else {
    // 降级到子查询、传统锁等方式
}
```

**Q：SKIP LOCKED 在分布式任务分发中相比 Redis 队列有什么优势？**

> MySQL 任务队列的优势：**事务保障 + 持久化**。任务一旦被"认领"（SELECT FOR UPDATE），即使 worker 崩溃，任务也不会丢失（状态未更新，锁释放后其他 worker 可以重新认领）。Redis 队列的 LPOP 是"拿走"语义，崩溃会导致任务丢失，需要额外的 ACK 机制（如 BRPOPLPUSH）。

---

## 延伸阅读

- [MySQL 8.0 Window Functions 官方文档](https://dev.mysql.com/doc/refman/8.0/en/window-functions.html)
- [MySQL 8.0 CTE 官方文档](https://dev.mysql.com/doc/refman/8.0/en/with.html)
- [MySQL 8.0 JSON_TABLE](https://dev.mysql.com/doc/refman/8.0/en/json-table-functions.html)
- [SKIP LOCKED and NOWAIT官方文档](https://dev.mysql.com/doc/refman/8.0/en/select.html#idm140506368496)
- [Go-MySQL-Driver 并发连接最佳实践](https://github.com/go-sql-driver/mysql#connection-pool)