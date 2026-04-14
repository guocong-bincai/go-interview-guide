# Feed 流系统设计

> 考察频率：★★★★☆  难度：★★★★★

## 核心问题

| 问题 | 描述 |
|------|------|
| 高并发读 | 用户量大，Feed 读取 QPS 极高 |
| 写入放大 | 一个用户发消息，推送给所有粉丝 |
| 个性化 | 每个人看到的 Feed 不同 |
| 延迟 | 新消息需要尽快可见 |

---

## 三种架构模式

### 1. 拉模式（Fan-out on Read）

```
用户请求 Feed → 关注列表 → 拉取每个关注用户的最新 N 条 → 合并排序 → 返回
```

**Timeline Service**: 每个用户一个收件箱（列表），存储关注对象 ID

```go
type TimelineService struct {
    db *sql.DB
}

// 读取 Feed：按关注列表拉取
func (s *TimelineService) GetFeed(userID string, limit int) ([]*Post, error) {
    // 1. 获取关注列表
    following, _ := s.getFollowing(userID)

    // 2. 拉取每个用户的最新帖子
    var allPosts []*Post
    for _, uid := range following {
        posts := s.getUserPosts(uid, 20)
        allPosts = append(allPosts, posts...)
    }

    // 3. 合并排序（按时间）
    sort.Slice(allPosts, func(i, j int) bool {
        return allPosts[i].CreatedAt.After(allPosts[j].CreatedAt)
    })

    return allPosts[:limit], nil
}
```

**优点**：存储成本低，新帖子立即可见
**缺点**：读取成本高（关注者多时，需要拉取大量数据）

### 2. 推模式（Fan-out on Write）

```
用户发消息 → 写入自己的发件箱 → 异步推送到所有粉丝的收件箱
```

**Message Queue + Fan-out Service**:

```go
// 发帖
func publishPost(post *Post) error {
    // 1. 存储帖子
    db.Save(post)

    // 2. 发布到消息队列，异步推送
    for _, followerID := range getFollowers(post.AuthorID) {
        mq.Publish("feed_fanout", FeedMessage{
            UserID:  followerID,
            PostID:  post.ID,
            AuthorID: post.AuthorID,
        })
    }
    return nil
}

// 消费：写入粉丝收件箱
func fanoutWorker() {
    for msg := range mq.Consume("feed_fanout") {
        var m FeedMessage
        json.Unmarshal(msg.Data, &m)
        // ZADD 写入粉丝的 Redis 有序集合（按时间排序）
        rdb.ZAdd(fmt.Sprintf("feed:%s", m.UserID), redis.Z{
            Score:  float64(m.Timestamp),
            Member: m.PostID,
        })
    }
}

// 读取：直接从 Redis 读
func readFeed(userID string, offset, limit int) []*Post {
    postIDs, _ := rdb.ZRevRange("feed:"+userID, offset, offset+limit).Result()
    return getPostsByIDs(postIDs)
}
```

**优点**：读取极快（O(1) 从 Redis 取）
**缺点**：存储成本高（每个粉丝都要存一份）；新帖子推送延迟

### 3. 推拉结合（Hybrid）

```
普通用户发帖 → 推送给活跃粉丝（少量）
名人发帖 → 只写入自己的发件箱，拉模式读取（粉丝多，推送成本高）
```

```go
func publishPost(post *Post) error {
    followers := getFollowers(post.AuthorID)
    followerCount := len(followers)

    // 名人阈值：超过 1 万粉丝
    const CELEBRITY_THRESHOLD = 10000

    if followerCount < CELEBRITY_THRESHOLD {
        // 普通用户：推模式
        for _, fid := range followers {
            rdb.ZAdd(fmt.Sprintf("feed:%s", fid), redis.Z{
                Score:  float64(post.CreatedAt.Unix()),
                Member: post.ID,
            })
        }
    } else {
        // 名人：只写自己的发件箱，读的时候再拉
        saveToOutbox(post)
    }
    return nil
}
```

---

## 存储选型

