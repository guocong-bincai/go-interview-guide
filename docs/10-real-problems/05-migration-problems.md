# 数据迁移与重构问题

> 考察频率：★★★★☆  优先级：P1

---

## 1. 面试官考察意图

考察候选人有没有真正做过生产环境的数据迁移或架构重构。这类问题没有标准答案，考察的是：1）是否理解迁移的风险（数据一致性、停机时间）；2）有没有系统性的迁移方法论；3）能否设计出可以灰度回滚的方案。

---

## 2. 核心答案（30秒版）

数据迁移的核心是**不停服 + 可回滚**。推荐方案是**双写双读**：新旧系统同时写入，逐步切读流量。全量同步用 binlog 增量追赶保证数据一致。迁移期间每次变更都要能回退，回滚方案要和迁移方案同等重视。

---

## 3. 深度展开

### 3.1 不停服数据库迁移

#### 方案：双写 + 灰度切读

```
迁移四阶段
═══════════════════════════════════════════════════════════

阶段1：双写（写入新旧两份数据）
┌─────────┐    写    ┌───────┐    写    ┌───────┐
│  App    │ ──────▶ │ 老DB  │         │ 新DB  │
└─────────┘         └───────┘         └───────┘
       │  同时写入新老库
       │  老库服务读
       │
阶段2：全量+增量同步
       │  binlog 同步老库→新库（增量追赶）
       │
阶段3：灰度切读
       │  5% → 10% → 30% → 50% → 100% 逐步切到新库
       │  对比新老库返回数据是否一致（影子流量）
       │
阶段4：下掉老库
       │  新库100%读，确认N天无问题后删除老库写入路径

═══════════════════════════════════════════════════════════
```

#### Go 实现：双写模式

```go
type DualWriter struct {
   老DB *sql.DB
    新DB *sql.DB
}

func (w *DualWriter) Insert(ctx context.Context, table string, data map[string]interface{}) error {
    var errs []error

    // 同时写两个库
    if err := w.insertInto(ctx, w.老DB, table, data); err != nil {
        errs = append(errs, fmt.Errorf("old db: %w", err))
    }
    if err := w.insertInto(ctx, w.新DB, table, data); err != nil {
        errs = append(errs, fmt.Errorf("new db: %w", err))
    }

    // 如果新库失败，必须返回错误（保证新库数据完整）
    if len(errs) > 1 {
        return fmt.Errorf("dual write failed: %v", errs)
    }
    return nil
}

func (w *DualWriter) insertInto(ctx context.Context, db *sql.DB, table string, data map[string]interface{}) error {
    keys := make([]string, 0, len(data))
    vals := make([]interface{}, 0, len(data))
    placeholders := make([]string, 0, len(data))

    for k, v := range data {
        keys = append(keys, k)
        vals = append(vals, v)
        placeholders = append(placeholders, "?")
    }

    query := fmt.Sprintf("INSERT INTO %s (%s) VALUES (%s)",
        table, strings.Join(keys, ","), strings.Join(placeholders, ","))
    _, err := db.ExecContext(ctx, query, vals...)
    return err
}
```

#### binlog 增量同步

```go
// 用 canal 监听 MySQL binlog，实现增量数据同步
// 简化示例

func startBinlogSync() {
    // 1. 启动 canal 监听 binlog
    client := NewCanalClient("127.0.0.1:3306", "canal", "canal")

    client.Subscribe("schema-change-TABLE", func(entry *BinlogEntry) {
        // 2. 解析 binlog event
        switch entry.EventType {
        case INSERT, UPDATE, DELETE:
            // 3. 将变更同步到新库
            syncToNewDB(entry)
        }
    })
}

func syncToNewDB(entry *BinlogEntry) error {
    // 按表名路由，根据操作类型构建 SQL
    switch entry.EventType {
    case INSERT:
        return execInsert(entry)
    case UPDATE:
        return execUpdate(entry)
    case DELETE:
        return execDelete(entry)
    }
    return nil
}
```

#### 切流验证

