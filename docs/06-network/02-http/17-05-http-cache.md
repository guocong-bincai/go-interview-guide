# HTTP 缓存机制：强缓存与协商缓存（Cache-Control / ETag / 304）

> 考察频率：★★★★★  优先级：P0
> 关键词：强缓存、协商缓存、Cache-Control、Expires、ETag、Last-Modified、304、Vary、CDN 缓存、cache busting

---

## 🎯 面试官考察意图

HTTP 缓存是"性能优化"和"线上问题排查"两条线的交汇点。面试官想确认三件事：

1. 你能不能**分清强缓存和协商缓存**，而不是把两个概念揉成一团；
2. 你知不知道**ETag 和 Last-Modified 的取舍**（为什么有了时间戳还要 ETag）；
3. 你有没有**踩过缓存的坑**——接口返回旧数据、CDN 缓存了 500 页面、改了代码用户还是看到老版本。

第 3 点是 5~8 年工程师和 3 年工程师的分水岭。只讲原理是背书，讲得出"怎么让缓存立即失效"才是工程能力。

---

## ⚡ 核心答案（30秒）

**缓存的本质：浏览器（或 CDN）拿到响应后，把"能不能复用"的判断权交给一组头部。**

| 阶段 | 判断依据 | 命中时状态码 | 是否发请求 |
|------|---------|------------|-----------|
| **强缓存** | `Cache-Control: max-age` / `Expires` | 200（from disk/memory cache） | ❌ 不发 |
| **协商缓存** | `ETag` + `If-None-Match` / `Last-Modified` + `If-Modified-Since` | 304 Not Modified | ✅ 发（但不带 body） |

**一句话记忆：**
> 强缓存 = 我自己说了算，不问服务器；协商缓存 = 我问一下服务器"还能用吗"，服务器说 304 就继续用本地副本。

**优先级：** `Cache-Control` > `Expires`；`ETag` > `Last-Modified`（两者同时存在时，`If-None-Match` 优先）。

---

## 🔬 深度展开

### 1. 完整判定流程图

```
浏览器发起请求
      │
      ▼
┌─────────────────────┐
│ 是否有本地缓存副本？  │──否──► 正常请求，走网络
└─────────────────────┘
      │是
      ▼
┌──────────────────────────────┐
│ 强缓存是否有效？               │
│ 检查 Cache-Control: max-age  │
│ 检查 Expires（HTTP/1.0 遗留） │
└──────────────────────────────┘
      │有效                    │过期
      ▼                        ▼
  200 (from cache)      ┌──────────────────────────┐
  不发请求               │ 协商缓存：带条件请求头      │
                        │ If-None-Match: <etag>     │
                        │ If-Modified-Since: <time> │
                        └──────────────────────────┘
                                   │
                        ┌──────────┴──────────┐
                        ▼                     ▼
                   304 Not Modified      200 + 新内容
                   读本地副本            更新本地缓存
```

### 2. Cache-Control 常用指令（重点）

| 指令 | 含义 | 典型场景 |
|------|------|---------|
| `max-age=3600` | 相对过期时间（秒），优先级高于 Expires | 静态资源 |
| `no-cache` | **可以缓存，但每次必须协商**（不是"不缓存"！） | HTML 入口 |
| `no-store` | **完全不缓存**，连磁盘都不写 | 含敏感信息的接口（账单、身份证） |
| `public` | 任何节点（含 CDN、代理）都能缓存 | 静态资源 |
| `private` | 只有浏览器能缓存，CDN 不能 | 用户个人数据 |
| `must-revalidate` | 过期后必须回源验证，不允许用过期的 | 强一致要求 |
| `s-maxage=600` | 只作用于共享缓存（CDN），覆盖 max-age | CDN 单独控制 |
| `immutable` | 有效期内**连协商都不发**，连刷新都直接用缓存 | 带 hash 的静态文件 |
| `stale-while-revalidate=60` | 过期后 60s 内先用旧值响应，后台异步更新 | 追求极致 TTFB |

> ⚠️ 高频陷阱：`no-cache` 是"必须协商"，`no-store` 才是"完全不缓存"。面试中把这两个说反，基本会被判定为"只背了八股没写过"。

### 3. ETag vs Last-Modified

