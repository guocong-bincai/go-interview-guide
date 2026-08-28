# 深度分页优化：MySQL OFFSET 瓶颈与替代方案

> 考察频率：★★★★☆  优先级：P1  电商/内容平台必考
> 关键词：Deep Pagination、游标分页、Keyset、覆盖索引、Limit Offset 优化

---

## 核心答案（30 秒版）

| 方案 | 原理 | 适用场景 | P99 延迟 |
|------|------|----------|----------|
| **传统 OFFSET** | `LIMIT n OFFSET m` | 前几页快速响应 | ⭐⭐⭐ 快 |
| **游标分页 (Cursor)** | WHERE id > last_id LIMIT n | 无限滚动/Feed流 | ⭐⭐⭐⭐⭐ 稳定 |
| **延迟关联** | JOIN 子查询先定位 ID | 需要大字段但不查全部行 | ⭐⭐⭐⭐ 良好 |
| **倒排索引** | ES/Lucene 做深翻 | 搜索场景，百万级以上 | ⭐⭐⭐⭐⭐ |

**生产最佳实践**：前端展示类用游标分页；列表查询用延迟关联。

---

## 深度展开

### 1. OFFSET 的性能陷阱

```sql
-- 第 1 页：毫秒级
SELECT * FROM articles ORDER BY created_at DESC LIMIT 20 OFFSET 0;

-- 第 10000 页：扫描并丢弃 200,000 行！
SELECT * FROM articles ORDER BY created_at DESC LIMIT 20 OFFSET 200000;

-- 第 100000 页：严重超时 ⚠️
SELECT * FROM articles ORDER BY created_at DESC LIMIT 20 OFFSET 2000000;
```

**根因分析**：

```
MySQL 执行计划：
1. 按 ORDER BY 排序（假设已有 created_at 索引）
2. 跳过前 200000 行（OFFSET 开销）
3. 取接下来 20 行
4. SELECT * 需要回表获取完整行数据 ← 这是真正的性能杀手！

如果每行 1KB，回表 200,000 次 × 1KB = 200MB 内存 I/O！
```

### 2. 游标分页（Cursor-based / Keyset Pagination）—— 面试重点

```sql
-- ❌ 传统方式（慢）
SELECT * FROM articles ORDER BY created_at DESC LIMIT 20 OFFSET 200000;

-- ✅ 游标方式（快）
-- 上一页最后一条记录：created_at = '2026-01-15 10:30:00', id = 98765
SELECT * FROM articles 
WHERE (created_at < '2026-01-15 10:30:00' OR 
       (created_at = '2026-01-15 10:30:00' AND id < 98765))
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

**核心思想**：不用 OFFSET 跳行，而是用 WHERE 条件定位起始位置。

```go
// Go 中实现游标分页
type CursorParams struct {
    Limit int     // 每页数量
    After string  // 上一页最后一条的 cursor（序列化后的 created_at+id）
}

func paginate(cursor CursorParams, userID int64) ([]*Article, error) {
    query := "SELECT id, title, summary, created_at FROM articles " +
        "WHERE user_id = ? AND status = 'published'"
    
    if cursor.After != "" {
        // 解析 cursor: "timestamp:id"
        parts := strings.SplitN(cursor.After, ":", 2)
        timestamp, _ := time.Parse(time.RFC3339, parts[0])
        articleID, _ := strconv.ParseInt(parts[1], 10, 64)
        
        query += fmt.Sprintf(" AND (created_at < %s OR (created_at = %s AND id < %d))",
            sqlTimestamp(timestamp), sqlTimestamp(timestamp), articleID)
    }
    
    query += " ORDER BY created_at DESC, id DESC LIMIT ?"
    
    rows, err := db.Query(query, userID, cursor.Limit)
    // ... 处理结果
    return articles, nil
}

// 返回下一页 cursor
func makeCursor(article *Article) string {
    return fmt.Sprintf("%s:%d", article.CreatedAt.Format(time.RFC3339), article.ID)
}
```

**游标分页的优势**：

```
1. 无论翻到第几页，时间复杂度都是 O(log N)，不受 OFFSET 影响
2. 不重复、不漏数据 —— 因为基于唯一键组合排序
3. 可以配合二级排序消除并列值导致的重复

