# 缓存与数据库双写一致性：策略、陷阱与实战方案

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：Cache Aside / Write Through / Delayed Dual Delete / Canal + Binlog / 最终一致性

## 面试官考察意图

这道题是「数据库」模块中最经典的追问型题目，通常从这条链展开：

1. "你的系统用了 Redis 做缓存，怎么保证 MySQL 和 Redis 数据一致？"
2. "先更新 DB 还是先删 Cache？顺序反了会怎样？"
3. "删了 Cache 之后有并发读请求怎么办？"
4. "你说的这些方案在生产中真的够吗？有没有更可靠的？"
5. "Canal + Binlog 的方案实现起来麻烦吗？"

**区分度**在于能否说清每种方案的适用场景、失败路径和工程代价。

---

## 核心答案（30 秒版）

| 问题 | 一句话回答 |
|------|-----------|
| **最推荐的方案是什么？** | **Cache Aside Pattern（先写 DB，再删 Cache）**，简单可靠，90% 场景够用 |
| **如果删 Cache 失败了怎么办？** | 异步重试删除 + binlog 订阅兜底，或给 Cache 设过期时间作为最终一致性保障 |
| **先写 Cache 再写 DB 可以吗？** | **绝对不行！** DB 更新后 Cache 是新值，DB 回滚则不一致 |
| **Canal + Binlog 方案为什么更可靠？** | 应用不直接操作 Cache，由 Canal 监听 binlog 异步同步到 Cache，解耦且容错率高 |
| **哪种方案能实现强一致性？** | 理论上都不行，缓存天生就是最终一致性模型，设计时要接受弱一致 |

---

## 深度展开

### 1. 四种经典双写策略对比

#### 策略 A：Cache Aside Pattern（推荐 ★★★★★）

```
写流程：
1. UPDATE database SET value = ? WHERE id = ?
2. DELETE cache WHERE key = ?

读流程：
1. GET cache WHERE key = ?
   → HIT: 返回缓存值
   → MISS: 查 DB → 写入 Cache → 返回
```

**优点**：
- 实现简单，无额外组件依赖
- 只删不更新的策略减少了冗余写入
- 读的时候自动刷新缓存（懒加载）

**缺点**：
- 极端并发下可能出现短暂不一致（见下方时序图）
- 删除 Cache 可能因网络抖动失败

```
⚠️ 极端并发不一致场景（读多写少时极难出现）：

T1: 线程 W 更新 DB (old_val → new_val)     ✅ DB: new_val
T2: 线程 R 读 Cache — MISS                📭 Cache: 无
T3: 线程 R 读 DB → old_val              💾 DB: new_val ← R读到的是new_val(正确！)
T4: 线程 R 写 Cache = old_val            📭 Cache: old_val ← ❌ 脏数据
T5: 线程 W 删 Cache                       📭 Cache: 空
T6: 后续读走 DB 重新写入 new_val          💾 Cache: new_val ← 最终修正

结论：即使出现不一致，由于 W 会执行删 Cache，后续读会走 DB 修复。
前提条件是：W 的删 Cache 一定在 R 的写 Cache 之前完成（几乎必然）。
```

#### 策略 B：Write Through Pattern（写入时同步更新）

```
写流程：
1. Cache.write(key, value)  — 缓存层内部同时更新 DB
2. 缓存层负责持久化到 DB

读流程：同 Cache Aside
```

**优点**：
- 读写都在同一层处理，看起来一致性强
- 应用代码无感知

**缺点**：
- Redis 不支持原生的 Write Through（需要中间件如 Tair/CacheLib）
- 耦合度高，换存储引擎需改架构
- Redis 挂掉时整个写操作失败

#### 策略 C：Delayed Dual Delete（延迟双删）

```
写流程：
1. DELETE cache WHERE key = ?          -- 第一次删除
2. UPDATE database SET value = ?      -- 更新 DB
3. SLEEP(500ms)                        -- 等待旧读请求读完
4. DELETE cache WHERE key = ?         -- 第二次删除（防止漏网之鱼）

读流程：同 Cache Aside
```

**优点**：
- 解决了策略 A 中极少数并发读导致脏缓存的场景
- 二次删除作为兜底

**缺点**：
- `SLEEP` 时间是经验值，无法精确保证覆盖所有并发读
- 增加了写延迟，影响吞吐量
- 第二次删除也可能失败

#### 策略 D：Canal + Binlog 订阅（最可靠 ★★★★★）

```
写流程：
1. UPDATE database SET value = ?    -- 只写 DB，不碰 Cache

Canal 后台进程：
2. 监听 MySQL binlog
3. 解析出 UPDATE 事件
4. 发送消息到 MQ（或直接调 Cache 服务）
5. Cache 服务收到消息后 DELETE key

读流程：同 Cache Aside
```

**优点**：
- 应用层零耦合，DB 和 Cache 完全解耦
- binlog 保证了数据来源的可靠性
- MQ 提供了异步削峰和重试能力
- 即使某个 Cache 节点挂了也能恢复

**缺点**：
- 引入 Canal + MQ 等组件，架构复杂度增加
- 端到端延迟比实时删除稍高（几十毫秒级）

