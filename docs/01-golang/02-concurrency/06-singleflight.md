# SingleFlight：并发去重与请求合并

> 考察频率：★★★★☆  优先级：P1
> 关键词：SingleFlight、请求合并、并发去重、缓存击穿、golang.org/x/sync/singleflight

## 面试官考察意图

考察候选人对 Go 扩展并发原语的掌握，以及在真实高并发场景下的问题解决能力。
初级只知道 SingleFlight 是"防止重复请求"的工具，高级要能讲清楚**请求合并（request coalescing）原理、与缓存击穿的关系、在 HTTP/gRPC 服务中的集成方式**，并能分析 SingleFlight 的局限性和替代方案。

---

## 核心答案（30 秒版）

`SingleFlight` 是 Go 官方扩展库（`golang.org/x/sync/singleflight`）提供的**请求合并**原语，作用是：**多个 goroutine 同时请求同一个 key 时，只让一个去执行，其他等待结果并共享返回值**。

```
无 SingleFlight（1000 并发请求同一 key）:
  请求A → 调用后端（慢，1s）
  请求B → 调用后端（慢，1s）  ← 999 个重复请求同时打到后端
  ...
  请求1000 → 调用后端（慢，1s）

有 SingleFlight:
  请求A → 调用后端（慢，1s）
  请求B → 等待A的结果共享 ← 不再打后端
  ...
  请求1000 → 等待A的结果共享 ← 不再打后端
  后端只被调用 1 次！
```

---

## 深度展开

### 1. 什么是请求合并（Request Coalescing）

请求合并解决的是**缓存击穿（Cache Stampede）**问题：当大量请求同时访问一个不存在的缓存 key 时，这些请求会同时穿透到数据库/后端，压垮服务。

SingleFlight 的核心思想：**用 map + channel 机制，让同一 key 的所有请求"合并"成一个请求执行，其他请求共享这个结果**。

### 2. SingleFlight 实现原理

```go
// golang.org/x/sync/singleflight/singleflight.go
type Group struct {
    mu sync.Mutex
    m  map[string]*call  // key → 正在执行中的 call
}

type call struct {
    wg  sync.WaitGroup  // 等待组
    val interface{}     // 返回值
    err error           // 错误
    dups int            // 共享结果的其他请求数
}

func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
    g.mu.Lock()

    // 第一次请求：创建 call，执行 fn
    if c, ok := g.m[key]; ok {
        // 已有相同 key 正在执行，加入等待
        g.mu.Unlock()
        c.wg.Wait()  // 等待第一个请求完成
        return c.val, c.err, true
    }

    c := new(call)
    c.wg.Add(1)
    g.m[key] = c
    g.mu.Unlock()

    // 执行真正的请求
    c.val, c.err = fn()
    c.wg.Done()  // 唤醒所有等待者

    // 从 map 中移除（允许下一个相同 key 的新请求）
    g.mu.Lock()
    delete(g.m, key)
    g.mu.Unlock()

    return c.val, c.err, c.dups > 0
}
```

**关键机制：**

```
第一次请求（key="product:1001"）：
  Do() → 创建 call → 放入 map → 执行 fn() → 完成后 delete(key)
  └─ 期间其他相同 key 的请求会：加入同一个 call 的 WaitGroup → 共享结果

为什么执行完要 delete？
→ 让下一个相同 key 的请求可以发起新的后端调用（数据可能已更新）
→ 避免永远只用一个旧结果
```

**DoChan：用 channel 返回结果（非阻塞等待）**

```go
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
    ch := make(chan Result, 1)
    go func() {
        v, err, _ := g.Do(key, fn)
        ch <- Result{v, err}
    }()
    return ch
}
```

**Forget：主动让某个 key 可被新请求替换**

```go
g.Do("expensive-query", expensiveQuery)  // 第一次
g.Forget("expensive-query")              // 主动从 map 移除
g.Do("expensive-query", expensiveQuery)  // 再次执行（绕过等待）
```

典型用途：同一个 key 但你知道底层数据已变化，需要立即发起新请求。

### 3. 生产使用场景

**场景一：防止缓存击穿**

```go
type Service struct {
    cache  *redis.Client
    sf     singleflight.Group
}

func (s *Service) GetProduct(ctx context.Context, id string) (*Product, error) {
    // 1. 先查缓存
    cached, err := s.cache.Get(ctx, "product:"+id).Result()
    if err == nil {
        var p Product
        json.Unmarshal([]byte(cached), &p)
        return &p, nil
    }

    // 2. 缓存未命中，用 SingleFlight 请求后端
    // 10000 个并发请求只会触发 1 次后端调用
    val, err, shared := s.sf.Do("product:"+id, func() (interface{}, error) {
        // 这里加分布式锁双重保险（可选）
        // 再查一次缓存（防止等待期间其他请求已写入）
        cached, err := s.cache.Get(ctx, "product:"+id).Result()
        if err == nil {
            var p Product
            json.Unmarshal([]byte(cached), &p)
            return &p, nil
        }

        // 查数据库
        p, err := s.db.QueryProduct(ctx, id)
        if err != nil {
            return nil, err
        }

        // 回填缓存
        data, _ := json.Marshal(p)
        s.cache.Set(ctx, "product:"+id, data, 5*time.Minute)
        return p, nil
    })

    if err != nil {
        return nil, err
    }

    if shared {
        // 打印日志：说明发生了请求合并，缓存击穿被阻止
        log.Printf("请求合并：product:%s 阻止了击穿", id)
    }

    return val.(*Product), nil
}
```