```go
// 影子流量对比：新老库返回数据一致性对比
func validateShadowTraffic(ctx context.Context, req *Request) error {
    // 读老库
    oldResult, err := oldDB.QueryContext(ctx, req.SQL)
    if err != nil {
        return err
    }
    defer oldResult.Close()

    // 读新库
    newResult, err := newDB.QueryContext(ctx, req.SQL)
    if err != nil {
        return err
    }
    defer newResult.Close()

    // 对比结果
    if !compareResults(oldResult, newResult) {
        return fmt.Errorf("result mismatch, rolling back traffic")
    }
    return nil
}

// 按百分比切流量
func getTrafficRouter(percentage int) bool {
    // 0~99 随机数，小于 percentage 就走新库
    return rand.Intn(100) < percentage
}
```

#### 回滚方案

```go
// 随时可回滚的关键：保留老库写入路径
func handleRequest(w http.ResponseWriter, r *http.Request) {
    trafficPercentage := getConfig("new_db_percentage") // 配置文件，随时可改

    if getTrafficRouter(trafficPercentage) {
        // 走新库
        serveFromNewDB(w, r)
    } else {
        // 走老库（随时可切回）
        serveFromOldDB(w, r)
    }
}

// 紧急回滚：把配置改成0即可
// new_db_percentage: 0  → 100%走老库
// new_db_percentage: 5 → 只留5%在新库
```

---

### 3.2 单体服务拆分成微服务

#### 识别拆分时机

| 信号 | 说明 | 是否该拆 |
|------|------|---------|
| 团队 > 15人，同一代码库冲突频繁 | 协作成本高 | 是 |
| 部署周期长，一个模块出问题影响全部 | 稳定性差 | 是 |
| 技术栈需求差异大（Go + Java混合）| | 是 |
| 业务边界清晰（用户/订单/支付天然分离）| | 是 |
| 系统规模小（QPS < 100）| 拆分成本大于收益 | 否 |

#### 绞杀者模式（Strangler Pattern）

```
┌──────────────────────────────────────────────────────┐
│              绞杀者模式：渐进式替换                    │
├──────────────────────────────────────────────────────┤
│                                                      │
│  旧单体应用         新微服务                          │
│  ┌──────────┐     ┌──────────┐                      │
│  │ Module A │     │Service A │  ← 新服务            │
│  │ Module B │     │Service B │  ← 逐个迁移           │
│  │ Module C │     │Service C │                      │
│  └────┬─────┘     └────┬─────┘                      │
│       │                │                            │
│       ▼                │                            │
│  ┌─────────────────┐   │                            │
│  │  API Gateway    │◀──┘                            │
│  │  (路由到新老服务)│   路由规则：                    │
│  └─────────────────┘   - A,B,C 全部迁移完成后，      │
│                          删除旧模块                   │
└──────────────────────────────────────────────────────┘
```

#### 共享数据库的过渡方案

```go
// 问题：Service A 和 Service B 都要访问 User 数据
// 方案1：每个服务独立库（最终态）
// 方案2：迁移期间共享库（过渡态）

// 过渡态：共享 DB 访问约定
// 1. 服务只操作自己表前缀的数据
// 2. 禁止跨服务直接 JOIN
// 3. 通过 API 而非 DB 访问其他服务的数据

// User表属于 user-service，order-service 需要用户信息：
// ❌ 禁止：SELECT * FROM users WHERE id=?  (直接查别人的表)
// ✅ 正确：GET http://user-service/users/{id}  (通过 API)
```

#### 接口兼容

```go
// 迁移过程中，老客户端还在调老接口
// 新服务要兼容老接口，不能破坏向后兼容性

// 老接口
type OldUserResponse struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// 新接口（字段更多）
type NewUserResponse struct {
    ID        int    `json:"id"`
    Name      string `json:"name"`
    Email     string `json:"email"`
    CreatedAt string `json:"created_at"`
    Phone     string `json:"phone"`
}

// 兼容方案：新服务同时支持新旧响应格式
func userHandler(w http.ResponseWriter, r *http.Request) {
    version := r.Header.Get("API-Version")
    user := getUser()

    if version == "v1" {
        // 老客户端：只返回老字段
        resp := OldUserResponse{ID: user.ID, Name: user.Name, Email: user.Email}
        json.NewEncoder(w).Encode(resp)
    } else {
        // 新客户端：返回全部字段
        json.NewEncoder(w).Encode(user)
    }
}
```

