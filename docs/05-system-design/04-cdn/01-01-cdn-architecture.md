# CDN 架构设计：回源策略、缓存失效与热点分发

> 考察频率：★★★★☆  优先级：P1  字节/快手/抖音必考
> 关键词：CDN/PUSH_PULL/回源穿透/缓存失效/BFCache/Stale-While-Revalidate

---

## 核心答案（30 秒版）

| 维度 | 方案 | 适用场景 |
|------|------|----------|
| **加速方式** | PUSH（主动上传） / PULL（懒加载回源） | 静态资源用 PUSH，UGC 内容用 PULL |
| **缓存控制** | Cache-Control: max-age + ETag / If-Modified-Since | HTTP 标准协议，浏览器和 CDN 都支持 |
| **热点分发** | 热门文件预热到全边缘节点 vs 按地域分层回源 | 视频/大文件优先预热 |
| **回源保护** | 回源 QPS 限制 + 回源缓存 + 防盗链 | 防止恶意刷请求击穿回源带宽 |
| **实时刷新** | 域名级批量刷新 vs URL 精准刷新 | 发版时域名刷新，修图时 URL 刷新 |

---

## 深度展开

### 1. CDN 工作原理

```
用户请求 ──→ DNS 解析（GSLB 选最近节点）──→ 边缘节点(CDN Edge)
                                                │
                                          命中？──No──→ 回源站
                                            │                  │
                                           Yes                回源响应
                                            │                  │
                                   返回给用户              更新本地缓存
                                               │
                                        ┌──────▼──────┐
                                        │  L1: 浏览器  │  BFCache
                                        │  L2: CDN   │  边缘节点
                                        │  L3: 源站  │  原始服务器
                                        └────────────┘
```

### 2. PULL 模式 vs PUSH 模式的抉择

#### 2.1 PULL 模式（Lazy Loading）—— 更常用

```
1. 用户首次访问某个 URL → CDN 未命中
2. CDN 向源站发起回源请求（Origin Pull）
3. 源站返回资源，CDN 缓存并返回给用户
4. 后续请求直接从 CDN 缓存返回
```

```bash
# 源站配置回源规则
cache-control: public, max-age=3600  # 1小时缓存
etag: "v1-abc123"                     # 版本标识

# CDN 回源头部
Accept-Encoding: gzip, br            # 接受压缩，减少回源流量
X-Forwarded-For: 真实IP               # 传递真实用户IP
Host: origin.example.com             # 回源 Host
```

#### 2.2 PUSH 模式（主动推送）

适用于已知的、重要的静态资源：

```bash
# 通过 API 预下发文件到 CDN 所有节点
curl -X POST https://cdn.api/push \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{
    "urls": [
      "https://cdn.example.com/js/bundle.v2.0.js",
      "https://cdn.example.com/css/style.v2.0.css"
    ],
    "ttl": 86400
  }'
```

**选型判断表**：

| 对比维度 | PUSH | PULL |
|---------|------|------|
| 时效性 | ⭐⭐⭐⭐⭐ 立即可用 | ⭐⭐ 首访时才生效 |
| 存储成本 | CDN 节点需要预留空间 | 按需存储 |
| 管理复杂度 | 需要维护推送列表 | 自动处理 |
| 适用场景 | 前端框架/vendor JS/CSS | UGC 图片/文章/视频 |
| 删除能力 | 需额外调用刷新 API | 过期自动清理 |

### 3. 缓存控制策略详解

#### 3.1 Cache-Control 头部规范

```
Cache-Control: public, max-age=3600, must-revalidate
         │        │           │
         │        │           └── 过期后必须先与源站验证
         │        └── 缓存有效时间（秒）
         └── 允许任何中间层缓存

// 不同资源类型建议的 Cache-Control 配置：
CSS:          Cache-Control: public, max-age=31536000, immutable  // 带 hash 名，永久缓存
JS:           Cache-Control: public, max-age=31536000, immutable
图片:         Cache-Control: public, max-age=2592000              // 30天
HTML页面:     Cache-Control: no-cache                            // 每次校验
API响应:      Cache-Control: private, max-age=0                  // 不缓存

// 新版本化策略 —— 文件名包含 hash，内容变了文件名就变了，天然无缓存问题
<link rel="stylesheet" href="/css/style.a1b2c3d4.css">  // 内容改变 → hash 改变 → 新 URL
```