| 维度 | Last-Modified | ETag |
|------|---------------|------|
| 精度 | 秒级 | 文件内容级（可实现字节级） |
| 语义 | 文件最后修改时间 | 内容指纹 |
| 缺陷 1 | 1 秒内多次修改感知不到 | — |
| 缺陷 2 | 文件内容没变但被 touch，时间变了 → 误判为已更新 | — |
| 缺陷 3 | 只能精确到秒，分布式环境下各机器时间不一致 | — |
| 生成方式 | 由文件系统 / 业务给定 | `弱 ETag: W/"xxx"` / 强 ETag（Nginx 默认是 `mtime-size` 的弱 ETag） |

**为什么有 Last-Modified 还要 ETag？**
因为时间戳有两个致命缺陷：**精度只有秒**、**"修改时间变了但内容没变"会误判**。ETag 用内容哈希（或 inode+size+mtime 组合）来标识，能精确判断内容是否真的变了。

> 💡 生产建议：静态资源交给 Nginx/对象存储自动生成 ETag；**动态接口返回的 ETag 一定要业务自己算**（比如对 JSON 响应体做 hash），不能让框架凭空生成——否则每次响应 ETag 都不同，协商缓存形同虚设。

### 4. Go 实战：服务端如何正确实现协商缓存

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"net/http"
	"strings"
	"time"
)

// 动态接口的协商缓存实现：对响应体做 hash 得到强 ETag
func withETag(body []byte, lastModified time.Time, w http.ResponseWriter, r *http.Request) {
	sum := sha256.Sum256(body)
	etag := `"` + hex.EncodeToString(sum[:8]) + `"` // 只取前 8 字节，够用且短

	w.Header().Set("ETag", etag)
	w.Header().Set("Last-Modified", lastModified.UTC().Format(http.TimeFormat))
	w.Header().Set("Cache-Control", "no-cache, private, must-revalidate") // 每次都协商

	// 1) 先看 If-None-Match（优先级高于 If-Modified-Since）
	// 注意：If-None-Match 可能是 "a", "b" 这样的列表，也可能带 W/ 前缀
	if inm := r.Header.Get("If-None-Match"); inm != "" {
		for _, tag := range strings.Split(inm, ",") {
			tag = strings.TrimSpace(tag)
			if tag == "*" || strings.TrimPrefix(tag, "W/") == etag {
				w.WriteHeader(http.StatusNotModified) // 304 不能带 body
				return
			}
		}
	}

	// 2) 再看 If-Modified-Since
	if ims := r.Header.Get("If-Modified-Since"); ims != "" {
		if t, err := http.ParseTime(ims); err == nil && !lastModified.Truncate(time.Second).After(t) {
			w.WriteHeader(http.StatusNotModified)
			return
		}
	}

	w.WriteHeader(http.StatusOK)
	w.Write(body)
}
```

**要点：**
- `http.ServeContent` 已经内置了 ETag/Last-Modified/Range 处理，静态文件直接用，不要手写；
- 手动实现时，**304 响应体必须为空**，只写状态码；
- `If-None-Match` 的值可能是逗号分隔列表，也可能是 `W/"xxx"` 弱 ETag，必须做前缀裁剪再比较。

### 5. Vary 头：被忽视的缓存正确性守卫

```http
HTTP/1.1 200 OK
Vary: Accept-Encoding, Accept-Language
Cache-Control: public, max-age=3600
```

`Vary` 告诉缓存：「同一个 URL 的这个响应，会因为这些请求头不同而不同，请按请求头分组存」。

**踩坑案例：**
接口按 `Accept-Language` 返回中英文，但没设 `Vary: Accept-Language`。CDN 第一个请求是英文，于是把英文响应缓存了 1 小时，所有中文用户都拿到英文内容。

**Gzip 同理**：Nginx 开了 gzip 但没配 `Vary: Accept-Encoding`，可能出现"客户端不支持 gzip 却收到 gzip 内容"的乱码。

### 6. 缓存失效的两套工程方案

#### 方案 A：版本号 / Hash 文件名（静态资源首选）

```html
<!-- 构建时把内容 hash 打进文件名，内容变了 URL 就变了，天然失效 -->
<link rel="stylesheet" href="/static/app.a3f9c2e1.css">
<script src="/static/vendor.7b2d41f0.js"></script>
```

配合超长缓存：
```nginx
location ~* \.(js|css|png|woff2)$ {
    add_header Cache-Control "public, max-age=31536000, immutable";
    # immutable：连用户按 F5 刷新都不走协商，极致性能
}
```

**注意**：`index.html` 绝对不能这样做，必须 `no-cache`，否则用户永远拿不到新的资源引用列表。

```nginx
location = /index.html {
    add_header Cache-Control "no-cache, must-revalidate";
}
```

#### 方案 B：主动刷新（CDN / 网关层面）

| 手段 | 说明 | 生效速度 |
|------|------|---------|
| CDN 刷新 URL | 调用云厂商 API 按 URL 或目录刷新 | 秒级~分钟级 |
| CDN 刷新目录 | 整个目录失效，量大时费用高 | 分钟级 |
| 版本化 URL 变更 | 前端入口换成新 URL，老缓存自然被绕过 | 立即 |
| 缓存 Key 加版本 | 网关把 `?v=2` 纳入 cache key | 立即 |

> 💡 大厂实践：**发版前自动刷新 `index.html` + 新版本静态资源带新 hash 文件名**，两者结合，既不用刷全量 CDN（省钱），用户也能立刻拿到新版本。

### 7. 缓存策略速查表（可直接抄）

| 资源类型 | Cache-Control | 理由 |
|---------|---------------|------|
| 带 hash 的 JS/CSS/图片/字体 | `public, max-age=31536000, immutable` | 内容即版本，永不变 |
| index.html / 入口页 | `no-cache` | 必须实时拿到新的资源引用 |
| 公开的列表/详情接口（可容忍 1min 陈旧） | `public, max-age=60, stale-while-revalidate=300` | 抗突发流量 |
| 用户私有接口（我的订单、余额） | `private, no-cache` | 不能被 CDN 共享，且要实时校验 |
| 含敏感信息的接口 | `no-store` | 连浏览器磁盘都不留 |
| 写操作（POST/PUT/DELETE） | 默认不缓存；需要幂等可加 `no-store` | 避免重复提交 |

---

## ❓ 高频追问

### Q1：用户说"我改了代码但页面没变"，你怎么排查？

按顺序排除：

1. **看响应头**：DevTools Network 面板看 Size 列。`(from disk cache)` / `(from memory cache)` 说明强缓存命中，`304` 说明协商缓存命中。
2. **看是哪个层缓存**：
   - 打开无痕窗口正常 → 浏览器缓存问题；
   - 无痕也旧 → CDN / Nginx / 网关缓存；
   - `curl -I` 拿到旧的 → 服务端或中间层缓存；
   - 只有某个用户旧 → 该用户的运营商缓存或本地代理。
3. **定位缓存头**：`curl -I https://api.xxx.com/path` 看 `Age`、`X-Cache`、`Cache-Control`。有 `Age: 1234` 说明是 CDN 缓存的旧响应。
4. **修复**：临时加 `Cache-Control: no-cache` 或刷新 CDN，长期用版本化 URL。