**场景二：防止数据库瞬时洪峰**

```go
// 新闻首页：某热点事件突然爆火 → 大量请求同时查询相同内容
func (s *Service) GetHotNews(ctx context.Context, newsID string) (*News, error) {
    news, err := s.sf.Do("news:"+newsID, func() (interface{}, error) {
        // 查数据库（或者另一个缓存层）
        return s.fetchFromDB(ctx, newsID)
    })
    // ...
}

// 测试验证：
// 模拟 100 个并发请求同一 newsID
// Without SF: 100 次 DB 查询
// With SF: 1 次 DB 查询 + 99 次等待
```

**场景三：限制热key的穿透率**

```go
// 限流思路：用 SingleFlight + 计数器
type HotKeyLimiter struct {
    sf    singleflight.Group
    count map[string]int
    mu    sync.RWMutex
}

func (l *HotKeyLimiter) GetOrFetch(ctx context.Context, key string) (interface{}, error) {
    // 对于访问频率超过阈值的 key，强制走 SingleFlight
    l.mu.RLock()
    freq := l.count[key]
    l.mu.RUnlock()

    if freq > 100 {  // 100 QPS 以上的热 key
        return l.sf.Do(key, func() (interface{}, error) {
            return l.fetch(ctx, key)
        })
    }
    return l.fetch(ctx, key)
}
```

### 4. 注意事项与坑

**坑1：WaitGroup 只等待一次**

```go
// 错误理解：Do 返回后能再次 Wait
c := new(call)
c.wg.Add(1)
// ...执行 fn...
c.wg.Done()
// WaitGroup 只允许一次 WaitGroup.Wait()
// 如果再次调用 Do("same-key")，会创建新的 call，不共享旧结果
```

**坑2：SingleFlight 不保证公平性**

```go
// 所有等待者共享同一个结果，但结果返回的顺序取决于 goroutine 调度
// 不适合"第一个返回就能用"的场景（应该用 context.WithTimeout）
```

**坑3：key 设计要精确**

```go
// ❌ 错误：key 太宽泛 → 无关请求也被合并
sf.Do("products", fn)  // 所有产品请求都合并成一个

// ✅ 正确：key 要精确到具体资源
sf.Do("product:1001", fn)  // 只有 product:1001 的请求合并
sf.Do("product:1002", fn)  // product:1002 是独立的
```

### 5. 扩展：分布式 SingleFlight

单机 SingleFlight 只能防止**单实例**的击穿，多实例部署时击穿仍会发生。

**方案：Redis + SETNX 分布式锁**

```go
func (s *Service) GetProductDistributed(ctx context.Context, id string) (*Product, error) {
    // 1. 查缓存
    // ...

    // 2. 分布式锁（Redis SETNX）
    lockKey := "lock:product:" + id
    locked, err := s.redis.SetNX(ctx, lockKey, "1", 10*time.Second).Result()
    if err != nil {
        return nil, err
    }

    if !locked {
        // 没拿到锁：等一小会儿，再查缓存
        time.Sleep(50 * time.Millisecond)
        return s.GetProduct(ctx, id)  // 递归或重新尝试
    }
    defer s.redis.Del(ctx, lockKey)

    // 拿到锁：执行查询 + 回填缓存
    return s.fetchAndCache(ctx, id)
}
```

---

## 高频追问

**Q：SingleFlight 和 Sync.Once 有什么区别？**

| | SingleFlight | Sync.Once |
|---|---|---|
| 目的 | 防止多个并发请求同时执行相同操作 | 确保函数只执行一次 |
| 等待 | 等待已执行的请求完成 | 不等待 |
| 返回值 | 所有调用者共享同一返回值 | 只给第一个调用者返回 |
| 适用 | 并发重复请求去重（缓存击穿）| 一次性初始化（连接池、配置加载）|

**Q：SingleFlight 能替代分布式锁吗？**

→ 不能。SingleFlight 是单机并发去重工具，数据不跨进程共享。多实例部署时，每个实例都有自己的 SingleFlight map，击穿问题仍然存在。分布式场景要用 Redis SETNX 或 RedLock。

**Q：SingleFlight 等待期间，如果有另一个相同 key 的请求加入，会再开一个吗？**

→ 不会。`Do` 方法内部有 `g.mu.Lock()` 保护 `g.m` map，只有第一个请求能创建新 call，后续请求发现 map 中已有相同 key，会直接加入该 call 的 WaitGroup，等共享结果。

**Q：Go 1.21 的 `sync.OnceFunc` 可以替代 SingleFlight 吗？**

→ 不能。`OnceFunc` 保证函数**在整个程序生命周期内只执行一次**，之后所有调用直接返回。SingleFlight 是**每个 key 合并一次请求**，key 执行完可以再次发起新请求。OnceFunc 适合初始化，SingleFlight 适合处理大量重复查询。

---

## 延伸阅读

- [golang.org/x/sync/singleflight（官方文档）](https://pkg.go.dev/golang.org/x/sync/singleflight)
- [SingleFlight 源码解析](https://github.com/golang/sync/blob/master/singleflight/singleflight.go)
- [微服务中的缓存击穿问题与解决方案](https://en.wikipedia.org/wiki/Cache_stampede)

---

**[← 上一篇：context 原理](./05-context.md)**