```go
// Go 中使用 Canal client 监听 binlog 示例（示意）
import (
    "github.com/darwin-design/go-mysql-canal"
    "github.com/sirupsen/logrus"
)

func setupBinlogSync() {
    // 连接 MySQL 服务器
    server := &canal.Server{
        Host:     "127.0.0.1",
        Port:     3306,
        User:     "repl_user",
        Password: "password",
    }

    c, _ := canal.NewCanal(server)
    
    // 注册 handler
    c.RegisterEventHandler(&MyEventHandler{
        rdb: redisClient,
    })
    
    // 启动同步
    config := canal.NewDefaultConfig()
    config.Dump_exe = ""           // 不使用 mysqldump
    c.RunFromGTID(config)
}

type MyEventHandler struct {
    rdb *redis.Client
}

func (h *MyEventHandler) OnEvent(ctx *canal.EventContext) error {
    event := ctx.Event.(*canal.RowsEvent)
    
    // 解析变更的行
    for _, row := range event.Rows {
        if len(row) >= 2 {
            key := fmt.Sprintf("entity:%s", row[0])
            go h.rdb.Del(context.Background(), key).Err()
        }
    }
    return nil
}
```

### 2. 方案选型决策树

```
                    ┌─ 数据一致性要求极高？
                    │    ├─ YES → Canal + Binlog + MQ 兜底
                    │    └─ NO  ↓
┌──────────────────┼──── 对实时性要求如何？
│                  │    ├─ 极高（<1ms）→ Cache Aside（本地删 Cache）
│                  │    └─ 一般（<100ms）→ Canal + Binlog 异步
│                  │
│   架构复杂度能接受吗？
│    ├─ 不能 → Cache Aside
│    └─ 能 → 往下看
│
└─ 写入频率高吗？
     ├─ 是（每次写都刷 Cache 太贵）→ Cache Aside + 过期时间
     └─ 否 → 上面任一方案都行
```

### 3. 给 Cache 设置过期时间的意义

```
无论采用哪种方案，都建议给 Cache 设置合理的过期时间：

好处 1：最终一致性兜底
  - 即使 Cache 没被删除，过期后会自动失效
  - 下次读走 DB，获取最新数据
  
好处 2：自我保护
  - 避免坏数据长期驻留 Cache
  - 降低排查问题的时间窗口

过期时间设计原则：
  - 热数据：30min ~ 2h（根据业务 QPS 和数据变化频率）
  - 普通数据：1h ~ 24h
  - 不常变数据：可更长，但必须有上限
  - 随机化：加上 ±5% 的随机偏移，避免批量同时过期
```

### 4. 生产实践注意事项

| 要点 | 说明 |
|------|------|
| **只删不更新** | 写操作优先用 DEL 而不是 SET，减少多余写入和网络开销 |
| **重试机制** | 删除 Cache 失败时，至少重试 3 次（指数退避） |
| **监控告警** | 记录 Cache 删除失败的次数，超过阈值触发告警 |
| **降级开关** | 当 Cache 故障时，能通过开关关闭 Cache，直连 DB |
| **压测验证** | 上线前要用实际并发量压测，确认不会频繁出现脏数据 |

---

## 高频追问

### Q1：Canal + Binlog 方案中，MySQL 主从切换会导致丢消息吗？

**有风险，但有对策：**

```
风险：主库宕机切换新主 → Canal 可能还在连旧主 → 漏掉部分 binlog
对策：
  ① Canal 配置 auto_reset=true，断线重连时从最近位点恢复
  ② 配合 GTID 模式，确保复制连续性
  ③ 在 Canal -> MQ -> Cache 之间加幂等处理（去重表/布隆过滤器）
  ④ 定期跑对账任务，修复可能的不一致
```

### Q2：能不能用 Redis Lua 脚本做原子性的 DB+Cache 更新？

**不能。** Redis Lua 只能操作 Redis 本身，无法操作 MySQL。Redis 和 MySQL 不在同一个事务上下文中：

```lua
-- Lua 脚本只能在 Redis 内部执行，不能调用 MySQL API
EVAL "return redis.call('DEL', KEYS[1])" 1 mykey
-- 这只是删了 Cache，DB 的更新必须在应用层单独做
```

如果要保证原子性，只能用 **分布式锁 + 串行化操作**，但这会严重损害性能。

### Q3：本地缓存 + Redis 缓存双层结构下，一致性怎么处理？

双层缓存的一致性问题更加复杂：

```
写流程：
1. UPDATE DB
2. DEL local_cache[key]         -- 立即清空（最快生效）
3. DEL redis_cache[key]         -- 稍后清空（可能有延迟）

读流程：
1. GET local_cache[key]       -- HIT → 返回
2. GET redis_cache[key]       -- HIT → 填入 local_cache → 返回
3. SELECT from DB             -- MISS → 填入两层 cache

关键点：Local Cache 必须短 TTL（5~10 秒），否则一致性更难保证。
```

---

## 延伸阅读

- [美团技术团队：Redis 缓存与数据库一致性](https://tech.meituan.com/2018/02/09/redis-cache-consistency.html)
- [阿里 Cana 文档：binlog 同步到 Redis](https://github.com/alibaba/canal/wiki/Redis-Sync)
- [Netflix Priam：备份恢复与一致性权衡](https://netflixtechblog.com/priam-cassandra-backup-and-recovery-at-netflix-part-1-e93f068e80c5)