| 数据 | 存储 | 理由 |
|------|------|------|
| 用户收件箱 | Redis ZSET | 按时间排序，O(log N) 写入，O(log N + M) 读取 |
| 帖子内容 | MySQL / HBase | 需要全文检索用 ES |
| 用户关系 | MySQL / 图数据库 | 好友关系查询 |

### Redis ZSET 实现收件箱

```go
// ZSET key = feed:{userID}
// Score = 帖子时间戳（Unix）
// Member = postID

// 写入：O(log N)
rdb.ZAdd("feed:123", redis.Z{Score: 1700000000, Member: "post_abc"})

// 读取第 0-19 条（最新）：O(log N + M)
postIDs, _ := rdb.ZRevRange("feed:123", 0, 19).Result()

// 分页读取第 20-39 条：O(log N + M)
postIDs, _ := rdb.ZRevRange("feed:123", 20, 39).Result()

// 删除过期帖子（超过 7 天）：O(M log N)
cutoff := time.Now().Add(-7 * 24 * time.Hour).Unix()
rdb.ZRemRangeByScore("feed:123", "-inf", strconv.FormatInt(cutoff, 10))
```

---

## 排序策略

### 时间线排序（Timeline）

```go
// 直接按时间倒序
postIDs, _ := rdb.ZRevRange("feed:"+userID, offset, offset+limit-1).Result()
```

### 智能排序（Feed Rank）

```go
// Score = timestamp * α + engagement * β
func calcScore(post *Post) float64 {
    now := time.Now().Unix()
    timeDecay := 1.0 / (float64(now-post.CreatedAt.Unix()) + 1)
    engagement := math.Log(float64(post.LikeCount+post.CommentCount+1))
    return engagement*0.7 + timeDecay*0.3
}

// 写入时计算
rdb.ZAdd("feed:"+userID, redis.Z{
    Score:  calcScore(post),
    Member: post.ID,
})
```

---

## 性能优化

### 缓存热点用户 Feed

```go
func getFeed(userID string) []*Post {
    // 先查本地缓存（L1）
    if posts := localCache.Get(userID); posts != nil {
        return posts
    }

    // 查 Redis（L2）
    postIDs, _ := rdb.ZRevRange("feed:"+userID, 0, 19).Result()
    posts := getPosts(postIDs)

    // 回填本地缓存
    localCache.Set(userID, posts, 30*time.Second)
    return posts
}
```

### 并发拉取合并

```go
func getFeedParallel(userID string, following []string) []*Post {
    ch := make(chan []*Post, len(following))
    var wg sync.WaitGroup

    for _, uid := range following {
        wg.Add(1)
        go func(uid string) {
            defer wg.Done()
            posts := getUserPosts(uid, 10)
            ch <- posts
        }(uid)
    }

    go func() {
        wg.Wait()
        close(ch)
    }()

    // 合并
    var all []*Post
    for posts := range ch {
        all = append(all, posts...)
    }
    sort.Slice(all, func(i, j int) bool {
        return all[i].CreatedAt.After(all[j].CreatedAt)
    })
    return all[:20]
}
```

---

## 总结

| 模式 | 适合场景 | 读延迟 | 写成本 |
|------|---------|--------|--------|
| 拉模式 | 粉丝少、内容多 | 高 | 低 |
| 推模式 | 粉丝多、内容少 | 低 | 高 |
| 推拉结合 | 混合场景 | 中 | 中 |

### 面试话术

**Q：推拉结合如何判断名人？**
> 维护粉丝计数器，超过阈值（如 1 万）标记为名人。名人发推走拉模式，避免向百万粉丝重复写入。普通用户走推模式，读取时直接命中缓存。

**Q：如何实现无限滚动分页？**
> 用 Redis ZSET + 时间戳分页。第一页用 `ZREVRANGE feed:uid 0 19`，下一页用 `ZREVRANGE feed:uid cursor 19+cursor`。Cursor 是上一页最后一条的时间戳（作为 Score），避免数据追加导致的分页错位。
