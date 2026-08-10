[🏠 首页](../../../README.md) · [🗤️ 数据库](../../README.md) · [🔴 Redis](../README.md)

---

# Redis 热 Key 与大 Key：识别、定位与解决方案

## 面试官考察意图

考察候选人对 Redis 生产问题的实战经验。
初级只能说出"热 key 用本地缓存，大 key 拆分"，高级要能讲清楚 **热 key 的识别方法（monitor/log/sampling）、多级缓存架构、本地缓存的一致性问题、大 key 的危害（内存不均、阻塞 AOF/主从复制）、以及离线分析 vs 在线识别的 tradeoff**，并能给出生产级的完整解决方案。

---

## 核心答案（30 秒版）

| 问题 | 核心危害 | 识别方法 | 解决方案 |
|------|----------|----------|----------|
| **热 Key** | 单节点 QPS 过高，网卡打满 | monitor 采样、Redis 慢日志、客户端统计 | 本地缓存 + 热点 key 打散 |
| **大 Key** | 内存不均、fork 阻塞、持久化卡顿 | `redis-cli --bigkeys`、`--memkeys` | 拆分为小 key、压缩、lazyfree |
| **big string** | value > 10MB 时危害极大 | 同上 + SCAN 离线分析 | Hash 存储、拆分 value |

生产原则：**防大于治**——写入时避免集中，存储时控制单 key 大小。

---

## 深度展开

### 1. 热 Key 问题（Hot Key）

#### 1.1 什么是热 Key

热 Key 指**被高频访问的 key**，访问频率远超其他 key，导致某个 Redis 节点 CPU / 网卡过载。

典型场景：
```
双十一商品详情页 → 所有用户访问同一个商品 ID
秒杀活动 → 万人抢同一个库存 key（seckill:product:123）
明星塌房热搜 → 大量请求访问同一话题 key
```

**判断标准（经验值）：**
- 单 key QPS > 集群总 QPS 的 10%，即可认定为热 key
- 例如：集群总 QPS 50 万，某 key 单 QPS 超过 5 万 → 热 key

#### 1.2 热 Key 的危害

```
┌──────────────────────────────────────────────────┐
│                   Redis 节点                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│  │  普通key │  │  热key  │  │  普通key │           │
│  │  100/s  │  │ 10万/s  │  │  100/s  │           │
│  └─────────┘  └─────────┘  └─────────┘           │
│                   ↑ 单线程处理                    │
│              该节点已饱和，其他 key 被拖累          │
└──────────────────────────────────────────────────┘
```

- **单节点 CPU 过载**：Redis 是单线程，热 key 导致其他请求排队，P99 延迟飙升
- **网卡打满**：单个 key 流量太大，占满带宽
- **分布式锁失效**：热 key 上加分布式锁，所有请求打到同一节点，失去分布式意义

#### 1.3 热 Key 识别方法

**方法一：客户端层统计（最推荐）**
```go
// 在 Redis 客户端层做统计，不影响 Redis 性能
type HotKeyDetector struct {
    mu    sync.RWMutex
    counter map[string]*Counter
    topK   *TopK // 滑动窗口 TopK
}

func (h *HotKeyDetector) Record(key string) {
    h.mu.Lock()
    defer h.mu.Unlock()
    h.topK.Add(key)
}

func (h *HotKeyDetector) GetTopHotKey(n int) []string {
    return h.topK.QueryTop(n)
}
```

**方法二：Redis MONITOR 采样（离线分析）**
```bash
# 生产环境禁止长时间 MONITOR，用短时采样
redis-cli MONITOR --intrinsic-latency 100 --count 10000 > /tmp/monitor.log 2>&1 &
# 分析热 key（用 Python 脚本统计）
python3 analyze_hotkey.py /tmp/monitor.log
```
> ⚠️ MONITOR 会严重降低 Redis 性能，每次采样不超过 30 秒

**方法三：`redis-cli --hotkeys`（Redis 4.0+）**
```bash
redis-cli --hotkeys --scan --pattern "*"
# 输出格式：hot key [key_name] with 12345 accesses
```

**方法四：Redis 慢查询日志**
```bash
# 延迟 > 1ms 的命令记录到慢日志
redis-cli CONFIG SET slowlog-log-slower-than 1000
redis-cli SLOWLOG GET 100
```

**方法五：网络抓包分析**
- 在 Redis 节点上用 `tcpdump` 抓包，离线分析 top key
- 优点：不占用 Redis CPU；缺点：运维成本高

#### 1.4 热 Key 解决方案

**方案一：本地缓存（Most Common）**

```
客户端                                        Redis
  │                                             │
  ├── GET hot:product:12345 ──────────────────────┼──→ 10万/s（网卡打满）
  │                                             │
  │  优化后：                                     │
  ├── 本地缓存查询 ────────────────────────────────→ 只有缓存未命中时访问 Redis
  │      ├── 命中 → 直接返回（0延迟）              │
  │      └── 未命中 → 查 Redis → 回填本地缓存     │
```