#### 3.2 ETag / If-Modified-Since 协商缓存

```
用户请求                              CDN                          源站
   │                                    │                           │
   │── GET /style.css ─────────────────→│                           │
   │         (首次，未命中)              │                           │
   │                                    │── GET /style.css ──────→│
   │                                    │                           │
   │                                    │←── 200 OK + ETag ───────│
   │                                    │   + Last-Modified
   │←── 200 OK (含 ETag) ───────────────│
   │         (存入 CDN 缓存)             │
   │                                    │                           │
   │── GET /style.css ─────────────────→│                           │
   │         If-None-Match: "abc123"    │                           │
   │         If-Modified-Since: xxx     │                           │
   │                                    │── GET /style.css ──────→│
   │                                    │   + IF-NONE-MATCH/MODIFIED│
   │                                    │                           │
   │                                    │←── 304 Not Modified ────│
   │←── 304 Not Modified ───────────────│
   │         (用本地缓存)                │                           │
```

### 4. 回源穿透与回源防护

#### 4.1 回源穿透

大量用户同时请求同一个冷资源（从未被 CDN 缓存过），导致大量并发请求直接打到源站。

**解决方案**：

```go
// CDN 侧的回源节流（伪代码逻辑）
type OriginShield struct {
    // 记录正在回源的 URL -> chan result，复用已有的回源请求
    inFlight sync.Map
}

func (s *OriginShield) HandleRequest(url string) (*Response, error) {
    // 检查是否已有其他请求在回源这个 URL
    if ch, ok := s.inFlight.Load(url); ok {
        return <-ch.(chan *Response) // 等待已有结果复用
    }

    // 没有人在回源，创建新的回源通道
    ch := make(chan *Response, 1)
    s.inFlight.Store(url, ch)

    go func() {
        defer s.inFlight.Delete(url) // 完成后移除
        resp := s.fetchFromOrigin(url) // 真正发起回源
        ch <- resp                     // 通知所有等待者
    }()

    return <-ch
}
```

**生产实践**：阿里云 CDN / Cloudflare 都内置了回源节流功能，无需自建。

#### 4.2 回源带宽保护

```yaml
# Nginx 回源限流配置示例
limit_req_zone $binary_remote_addr zone=origin_limit:10m rate=100r/s;

location @backend {
    limit_req zone=origin_limit burst=50 nodelay;
    proxy_pass http://origin_server;
}
```

### 5. 缓存实时刷新（全站更新场景）

```
场景：前端发版，需要让所有用户拿到最新 JS/CSS 文件

方案 A：URL 精准刷新（最快，但数量多时需要分批）
curl -X POST https://cdn.api/flush/url \
  -H "Content-Type: application/json" \
  -d '{"urls":["https://cdn.com/js/app.js","https://cdn.com/css/main.css"]}'
// 耗时：几秒到几十秒生效

方案 B：域名级刷新（一键清空整个域名的缓存）
curl -X POST https://cdn.api/flush/domain?domain=cdn.example.com

方案 C：版本号策略（推荐！零停机）
<link href="/js/app.v2.abc123.js">
// 部署新版本 → 改 HTML 引用为新 hash → CDN 自然缓存新文件
// 旧文件不会被刷新，直到 TTL 自然过期
```

**面试加分项**：
> 实际生产中不建议使用"域名级刷新"一次性清所有缓存，会造成大量回源请求。推荐使用"文件名 Hash + 增量刷新"策略——每次发版只生成新文件的新 URL，旧 URL 继续保留，依靠 TTL 自然淘汰。