---

### 3.3 MySQL 迁移到分库分表

#### 分片键选择原则

```go
// ✅ 好的分片键
shardKey := userID // 按用户ID分片
// 理由：用户维度的查询最多（查自己的订单、余额）
// 热点问题：某些大V用户的数据会集中在某个分片 → 可以用虚拟分片

// ❌ 坏的分片键
shardKey := orderID // 按订单ID分片
// 理由：用户查自己所有订单 → 跨分片查询，查所有分片才能凑齐
```

#### 路由策略

```go
// 按 userID mod N 分片
func getShard(userID int64, shardCount int) int {
    return int(userID % int64(shardCount))
}

// 但 mod N 的问题：N变了（扩容），大量数据需要迁移
// ✅ 改进：一致性哈希
// 或：范围分片（按时间或ID范围）

type ShardRouter struct {
    shards []*sql.DB // 每个分片一个连接
}

func (r *ShardRouter) Query(userID int64, sql string) (*sql.Row, error) {
    shardIdx := r.getShardIdx(userID)
    return r.shards[shardIdx].QueryRow(sql, userID)
}
```

#### 跨分片查询

```go
// 问题：查询"所有用户的订单总量"（跨N个分片）
// 解决方案：ES 作为搜索和聚合的兜底

// 1. 所有分片数据实时同步到 Elasticsearch
// 2. 跨分片聚合查询走 ES
// 3. Go 代码：
result, err := es.Search(`
    {
      "query": { "match_all": {} },
      "aggs": {
        "total_orders": { "sum": { "field": "order_count" } }
      }
    }
`)
```

#### 全量迁移 + 双写验证

```go
// 迁移流程
// 1. 初始化：从老库全量导出到分片新库（离线跑）
// 2. 开启双写：新写入同时写老库和分片库
// 3. 增量同步：binlog 把老库的增量数据同步到新库
// 4. 数据对比：用 checksum 对比每个分片的数据是否一致
// 5. 切读：逐步把读流量切到新库
// 6. 停写老库：确认新库稳定后，关闭老库写入

// 验证工具
func verifyDataChecksum(shardDBs []*sql.DB, table string) error {
    for i, db := range shardDBs {
        oldSum := getTableChecksum(oldDB, table)
        newSum := getTableChecksum(db, table)
        if oldSum != newSum {
            return fmt.Errorf("shard %d checksum mismatch: old=%s new=%s", i, oldSum, newSum)
        }
    }
    return nil
}
```

---

## 4. 高频追问

### Q：迁移过程中数据不一致了怎么办？

> 立即停止迁移切换，回退到纯老库。排查不一致的原因：1）双写漏了（新库写入失败没回滚老库）；2）增量同步延迟（binlog 追不上）；3）应用层缓存和 DB 不一致。修复后用数据对比工具（如 pt-table-checksum）重新验证一致性，再继续迁移。

### Q：如何做到不停服迁移？

> 核心是**渐进式切换**：双写阶段新老库同时写，增量同步保证数据追平，灰度切读保证新库正确。每次切换都是可逆的，遇到问题立即回退。

### Q：什么情况下不应该分库分表？

> 如果单表数据量在 5000 万以下，优先优化索引、读写分离，而不是分库分表。分库分表引入的复杂度（跨分片查询、分片键选择、扩容）非常高，只有在索引和读写分离无法解决时才使用。

---

## 5. 延伸阅读

- [Strangler Fig Pattern - Martin Fowler](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [ShardingSphere 官方文档](https://shardingsphere.apache.org/)
- [MySQL 双写重建方案 - 美团技术博客](https://tech.meituan.com/2022/01/06/mysql-double-write-solution.html)
- [数据库迁移最佳实践 - Shopify](https://shopify.engineering/database-migrations-at-shopify)