```go
// 使用 Caffeine（Java）或 Go 本地缓存
import "github.com/bluele/gcache"

var localCache = gcache.New(10000).
    LRULoad(func(key string) (interface{}, error) {
        // 缓存未命中，从 Redis 加载
        return redis.Get(ctx, key).Result()
    }).
    Build()

func GetProduct(pid string) (string, error) {
    // 先查本地缓存
    val, err := localCache.Get(pid)
    if err == nil {
        return val.(string), nil
    }
    // 本地缓存未命中，查 Redis（只有第一次）
    result, err := redis.Get(ctx, "product:"+pid).Result()
    if err != nil {
        return "", err
    }
    localCache.Set(pid, result)
    return result, nil
}
```

**本地缓存的问题：**
1. **数据一致性**：如何保证本地缓存和 Redis 数据一致？
   - **TTL 过期**：设置较短的 TTL（如 5~30s），接受短暂不一致
   - **发布订阅失效**：Redis 更新时，通过 MQ 通知所有实例清除本地缓存
2. **缓存雪崩**：大量本地缓存同时过期 → 击穿到 Redis
   - 方案：本地缓存 TTL = baseTTL + random(0, 5s)

**方案二：热点 key 打散（Hash Tag）**

如果热 key 是**聚合操作**（如排行、计数），可以把一个 key 拆成 N 个：
```
# 原方案：所有请求打到一个 key
RANK_GLOBAL → 10万/s

# 优化：拆成 100 个 key，路由分散
RANK_GLOBAL:{user_id % 100}
# 客户端用 Hash Tag 请求：ZUNIONSTORE dest 100 key1 key2 ... key100
```

**方案三：热点 key 备份到多个节点**

```
# 写：同时写入 3 个节点（主 + 2 个从）
redis.MSet("hot:key", val, "hot:key:backup1", val, "hot:key:backup2", val)

# 读：随机选一个节点
node := nodes[rand.Intn(len(nodes))]
```

**方案四：热点探测 + 自动扩容（Kubernetes HPA）**
- Prometheus 监控 Redis 节点 CPU/网卡 → 超过阈值自动扩容 Pod
- 缺陷：Redis Cluster 不支持动态添加节点（需要手动 resharding）

---

### 2. 大 Key 问题（Big Key）

#### 2.1 什么是大 Key

单 key 的 value 过大，或 key 持有的元素数量过多：
```
字符串类型：value 长度 > 10MB
集合类型：元素数量 > 1万个
哈希类型：field 数量 > 1万个
```

**典型反面案例：**
```go
// 把整个商品库列表存进一个 key
redis.Set("all_products", JSONMarshal(products)) // 假设 products 有 100 万条
// 后果：GET all_products 耗时 > 1s，阻塞单线程，其他请求全部超时
```

#### 2.2 大 Key 的危害

| 危害 | 原因 | 后果 |
|------|------|------|
| **内存不均** | Redis Cluster 按槽分片，大 key 导致节点内存差异大 | 部分节点 OOM，其他节点空闲 |
| **阻塞主线程** | RDB / AOF 重写时遍历大 key | save 阻塞，持久化失败 |
| **fork 阻塞** | bgsave 时 fork 复制大内存，阻塞主进程 | 主线程停顿几百毫秒～几秒 |
| **主从复制卡顿** | psync 传输大 value，主从数据不一致窗口大 | 从节点数据滞后，切换时丢失数据 |
| **删除阻塞** | DEL 命令同步删除大 key，主线程阻塞 | 删除一个含 100 万元素的 zset，耗时 > 10s |

#### 2.3 大 Key 识别方法

**方法一：`redis-cli --bigkeys`（在线扫描，推荐）**
```bash
redis-cli -h <host> -p 6379 --bigkeys
# 输出每种类型最大的 key，给出 size 和类型
# 优点：不影响 Redis 性能（SCAN 实现）；缺点：只能找到单类型最大 key
```

**方法二：`redis-cli --memkeys`（内存视角）**
```bash
redis-cli --memkeys
# 按内存占用排序，Top N
```

**方法三：离线 SCAN 分析脚本**
```python
import redis

r = redis.Redis(host='localhost', port=6379)
big_keys = []

for key in r.scan_iter(count=1000):
    key_type = r.type(key).decode()
    if key_type == 'string':
        size = r.strlen(key)
    elif key_type in ('list', 'set', 'zset', 'hash'):
        size = r.memory_usage(key)  # Redis 4.0+
    else:
        size = 0
    
    if size > 10 * 1024 * 1024:  # > 10MB
        big_keys.append((key, key_type, size))

# 按 size 排序输出
for key, t, size in sorted(big_keys, key=lambda x: x[2], reverse=True)[:20]:
    print(f"{key}: {t}, {size/1024/1024:.1f}MB")
```

**方法四：`redis-cli INFO memory` 查看内存分布**
```bash
redis-cli INFO memory | grep used_memory_human
# 观察 memory_fragmentation_ratio > 1.5 说明有内存碎片，可能与大 key 有关
```

#### 2.4 大 Key 解决方案

**原则：拆大化小，控制单 key 大小在 100KB 以内**

**方案一：String 类型大 value → 拆分**

