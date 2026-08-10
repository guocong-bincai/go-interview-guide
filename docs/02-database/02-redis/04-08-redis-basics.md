# Redis 基础高频题：为什么快 / 过期策略 / 内存淘汰

> 考察频率：★★★★★  优先级：P0（高频基础题，逢面必问）

---

## 面试官考察意图

这道题是 Redis 模块的"准入考试"。
初级工程师只能说出"单线程+纯内存+IO多路复用"这九字真言，高级工程师要能深入到 **单线程模型的具体分工（主线程 vs I/O 线程）、epoll/kqueue 的高效实现、过期键的惰性+定期删除配合机制、8 种内存淘汰策略的场景化选择**，并能结合生产问题（Big Key 触发 OOM、惰性删除延迟、淘汰策略选错导致缓存失效）给出分析。

---

## 核心答案（30 秒版）

**Redis 为什么快？**

| 因素 | 原理 | 细节 |
|------|------|------|
| 单线程模型 | 避免上下文切换和锁竞争 | 主线程负责命令解析、执行、响应；I/O 多线程（6.0+）仅处理网络读写 |
| 纯内存操作 | O(1) / O(log n) 复杂度 | 无磁盘 I/O 延迟 |
| IO 多路复用 | epoll/kqueue 事件驱动 | 单线程监听大量 fd，O(1) 获取就绪事件 |
| 高效数据结构 | 每种类型有专属底层编码 | SDS O(1) 长度、ziplist 紧凑、skiplist 高效范围查询 |

**过期策略：** 惰性删除（访问时检查）+ 定期删除（定时扫描部分 key），两者配合。

**内存淘汰策略：** 8 种，`no-eviction` 禁止驱逐、`allkeys-lru` 最常用（热点数据）、`volatile-lru` 仅淘汰带 TTL 的 key。

---

## 深度展开

### 1. Redis 为什么这么快

#### 1.1 单线程模型：避免竞争

```bash
# Redis 6.0 之后的 I/O 线程模型
┌──────────────────────────────────────────────────┐
│                  主线程（Main Thread）            │
│  ┌─────────┐  ┌─────────┐  ┌──────────────────┐  │
│  │ 接收请求 │→ │ 解析命令 │→ │ 执行命令 + 响应  │  │
│  └─────────┘  └─────────┘  └──────────────────┘  │
└──────────────────────────────────────────────────┘
        ↑ 阻塞式          ↑ 非阻塞
        │                │
   ┌────┴────────────────┴────┐
   │   I/O Threads（6.0+，可选） │
   │  处理网络读写，非命令执行   │
   └────────────────────────────┘
```

**为什么不用多线程？**
- Redis 的瓶颈不在 CPU，而在**内存和网络 I/O**
- 多线程引入锁竞争、上下文切换，反而拖慢纯内存操作
- Redis 6.0 引入多线程仅用于**网络读写**，命令执行仍是单线程

```bash
# 查看 Redis 配置
io-threads 4        # I/O 线程数，建议 4~8
io-threads-do-reads yes  # 启用多线程读取
```

#### 1.2 IO 多路复用：单线程监听大量连接

```go
// epoll 核心思想：内核维护就绪列表，用户态无需遍历所有 fd
// 伪代码
for {
    nfds := epoll_wait(epfd, events, MAX_EVENTS, timeout)
    for i := 0; i < nfds; i++ {
        // 逐个处理已就绪的事件
        handle(events[i])
    }
}
```

对比三种 I/O 模型：

| 模型 | 描述 | 问题 |
|------|------|------|
| **阻塞 I/O** | 每个请求占用一个线程 | 大量并发时线程耗尽 |
| **非阻塞 + select** | 单线程轮询，但有 1024 fd 限制 | 扫描效率低 |
| **epoll/kqueue** | 事件驱动，只处理已就绪的 fd | 零遍历成本 |

#### 1.3 高效数据结构

Redis 快不是因为"单线程"，而是因为**数据结构设计精细**，能用 O(1) 绝不 O(log n)：

```
O(1) 操作：GET/SET/DEL/EXISTS/SETNX/INCR
O(log n) 操作：ZADD/ZRANGE（跳表）
O(n) 操作：KEYS/SCAN/LRANGE（需注意）
```

**SDS（Simple Dynamic String）vs C 字符串：**

```c
// C 字符串
strlen(buf); // O(n) —— 每次都要遍历到 \0

// SDS
struct sdshdr {
    int len;    // O(1) 获取长度
    int free;   // 预分配空间，避免频繁 realloc
    char buf[];
};
```

**ziplist / listpack 紧凑编码：** 小数据用连续内存块存储，省指针开销，避免内存碎片。

### 2. 过期策略：惰性删除 + 定期删除

#### 2.1 惰性删除（Lazy Expulsion）

**原理：** 访问 key 时检查是否过期，过期则删除。

```go
// 伪代码
func get(key):
    if (key 已过期):
        delete(key)
        return nil
    return value
```

**优点：** 对 CPU 友好，只在访问时处理。
**缺点：** 大量过期 key 无人访问时无法清理，可能导致内存持续增长。

**生产问题：** 大 key 删除可能阻塞主线程。

```bash
# bigkey 删除的正确姿势（避免阻塞）
UNLINK key  # 异步删除，释放内存后台进行
# 对比 DEL（同步阻塞）
```

#### 2.2 定期删除（Active Expires）

**原理：** 每隔一段时间，抽查一批 key，删除过期的。

```bash
# 配置
hz 10              # 每秒执行 10 次定期扫描
activ expire 100   # 每次随机检查 100 个带 TTL 的 key
```

**扫描算法：**
1. 从过期字典（expires dict）中随机抽取 100 个 key
2. 删除其中已过期的
3. 如果超过 25 个过期，重复步骤 1（最多 25 次循环）