缺点：
1. 不能跳到指定页码（不知道总页数）
2. 删除已显示的数据时，后面数据会上移 → "页面跳跃"现象
   （抖音/Twitter 的解决方案是加入时间戳精度和随机因子）
```

### 3. 延迟关联（Deferred Join）—— 经典面试解法

当 SELECT * 需要读取大量列但大部分时候不需要全量数据时：

```sql
-- ❌ 慢：先回表拿所有数据，然后筛选
SELECT id, title, content, author, created_at, updated_at, tags, thumbnail
FROM articles
WHERE user_id = 123 AND status = 'published'
ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
-- 回表 100,020 次 ✗

-- ✅ 快：子查询只定位主键 ID，外层 JOIN 拿需要的列
SELECT a.id, a.title, a.summary, a.created_at
FROM articles a
INNER JOIN (
    SELECT id FROM articles
    WHERE user_id = 123 AND status = 'published'
    ORDER BY created_at DESC LIMIT 20 OFFSET 100000
) AS tmp ON a.id = tmp.id;
-- 内层查询走 cover index (user_id, created_at, id)，不回表！
-- 外层 JOIN 只需要 20 次回表 ✓✓✓
```

**Go 工程化封装**：

```go
type ArticleListResult struct {
    Articles []*Article
    HasMore  bool
    NextCursor string
}

func (s *ArticleService) ListArticles(ctx context.Context, req *ListRequest) (*ArticleListResult, error) {
    baseQuery := "SELECT id, title, summary, created_at FROM articles WHERE user_id = ? AND status = 'published'"
    params := []interface{}{req.UserID}

    if req.After != "" {
        parts := strings.Split(req.After, ":")
        t, _ := time.Parse(time.RFC3339, parts[0])
        aid, _ := strconv.ParseInt(parts[1], 10, 64)
        baseQuery += fmt.Sprintf(" AND (created_at < ? OR (created_at = ? AND id < ?))")
        params = append(params, t, t, aid)
    }

    // 限制查询上限防止恶意请求
    limit := req.Limit
    if limit == 0 || limit > 50 {
        limit = 20
    }

    var articles []*Article
    err := s.db.QueryRowContext(ctx, baseQuery+" LIMIT ?", params...).Scan(
        /* scan into structs */).Error
    
    if err == nil && len(articles) > 0 {
        result.HasMore = true
        result.NextCursor = makeCursor(articles[len(articles)-1])
    }

    return result, nil
}
```

### 4. 各方案选型决策树

```
                    开始
                     │
              ┌──────┴──────┐
              │             │
         需要页码？      不需要页码
         (翻页 UI)?      (无限滚动?)
              │             │
         ┌────┴────┐       ↓
         │YES│NO    │  游标分页 (Cursor-based)
         │    │     │    ↑ 最简单最稳定
         ↓    ↓     │
      数据量小?  数据量大?   │
      (<1万行)? (>1万行)?   │
         │     │          │
      OFFSET   延迟关联     │
      直接上    或          │
      即可     ES分页       │
              │           │
              ↓           │
           ES / MySQL    │
           倒排索引       │
           (百万级以上)   │
```

### 5. 面试话术

**Q：你们的产品是怎么做分页的？**

> 我们的 Feed 流用的是游标分页。用户看到每条动态有个 created_at 和 id，这两个值序列化后作为 cursor 传给后端。下一批请求带上这个 cursor，我们就能精确地从这个位置之后继续拉取，不管翻了多少页性能都一样。对于管理后台那种需要跳转指定页的场景，我们用延迟关联来加速——子查询先通过覆盖索引拿到 ID 列表，外层再 join 获取详情。

**Q：为什么游标分页比 OFFSET 好？**

> OFFSET 的本质是「扫描并丢弃」——要取出第 10000 页的 20 条数据，MySQL 必须先扫描并丢弃前面 200,000 行，每行都要回表读完整数据。游标分页则是「精准定位」——WHERE 条件直接匹配上一页最后一行的 key，通过索引找到新起点，时间复杂度从 O(N) 降到了 O(log N)。实际测试中，第 10000 页用 OFFSET 可能需要 2~3 秒，游标分页只需 5ms。
