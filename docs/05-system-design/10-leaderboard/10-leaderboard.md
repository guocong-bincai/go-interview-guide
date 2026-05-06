# 排行榜系统设计（实时 + 历史）

> 考察频率：★★★★☆  优先级：P0
> 关键词：Redis ZSet、跳表、分桶、ZREVRANK、好友排行榜

## 为什么用 Redis ZSet 做排行榜

| 方案 | 优点 | 缺点 |
|------|------|------|
| MySQL `ORDER BY score DESC` | 简单 | QPS 低，数据量大时慢 |
| Redis ZSet | O(log N) 插入/查询，天然支持排名 | 内存占用 |
| 外部排序（MapReduce）| 亿级数据 | 延迟高，不适合实时 |

```
Redis ZSet 核心命令：
ZADD leaderboard score user_id   → 添加/更新用户分数，O(log N)
ZREVRANK leaderboard user_id    → 获取用户排名（从高到低），O(log N)
ZREVRANGE leaderboard 0 99      → 获取 Top 100，O(log N + M)
ZSCORE leaderboard user_id      → 获取用户分数，O(1)
ZINCRBY leaderboard 10 user_id  → 增量加分，O(log N)
```

---

## 1. 基础方案：Redis ZSet

### Go 实现

```go
import "github.com/redis/go-redis/v9"

type LeaderboardService struct {
    redis *redis.Client
}

const leaderboardKey = "leaderboard:global"

// AddScore 添加/更新用户分数（分数变化时自动更新）
func (s *LeaderboardService) AddScore(ctx context.Context, userID string, score float64) error {
    return s.redis.ZAdd(ctx, leaderboardKey, redis.Z{
        Score:  score,
        Member: userID,
    }).Err()
}

// IncrScore 增量加分（点赞、答题等场景）
func (s *LeaderboardService) IncrScore(ctx context.Context, userID string, delta float64) error {
    return s.redis.ZIncrBy(ctx, leaderboardKey, delta, userID).Err()
}

// GetRank 获取用户排名（从 0 开始，第 1 名 rank=0）
func (s *LeaderboardService) GetRank(ctx context.Context, userID string) (int64, error) {
    // ZREVRANK：从高到低排序，返回 rank
    rank, err := s.redis.ZRevRank(ctx, leaderboardKey, userID).Result()
    if err == redis.Nil {
        return -1, nil // 用户不存在
    }
    return rank, err
}

// GetTopN 获取 Top N 排行榜
func (s *LeaderboardService) GetTopN(ctx context.Context, n int64) ([]TopUser, error) {
    results, err := s.redis.ZRevRangeWithScores(ctx, leaderboardKey, 0, n-1).Result()
    if err != nil {
        return nil, err
    }
    users := make([]TopUser, 0, len(results))
    for i, z := range results {
        users = append(users, TopUser{
            Rank:   int64(i) + 1,
            UserID: z.Member.(string),
            Score:  int64(z.Score),
        })
    }
    return users, nil
}

// GetScore 获取用户分数
func (s *LeaderboardService) GetScore(ctx context.Context, userID string) (float64, error) {
    return s.redis.ZScore(ctx, leaderboardKey, userID).Result()
}
```

### 内存占用估算

```
Redis ZSet 每条记录存储：
- member: string（用户 ID，平均 10 字节）
- score: 8 字节（float64）
- 跳表节点：约 32 字节
- hash 表指针：约 8 字节

每条 ≈ 64 字节

100 万用户：100万 × 64 = 64MB
1 亿用户：  1亿 × 64 = 6.4GB

建议：单个 ZSet < 1000 万条，否则考虑分桶
```

---

## 2. 超大榜单：分桶 ZSet

### 什么时候需要分桶

```
单 ZSet 问题：
- 超过 1000 万条时，ZREVRANK 延迟上升到 10ms+
- 内存连续分配压力增大

分桶策略：
- 方案 1：按分数段分桶（0-1000 一个桶，1000-2000 一个桶...）
- 方案 2：按用户 ID 哈希分桶（多副本并行）
- 推荐：方案 1，因为排行榜查询通常有明确的分数范围
```

### 分桶实现