### Q2：为什么 POST 请求默认不缓存？

HTTP 规范中只有 GET/HEAD 是可缓存的（且响应需要显式允许）。POST 语义是"提交数据、可能改变服务端状态"，缓存 POST 响应会导致：
- 重复提交语义混乱（用户以为提交了，其实复用了旧响应）；
- 无法保证写操作的幂等性。

所以浏览器和 CDN 默认都不缓存 POST。需要缓存查询类请求时，也应该是 **POST → GET 的接口设计**（把查询参数放 URL），或者用 `no-store` 明确禁止。

### Q3：CDN 缓存了 5xx 错误页怎么办？

这是生产事故的经典场景。防御措施：

```nginx
# Nginx：错误响应不缓存
proxy_cache_valid 200 302 10m;
proxy_cache_valid 404 1m;
proxy_cache_valid 500 502 503 504 0s;   # 5xx 一律不缓存

# 或者：加一个只允许缓存 200 的缓存键
proxy_cache_bypass $http_upgrade;
proxy_no_cache $upstream_status;         # 上游状态码变了下游即不缓存
```

CDN 侧一般都有 `忽略 5xx 缓存` / `错误页缓存时长` 配置，务必确认默认值。**同时给源站做预热和健康检查，避免错误页被大量回源放大。**

### Q4：强缓存和协商缓存能同时存在吗？

能，而且是**推荐做法**：`Cache-Control: max-age=60` + `ETag` 一起用。

- 60s 内：强缓存命中，零请求；
- 60s 后：触发协商缓存；内容没变则 304（省流量，只走头部），变了则 200 新内容。

这就是"短期强缓存 + 协商兜底"的混合策略，兼顾性能与新鲜度。

### Q5：`Age` 和 `Date` 头在缓存里有什么用？