**为什么是 25 次循环？**
保证定期删除的 CPU 时间不超过 25ms（100ms / 4），不影响正常请求。

#### 2.3 过期策略配合机制

```
内存占用低 → 惰性删除处理访问到的过期 key，定期删除辅助清理
内存占用高 → 触发 maxmemory + 淘汰策略
```

**生产经验：**
```bash
# 监控过期 key 情况
redis-cli INFO stats | grep -i expire
expired_keys:12345

# 监控内存
redis-cli INFO memory | grep -i used
used_memory:123456789
```

### 3. 内存淘汰策略（8 种）

当 Redis 内存达到 `maxmemory` 时，触发淘汰策略：

#### 3.1 策略一览

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `noeviction` | 不淘汰，返回错误（默认） | 数据完全不能丢 |
| `volatile-lru` | 从带 TTL 的 key 中淘汰最近最少使用的 | 有明确冷热区分，允许部分数据丢失 |
| `allkeys-lru` | 从所有 key 中淘汰最近最少使用的 | **生产最常用** |
| `volatile-lfu` | 从带 TTL 的 key 中淘汰最少使用的 | 访问频率差异化明显 |
| `allkeys-lfu` | 从所有 key 中淘汰最少使用的 | 访问频率差异化明显 |
| `volatile-random` | 从带 TTL 的 key 中随机淘汰 | 几乎不用 |
| `allkeys-random` | 从所有 key 中随机淘汰 | 几乎不用 |
| `volatile-ttl` | 从带 TTL 的 key 中淘汰 TTL 最短的 | 有明确的 TTL 分层 |

#### 3.2 LRU vs LFU 区别

```
LRU（Least Recently Used）：最近访问时间越早越先淘汰
LFU（Least Frequently Used）：访问次数越少越先淘汰

LRU 场景：热点数据被访问后不会立即变热，适合访问模式稳定的场景
LFU 场景：热点数据访问频率差异大，适合有明确访问频率区分的场景
```

```bash
# 生产推荐配置
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-samples 5  # LRU/LFU 采样数，越大越精确但越慢
```

#### 3.3 生产踩坑

**坑 1：淘汰策略选错导致缓存雪崩**
```bash
# 错误配置
maxmemory-policy allkeys-random  # 随机淘汰，热点数据可能被清掉
# 导致缓存命中率骤降，大量请求打到 DB
```

**坑 2：没设置 maxmemory，导致系统 OOM**
```bash
# 默认 noeviction，但没设 maxmemory 时内存会无限增长
# 正确做法
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-policy volatile-lru  # 如果有明确 TTL 分层
```

**坑 3：FLUSHDB 触发淘汰行为**
```bash
# 如果用 volatile-xxx 策略，FLUSHDB 删除了所有 key，淘汰策略就不生效了
# 正确做法：定期主动删除过期 key
redis-cli EXPIRE mykey 3600  # 带 TTL，定期清理
```

### 4. Big Key 与内存优化

#### 4.1 Big Key 识别

```bash
# 方法 1：redis-cli --bigkeys（扫描整个库，较慢）
redis-cli --bigkeys

# 方法 2：scan 遍历 + debug OBJECT
redis-cli --scan | head -100 | xargs -I{} redis-cli DEBUG OBJECT KEYSPACE {} 

# 方法 3：MEMORY USAGE（推荐）
redis-cli MEMORY USAGE bigkeyname
```

#### 4.2 Big Key 处理

```bash
# String 类型 > 10MB → 拆分
SET article:1:content "......"  # 太大了
# 改用 Hash，拆分字段
HSET article:1 content "......"  # 单字段仍然大

# 正确做法：按 ID 拆分
SET article:1:part1 "......"   # 分片存储
SET article:1:part2 "......"
MGET article:1:part1 article:1:part2

# List 类型 → 异步删除
UNLINK big:list:key  # 不要用 DEL，会阻塞
```

---

## 高频追问

### Q1：Redis 6.0 多线程和传统多线程有什么区别？

Redis 6.0 的多线程**仅用于网络 I/O**（接收连接、读取请求、发送响应），命令执行仍是单线程。这与 Memcached 的多线程（每个请求一个线程）完全不同。好处：充分利用多核、避免锁竞争、保持单线程的命令执行语义。

### Q2：过期 key 被删除的时机是什么时候？

惰性删除：客户端访问 key 时发现已过期，立即删除。
定期删除：每 100ms 触发一次，主动扫描部分过期 key。
AOF 持久化时：重写期间会对过期 key 进行清理。
主从复制：从节点不会主动删除过期 key（依赖主节点 DEL 命令同步）。

### Q3：为什么 String 类型用 embstr 而不用 raw？

`embstr`（embedded string）在 Redis 3.0 之前用于保存短字符串（≤39 字节），将字符串直接嵌入 `redisObject` 结构中，只分配一次内存，减少内存碎片和小对象分配开销。Redis 3.0 之后改为 44 字节阈值，5.0 之后引入 `intset` 编码优化整数。`embstr` 缺点是内容不可修改（修改会触发重新分配）。

### Q4：内存淘汰和过期策略的关系？

过期策略是**主动/被动清理过期 key**，不触发淘汰机制。
内存淘汰是**当内存达到上限时，主动驱逐 key**。
两者独立工作：即使 key 没过期，当内存达到 maxmemory 时也会触发淘汰。

---

## 延伸阅读

- [Redis persistence - redis.io](https://redis.io/docs/management/persistence/)
- [Redis LRU algorithm - redis.io](https://redis.io/docs/reference/optimization/lru/)
- [Redis 6.0 I/O threading](https://redis.io/docs/reference/io-threading/)
- 《Redis 设计与实现》—— 黄健宏，数据结构与编码章节