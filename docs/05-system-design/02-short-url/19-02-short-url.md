# 短链系统设计

> 考察频率：★★★☆☆  难度：★★★☆☆

## 核心问题

| 问题 | 描述 |
|------|------|
| 长链转短链 | 6-8 位字符映射任意长 URL |
| 302 vs 301 | 302 统计点击，301 永久重定向 |
| 碰撞 | 全局唯一 ID |
| 高并发 | 大量点击需要高效跳转 |

---

## ID 生成方案

### 方案一：自增 ID + 进制转换

```go
// 62 进制（0-9a-zA-Z）
const charset = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func encode(id int64) string {
    base := int64(len(charset))
    result := ""
    for id > 0 {
        idx := id % base
        result = string(charset[idx]) + result
        id /= base
    }
    return result
}

func decode(s string) int64 {
    base := int64(len(charset))
    result := int64(0)
    for _, c := range s {
        result = result*base + int64(strings.IndexByte(charset, byte(c)))
    }
    return result
}

// 示例：id=999999999 → "7PSRR0"
```

**优点**：简单，ID 单调递增
**缺点**：ID 暴露，可被遍历；需要分布式 ID 生成器

### 方案二：分布式 ID 生成器（雪花算法）

```go
func generateID() int64 {
    // 使用 Snowflake，详见 05-distributed-id
    // 或用 etcd/Redis INCR
}

// Redis 生成
func generateShortCode() string {
    id := rdb.Incr("short_url:counter")
    return base62Encode(id)
}
```

### 方案三：哈希 + 碰撞处理

```go
func hashLongURL(longURL string) string {
    // MD5 前 8 字节
    h := md5.Sum([]byte(longURL))
    code := base64.URLEncoding.EncodeToString(h[:8])
    code = strings.TrimSuffix(code, "==")
    code = strings.ReplaceAll(code, "+", "0")
    code = strings.ReplaceAll(code, "/", "0")

    // 查库，如果碰撞则加盐重试
    for i := 0; i <= 10; i++ {
        suffix := code
        if i > 0 {
            suffix = code + strconv.Itoa(i)
        }
        if !existsInDB(suffix) {
            return suffix
        }
    }
    return code[:6]
}
```

---

## 存储设计

### MySQL 表结构

```sql
CREATE TABLE short_urls (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    short_code VARCHAR(20) NOT NULL UNIQUE,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expired_at TIMESTAMP NULL,
    click_count BIGINT DEFAULT 0,
    INDEX idx_short_code (short_code),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Redis 缓存

```go
// 缓存热点短链
func getLongURL(shortCode string) (string, error) {
    // 先查 Redis
    longURL, err := rdb.Get("short:" + shortCode).Result()
    if err == nil {
        return longURL, nil
    }

    // 缓存未命中，查 DB
    longURL, err = dbGet(shortCode)
    if err != nil {
        return "", err
    }

    // 回填缓存，设置过期时间
    rdb.Set("short:"+shortCode, longURL, 24*time.Hour)
    return longURL, nil
}
```

---

## 302 vs 301 重定向

| 类型 | 码 | 含义 | 适用场景 |
|------|-----|------|---------|
| 301 | 永久重定向 | 浏览器缓存，SEO 权重传递 | 永久链接，不需统计 |
| 302 | 临时重定向 | 不缓存，每次都访问服务端 | 运营活动、需统计点击 |

**推荐用 302**：便于统计点击量，不泄露真实 URL。

```go
func redirect(c *gin.Context) {
    longURL, err := getLongURL(c.Param("code"))
    if err != nil {
        c.JSON(404, gin.H{"error": "not found"})
        return
    }
    // 异步更新点击量
    go incrementClickCount(c.Param("code"))
    c.Redirect(http.StatusFound, longURL)
}
```

---

## 高并发优化

### 热点数据

- Redis 缓存热点短链（命中率 > 90%）
- 多级缓存：CDN → Redis → MySQL

### 点击量异步处理

```go
func incrementClickCount(code string) {
    // 用 Redis INCR 批量写，减少 DB 压力
    rdb.Incr("click:" + code)
    // 定时任务每秒同步到 MySQL
}
```

### 多机房部署

- 每个机房独立生成 ID 段（避免跨机房调用）
- 路由层根据用户地理位置就近访问

---

## 完整代码

```go
// 生成短链
func createShortURL(longURL string) (string, error) {
    // 1. 查重：相同长链是否已存在
    existing := dbQueryByLongURL(longURL)
    if existing != "" {
        return existing, nil
    }

    // 2. 生成短码
    id := generateID()
    shortCode := encodeBase62(id)

    // 3. 存储
    err := dbInsert(shortCode, longURL)
    if err != nil {
        return "", err
    }

    // 4. 预热缓存（热点数据）
    rdb.Set("short:"+shortCode, longURL, 24*time.Hour)
    return shortCode, nil
}

// 重定向
func handleRedirect(c *gin.Context) {
    code := c.Param("code")
    longURL, err := getLongURL(code)
    if err != nil {
        c.JSON(404, gin.H{"error": "URL expired or not found"})
        return
    }
    c.Redirect(http.StatusFound, longURL)
}
```

---

## 总结

| 模块 | 方案 |
|------|------|
| ID 生成 | 雪花算法 / Redis INCR |
| 存储 | MySQL + Redis 缓存 |
| 跳转 | 302 重定向 |
| 统计 | 异步 Redis + 批量入库 |

### 面试话术

**Q：短链长度如何确定？**
> 6 位 62 进制 = 62^6 ≈ 568 亿，足够全球使用。7 位更宽松，但更影响美观和 URL 长度。

**Q：如何防止短链被遍历？**
> 使用 Snowflake 等无规律 ID 生成，不暴露自增序列。或在短码中混入随机数，碰撞检测。