- `Date`：源站生成响应的时间；
- `Age`：该响应在**共享缓存中已经存活的秒数**。

判定是否过期不只是看 `max-age`，而是：

```
当前年龄 = Age + (当前时间 - Date 时间)
是否新鲜 = 当前年龄 < max-age
```

CDN 回源时会把 `Age` 累加，所以看到 `Age` 大于 `max-age` 的响应说明已经陈旧、可能即将失效。

### Q6：如何让某个接口"立刻全部失效"？

| 场景 | 做法 |
|------|------|
| 有 CDN | 调刷新 API（URL 级刷新，秒级） |
| 只有 Nginx 缓存 | `proxy_cache_purge` 模块或重载配置 |
| 应用层缓存 | 用发布订阅广播删除 key（Redis + 多实例监听） |
| 想彻底规避缓存 | URL 加版本号 / 时间戳 query |

**不要**靠"改 max-age=0 重新发版"来指望缓存失效——已经缓存的内容在过期前依然是旧响应。

---

## 💻 面试手写题

**题目：写一个 HTTP 中间件，对 GET 接口实现 ETag 协商缓存。**

```go
package middleware

import (
	"bytes"
	"crypto/sha1"
	"encoding/hex"
	"net/http"
	"strings"
)

// ETagMiddleware 缓存响应体并基于内容生成 ETag，命中则返回 304
func ETagMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// 只处理 GET/HEAD
		if r.Method != http.MethodGet && r.Method != http.MethodHead {
			next.ServeHTTP(w, r)
			return
		}

		rec := &responseRecorder{ResponseWriter: w, status: http.StatusOK}
		next.ServeHTTP(rec, r)

		// 只对成功且无编码的响应做 ETag（gzip 后内容变化，hash 必然不同）
		if rec.status != http.StatusOK || rec.Header().Get("Content-Encoding") != "" {
			return
		}

		sum := sha1.Sum(rec.body.Bytes())
		etag := `"` + hex.EncodeToString(sum[:]) + `"`

		// 协商：If-None-Match 命中直接 304
		if matchETag(r.Header.Get("If-None-Match"), etag) {
			// 必须在写入 header 前清掉 Content-Length
			w.Header().Del("Content-Length")
			w.Header().Set("ETag", etag)
			w.WriteHeader(http.StatusNotModified)
			return
		}

		w.Header().Set("ETag", etag)
		w.WriteHeader(rec.status)
		w.Write(rec.body.Bytes())
	})
}

func matchETag(header, etag string) bool {
	if header == "" {
		return false
	}
	if header == "*" {
		return true
	}
	for _, t := range strings.Split(header, ",") {
		if strings.TrimSpace(strings.TrimPrefix(strings.TrimSpace(t), "W/")) == etag {
			return true
		}
	}
	return false
}

type responseRecorder struct {
	http.ResponseWriter
	status int
	body   bytes.Buffer
}

func (r *responseRecorder) WriteHeader(code int) { r.status = code }
func (r *responseRecorder) Write(b []byte) (int, error) { return r.body.Write(b) }
```

**面试时可说的加分点：**
1. 只对 GET/HEAD 生效，POST 不做；
2. 压缩后的内容 hash 一定不同，所以遇到 `Content-Encoding` 直接跳过（或先压缩再算 hash）；
3. 304 响应必须清掉 `Content-Length`，否则浏览器会等待一个永远不会来的 body；
4. 生产里如果用 `http.ServeContent` 或 `http.ServeFile`，标准库已经处理好了 ETag 和 Range，不要重复造轮子。

---

## 📋 总结 checklist

- [ ] 能画出「强缓存 → 协商缓存 → 304」的完整判定流程
- [ ] 能说清 `no-cache` ≠ `no-store`
- [ ] 能讲出 ETag 相对 Last-Modified 的两个优势（秒级精度、内容指纹）
- [ ] 知道 `Cache-Control` 优先级高于 `Expires`，`ETag` 高于 `Last-Modified`
- [ ] 知道 `immutable` 和 `stale-while-revalidate` 的适用场景
- [ ] 能说出 `Vary` 的作用和一个真实踩坑案例
- [ ] 能给出「静态资源用 hash 文件名 + index.html no-cache」的失效方案
- [ ] 能说清 5xx 响应必须显式禁止缓存

---

## 📚 延伸阅读

- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [MDN: Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
- [Go 标准库 net/http: func ServeContent](https://pkg.go.dev/net/http#ServeContent)