### 6. Go 实现 CDN-like 本地缓存代理

```go
package cdncache

import (
    "io"
    "net/http"
    "sync"
    "time"
)

// HTTPCache 是一个简易的 HTTP 缓存代理，类似 CDN 的边缘节点
type HTTPCache struct {
    mu       sync.RWMutex
    entries  map[string]*cacheEntry
    maxSize  int
    maxSizeMB int
}

type cacheEntry struct {
    data      []byte
    headers   http.Header
    accessedAt time.Time
}

const defaultTTL = 30 * time.Minute

func NewHTTPCache(maxSizeEntries int, maxSizeMB int) *HTTPCache {
    c := &HTTPCache{
        entries:   make(map[string]*cacheEntry),
        maxSize:   maxSizeEntries,
        maxSizeMB: maxSizeMB,
    }
    // 定期清理过期条目
    go c.cleanupLoop()
    return c
}

func (c *HTTPCache) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    key := r.URL.String()

    // 尝试从缓存读取
    c.mu.RLock()
    entry, hit := c.entries[key]
    c.mu.RUnlock()

    if hit && time.Since(entry.accessedAt) < defaultTTL {
        // 缓存命中
        for k, vv := range entry.headers {
            for _, v := range vv {
                w.Header().Set(k, v)
            }
        }
        w.WriteHeader(http.StatusOK)
        w.Write(entry.data)
        return
    }

    // 缓存未命中，回源
    proxyResp, err := http.DefaultTransport.RoundTrip(r)
    if err != nil {
        http.Error(w, "upstream error", http.StatusBadGateway)
        return
    }
    defer proxyResp.Body.Close()

    body, _ := io.ReadAll(proxyResp.Body)

    // 写入缓存
    c.mu.Lock()
    if len(c.entries) >= c.maxSize {
        c.evictOldest()
    }
    c.entries[key] = &cacheEntry{
        data:       body,
        headers:    proxyResp.Header.Clone(),
        accessedAt: time.Now(),
    }
    c.mu.Unlock()

    // 返回给客户端
    for k, vv := range proxyResp.Header {
        for _, v := range vv {
            w.Header().Set(k, v)
        }
    }
    w.WriteHeader(proxyResp.StatusCode)
    w.Write(body)
}

func (c *HTTPCache) cleanupLoop() {
    ticker := time.NewTicker(5 * time.Minute)
    for range ticker.C {
        c.mu.Lock()
        cutoff := time.Now().Add(-defaultTTL)
        for key, entry := range c.entries {
            if entry.accessedAt.Before(cutoff) {
                delete(c.entries, key)
            }
        }
        c.mu.Unlock()
    }
}

func (c *HTTPCache) evictOldest() {
    var oldestKey string
    var oldestTime time.Time
    first := true
    for key, entry := range c.entries {
        if first || entry.accessedAt.Before(oldestTime) {
            oldestKey = key
            oldestTime = entry.accessedAt
            first = false
        }
    }
    if !first {
        delete(c.entries, oldestKey)
    }
}
```

### 7. 面试话术

**Q：CDN 不回源能扛多少 QPS？**

> 纯 CDN 缓存命中的情况，理论上取决于边缘节点的硬件能力。单台 Nginx/CDN 节点可以扛 5~10 万 QPS（取决于资源大小和网络带宽）。我们系统上线前做过压测，CDN 命中率达到 95% 时，源站只需要承受约 5% 的回源 QPS，大大减轻了源站压力。

**Q：如何设计一个全球分发的 CDN 架构？**

> 核心思路是就近服务和分层回源。用户 DNS 查询时由 GSLB（全局负载均衡）根据地理位置和延迟选择最近的边缘节点。全球部署 100+ POP 点，每个 POP 有 LRU 缓存层。当边缘节点未命中时，先尝试回源到区域中心节点（Region Hub），如果区域中心也没有才回源到原始站。这样可以避免所有回源请求都打到远端源站，同时利用 Region Hub 作为中转缓存降低回源带宽。