```
# 原方案（危险）
SET article:10000 "《Go高级工程师实战》全文内容...（20MB）"

# 优化：拆分为小片段
SET article:10000:part1 "第1-1000字..."
SET article:10000:part2 "第1001-2000字..."
SET article:10000:meta '{"total_parts": 20, "total_size": "20MB"}'
# 客户端按需拼接
```

**方案二：Hash 类型大 field → 字段拆分或压缩**

```
# 原方案（危险）：一个哈希有 100 万 field
HSET user:12345 "field1" "val1" "field2" "val2" ... "field1000000" "val..."

# 优化：按 Hash 结构存，但单个 field 的 value 不要太大
HSET user:12345 "profile" '{"name":"张三","age":30}'
HSET user:12345 "settings" '{"theme":"dark","lang":"zh"}'

# 或：用压缩
HSET user:12345 "profile:gz" "<gzip压缩后的数据>"
```

**方案三：List/Set/ZSet 大集合 → 分桶**

```
# 原方案：一个 ZSet 有 10 万元素（排行）
ZADD leaderboard "score1" "player1" "score2" "player2" ...

# 优化：按时间/ID 分桶（按月分表思想）
ZADD leaderboard:2026:04 "score1" "player1" ...
ZADD leaderboard:2026:05 "score2" "player2" ...

# 查询4月 TOP100：ZREVRANGE leaderboard:2026:04 0 99
```

**方案四：Lazyfree 异步删除**

```bash
# 启用 lazyfree，对大 key 删除不阻塞主线程
redis-cli CONFIG SET lazyfree-lazy-eviction yes
redis-cli CONFIG SET lazyfree-lazy-expire yes
redis-cli CONFIG SET lazyfree-lazy-server-del yes
redis-cli CONFIG SET slave-lazy-flush yes
```

```go
// Go 中使用 UNLINK（异步删除）代替 DEL
// go-redis 默认支持
rdb.Unlink(ctx, "big:zset:12345")
// 等效 DEL，但 Redis 用后台线程释放内存，不阻塞主线程
```

**方案五：内存压缩（Redis 7.0+）**

Redis 7.0 引入了 **listpack**（替代 ziplist），对小型集合有更好的内存效率：
- listpack 不存储反向长度字段，内存更紧凑
- 对小 value 场景自动生效，无需改代码

**方案六：AOF 持久化优化**

大 key 对 AOF 影响巨大：
```bash
# AOF 刷盘策略选择
appendfsync always    # 每条命令刷盘，最安全但最慢，不推荐
appendfsync everysec # 每秒刷盘，最多丢1秒数据，推荐
appendfsync no       # 依赖 OS，不推荐

# 如果 AOF 重写触发 bgsave，大量写入会阻塞
# 解决方案：用 SSD 盘，或改用 RDS（云数据库）
```

---

### 3. 综合问题排查流程

```
收到 Redis 延迟告警
        │
        ├── 查 slowlog
        │   redis-cli SLOWLOG GET 10
        │
        ├── 查 bigkeys
        │   redis-cli --bigkeys --scan
        │
        ├── 查热keys
        │   redis-cli --hotkeys
        │
        ├── 查客户端连接数
        │   redis-cli INFO clients
        │   # 如果 connected_clients > 10000，检查连接池泄漏
        │
        └── 查内存碎片
            redis-cli INFO memory | grep mem_fragmentation_ratio
            # > 1.5 → 需要重启或 CONFIG RESETSTAT
```

---

### 4. 面试高频追问

**Q1：本地缓存和 Redis 缓存不一致怎么解决？**
> 核心思路：接受最终一致，不追求强一致。
> 方案1（TTL兜底）：本地缓存设5s TTL，过期后自然重新读Redis
> 方案2（推送失效）：Redis 更新时，通过 pub/sub 广播"key已更新"，所有实例删除本地缓存
> 方案3（写操作双写）：写Redis成功后，同时写本地缓存（增加复杂度，慎用）

**Q2：大 key 删除时 Redis 卡住了怎么办？**
> 使用 `UNLINK`（异步删除）代替 `DEL`，后台线程释放内存。如果已经执行了 `DEL`，只能等待（Redis 单线程），下次用 `UNLINK`。

**Q3：热 key 分散到多个节点后，跨节点操作（如 ZUNIONSTORE）怎么办？**
> 这是 Cluster 的局限——跨槽操作需要客户端自己聚合。可以把相关数据放到同一个槽（用 Hash Tag），牺牲一定的均衡性换取引用。

**Q4：如何预防热 key，而不是事后处理？**
> 写入时：对 id 做 hash + random 打散到 N 个 key；产品设计：秒杀场景用"抢资格"代替"抢库存"（先到先得）。

---

## 延伸阅读

- [小林 coding - Redis 大 key 对持久化有什么影响？](https://xiaolincoding.com/redis/storage/bigkey_aof_rdb.html)
- [小林 coding - Redis 内存淘汰策略和过期删除策略](https://xiaolincoding.com/redis/module/strategy.html)
- Redis 设计与实现（第三版）：第 8 章持久化、第 9 章哨兵、第 10 章集群