```go
const (
    bucketSize   = 10000   // 每个桶最多 1 万人
    bucketScoreStep = 100  // 每桶覆盖 100 分
)

func getBucketKey(score float64) string {
    bucket := int(score) / bucketScoreStep
    return fmt.Sprintf("leaderboard:bucket:%d", bucket)
}

// 添加分数（自动路由到对应桶）
func (s *LeaderboardService) AddScoreBatched(ctx context.Context, userID string, score float64) error {
    bucketKey := getBucketKey(score)
    return s.redis.ZAdd(ctx, bucketKey, redis.Z{Score: score, Member: userID}).Err()
}

// 获取用户全局排名（需跨桶累加）
func (s *LeaderboardService) GetGlobalRank(ctx context.Context, userID string) (int64, error) {
    // 1. 先找到用户所在的桶（扫描所有桶，找到用户）
    // 实际实现：从最高分桶开始向下扫描
    var globalRank int64 = 0
    for i := 100; i >= 0; i-- {
        bucketKey := fmt.Sprintf("leaderboard:bucket:%d", i)
        rank, err := s.redis.ZRevRank(ctx, bucketKey, userID).Result()
        if err == redis.Nil {
            continue // 用户不在这个桶，继续下一个
        }
        if err != nil {
            return 0, err
        }
        // 全局排名 = 高分桶总人数 + 桶内排名
        globalRank += rank + 1
    }
    return globalRank, nil
}
```

---

## 3. 历史排行榜（按天/周/月）

```go
// 时间维度桶
func getTimeBasedKey(prefix string, t time.Time) string {
    return fmt.Sprintf("%s:%s", prefix, t.Format("2006-01-02"))
}

// 周榜：每周一重置
func getWeeklyKey(t time.Time) string {
    year, week := t.ISOWeek()
    return fmt.Sprintf("leaderboard:weekly:%d-%02d", year, week)
}

// 获取上周 Top 100
func (s *LeaderboardService) GetLastWeekTop(ctx context.Context) ([]TopUser, error) {
    lastWeek := time.Now().AddDate(0, 0, -7)
    key := getWeeklyKey(lastWeek)
    return s.GetTopN(ctx, key, 100)
}

// 自动过期：月榜保留 3 个月，周榜保留 3 个月
func (s *LeaderboardService) SetExpire(ctx context.Context, key string) error {
    return s.redis.Expire(ctx, key, 90*24*time.Hour).Err()
}
```

### 定时归档策略

```go
// crontab：每天凌晨生成昨日排行榜快照
func archiveYesterday() {
    yesterday := time.Now().AddDate(0, 0, -1)
    key := getTimeBasedKey("leaderboard:daily", yesterday)

    // 1. 获取 Top 1000（快照，固定数据量）
    results, _ := s.redis.ZRevRangeWithScores(ctx, key, 0, 999).Result()

    // 2. 存入 MySQL 持久化
    for i, z := range results {
        db.Exec(`INSERT INTO leaderboard_archive (date, rank, user_id, score)
            VALUES (?, ?, ?, ?)`,
            yesterday.Format("2006-01-02"), i+1, z.Member.(string), int64(z.Score))
    }

    // 3. 设置过期（保留 90 天）
    s.redis.Expire(key, 90*24*time.Hour)
}
```

---

## 4. 分布式多节点写入

### 本地聚合 + 批量提交

```
问题：多个服务实例同时写 Redis，每次加分都调用 ZINCRBY → 网络开销大

解决：本地 counter + 定时批量同步

用户点赞 → 本地内存 +1 → 每 1s 批量 ZINCRBY → Redis
```

```go
type LocalBatchWriter struct {
    redis       *redis.Client
    localScores map[string]float64  // 本地聚合
    mu          sync.Mutex
    ticker      *time.Ticker
}

func (w *LocalBatchWriter) Start() {
    w.ticker = time.NewTicker(1 * time.Second)
    go func() {
        for range w.ticker.C {
            w.flush()
        }
    }()
}

func (w *LocalBatchWriter) Incr(userID string, delta float64) {
    w.mu.Lock()
    w.localScores[userID] += delta
    w.mu.Unlock()
}

func (w *LocalBatchWriter) flush() {
    w.mu.Lock()
    defer w.mu.Unlock()
    if len(w.localScores) == 0 {
        return
    }

    pipe := w.redis.Pipeline()
    for userID, score := range w.localScores {
        pipe.ZIncrBy(ctx, leaderboardKey, score, userID)
    }
    pipe.Exec(ctx)
    w.localScores = make(map[string]float64)
}
```

---

## 5. 好友排行榜（Group By Friends）

```go
// 好友排行榜：用户只关心自己好友中的排名
// 核心：用 ZINTERSTORE 求交集（自己的好友集合 ∩ 全局排行榜）

func (s *LeaderboardService) GetFriendLeaderboard(ctx context.Context, userID string, friendIDs []string) ([]TopUser, error) {
    friendSetKey := "friends:" + userID
    tempKey := "temp:friend_rank:" + userID

    // 1. 先把好友 ID 存入临时 ZSet（分数先设为 0）
    pipe := s.redis.Pipeline()
    for _, fid := range friendIDs {
        pipe.ZAdd(ctx, tempKey, redis.Z{Score: 0, Member: fid})
    }
    pipe.Exec(ctx)

    // 2. ZINTERSTORE：把自己的好友 ZSet 和全局排行榜求交集
    // 结果存入 tempKey，权重用全局排行榜的分数
    s.redis.ZInterStore(ctx, tempKey, &redis.ZStore{
        Keys:    []string{tempKey, leaderboardKey},
        Weights: []float64{0, 1},  // 第二个权重=1，取全局排行榜的分数
    })

    // 3. 获取好友排行榜
    results, _ := s.redis.ZRevRangeWithScores(ctx, tempKey, 0, -1).Result()

    // 4. 清理临时 key
    s.redis.Del(ctx, tempKey)

    // 转换结果
    users := make([]TopUser, 0, len(results))
    for i, z := range results {
        users = append(users, TopUser{
            Rank:   int64(i) + 1,
            UserID: z.Member.(string),
            Score:  int64(z.Score),
        })
    }
    return users, nil
}
```

---

## 6. 数据库持久化

```go
// 定时同步 Redis → MySQL（最终一致性）
func syncToDB() {
    // 每 5 分钟同步一次
    ticker := time.NewTicker(5 * time.Minute)

    for range ticker.C {
        // 1. 获取全量数据（Redis → MySQL）
        results, _ := s.redis.ZRevRangeWithScores(ctx, leaderboardKey, 0, 9999).Result()

        tx, _ := db.Begin()
        for i, z := range results {
            tx.Exec(`INSERT INTO leaderboard_snapshot (date, rank, user_id, score)
                VALUES (CURDATE(), ?, ?, ?)
                ON DUPLICATE KEY UPDATE score = ?`,
                i+1, z.Member.(string), int64(z.Score), int64(z.Score))
        }
        tx.Commit()
    }
}

// Redis 宕机恢复：从 MySQL 加载
func recoverFromDB() {
    results, _ := db.Query(`SELECT user_id, score FROM leaderboard_snapshot
        ORDER BY score DESC LIMIT 100000`)

    pipe := s.redis.Pipeline()
    for _, row := range results {
        pipe.ZAdd(ctx, leaderboardKey, redis.Z{
            Score:  float64(row.Score),
            Member: row.UserID,
        })
    }
    pipe.Exec(ctx)
}
```

---

## 高频追问

### Q1：ZSet 底层数据结构是什么？为什么用跳表不用红黑树？

**ZSet 底层 = 哈希表（dict）+ 跳表（skiplist）：**
- dict：O(1) 查找 member 对应的 score
- skiplist：O(log N) 按分数排序，支持 ZRANGE/ZREVRANGE

**为什么用跳表，不用红黑树：**

| 维度 | 跳表 | 红黑树 |
|------|------|--------|
| 范围查询（ZRANGE）| ✅ O(log N + M) | ❌ 需要中序遍历，复杂 |
| 插入/删除 | ✅ O(log N) | ✅ O(log N) |
| 实现难度 | 简单 | 复杂 |
| 内存占用 | 稍高（多层指针）| 低 |
| 线程安全 | 不需要改算法 | 需要 RBTree 库 |

**结论**：Redis 作者 antirez 的解释：跳表实现简单，且范围查询（ZRANGE）比红黑树自然，不需要额外结构。

### Q2：好友排行榜怎么设计？

**核心思路：Redis ZSet 交集 ZINTERSTORE：**

```go
// 把好友列表变成 ZSet，分数设为 0
// 用 ZINTERSTORE 和全局排行榜求交集（保留全局排行榜的分数）
// 时间复杂度：O(N * log M)，N=好友数，M=全局人数
```

**替代方案（数据量大时）：**
```go
// 好友列表存 MySQL，查询时 JOIN 排行榜
SELECT u.user_id, u.score,
    RANK() OVER (PARTITION BY f.friend_group ORDER BY u.score DESC) as friend_rank
FROM friends f
JOIN leaderboard u ON f.friend_id = u.user_id
WHERE f.user_id = ?
```
**缺点**：延迟高（每次 JOIN），适合查询不频繁的场景。

### Q3：Redis 宕机怎么恢复？

**三层恢复策略：**
1. **AOF 持久化**（推荐）：`appendfsync everysec`，最多丢失 1 秒数据
2. **RDB 快照**（辅助）：定期 `BGSAVE`，恢复快但可能丢失一个快照周期
3. **MySQL 归档兜底**：定时同步 Top 10000 到 MySQL，宕机后从 MySQL 恢复

**Redis 宕机恢复流程：**
```
Redis 重启
    ↓
加载 AOF / RDB
    ↓
检查数据完整性
    ↓
若数据过旧：从 MySQL 加载 Top N 补充
    ↓
恢复正常服务
```
