# 读写分离延迟补偿：从原理到 Go 实战方案

> 考察频率：★★★★☆  难度：★★★★★
> 关键词：主从复制延迟、binlog delay、强制路由、Session Stickiness、最终一致性

## 面试官考察意图

这道题是「系统设计」+「数据库」交叉领域的经典高频追问：

1. "你的系统用了读写分离吗？" → 引出架构设计
2. "主库写完数据，从库多久能同步到？" → 引出复制延迟
3. "如果用户刚改了密码，立刻去查发现还没改好怎么办？" → 核心问题
4. "你们怎么解决这个问题的？" → 看实际经验

**区分度**在于能否说清复制延迟的根本原因（异步复制 vs 半同步）、不同方案的适用场景和代码级实现。

---

## 核心答案（30 秒版）

| 问题 | 一句话回答 |
|------|-----------|
| **为什么有延迟？** | MySQL 默认异步复制，主库不等待从库确认就返回成功；从库需独立回放 binlog |
| **典型延迟范围是多少？** | 正常 ~1~50ms，高并发写入时可达到秒级甚至几分钟 |
| **最简单的解决方案是什么？** | 写操作后立即读主库（强制主库读取），牺牲一点性能但保证一致性 |
| **高性能方案怎么设计？** | Session Stickiness + binlog 延迟监控 + 关键操作强制走主库 |
| **Go 客户端怎么实现？** | 通过连接池管理多数据源，拦截特定 SQL 或按 session key 路由 |

---

## 深度展开

### 1. 主从复制延迟的本质原因

```
异步复制流程：

┌──────────┐    ┌──────────────┐    ┌─────────────┐
│ Master   │    │ Binary Log   │    │ Slave IO     │
│          │──→ │ (Binlog)     │──→ │ Thread       │
│ Write OK │←── │              │←── │              │←── Relay Log
│ (立即返回)│    └──────────────┘    └─────────────┘
│           │                              │
│           │                              ▼
│           │                      ┌─────────────┐
│           │                      │ Slave SQL   │
│           │                      │ Thread      │
│           │                      │ (回放中继日志)│
└──────────┘                      └─────────────┘
                                   (应用看不到新数据！)
                                   
延迟来源：
1. Binlog 网络传输耗时
2. Slave 端磁盘 IO 回放速度
3. 高并发写入时 Slave 的 CPU/IO 资源竞争
4. 大事务导致 Slave 单线程回放阻塞
```

### 2. 五种解决方案对比

#### 方案一：写后强制读主库（最简单 ★★★★★）

```go
// 最推荐：用 session key 标记最近写过的 key，读时检查
type DBRouter struct {
    masterDB *sql.DB
    slaveDB  *sql.DB
    
    // 最近写了哪些 key，map[string]int64 存过期时间戳
    writeSessions sync.Map
}

func (r *DBRouter) SetWithMasterFlush(key string, value string) error {
    // 1. 写主库
    err := r.masterDB.Exec("UPDATE users SET name = ? WHERE id = ?", value, 1).Error
    if err != nil {
        return err
    }
    
    // 2. 设置 session 标记：这个 key 刚被写过
    r.writeSessions.Store(key, time.Now().Add(2*time.Second))
    
    // 3. 同时更新缓存（策略 A：Cache Aside Pattern）
    // ... 删除缓存逻辑
    
    return nil
}

func (r *DBRouter) GetUser(id int64) (*User, error) {
    userKey := fmt.Sprintf("user:%d", id)
    
    // 检查是否有未刷新的写操作
    if expire, ok := r.writeSessions.Load(userKey); ok {
        if time.Now().Before(expire.(time.Time)) {
            // 刚写过，读主库确保拿到最新数据
            return r.queryFromMaster(id)
        }
        r.writeSessions.Delete(userKey) // 过期了，清理掉
    }
    
    // 正常走从库（节省主库压力）
    return r.queryFromSlave(id)
}

func (r *DBRouter) queryFromMaster(id int64) (*User, error) {
    var u User
    err := r.masterDB.QueryRowContext(ctx, 
        "SELECT name, email FROM users WHERE id = ?", id).Scan(&u.Name, &u.Email)
    return &u, err
}

func (r *DBRouter) queryFromSlave(id int64) (*User, error) {
    var u User
    err := r.slaveDB.QueryRowContext(ctx,
        "SELECT name, email FROM users WHERE id = ?", id).Scan(&u.Name, &u.Email)
    return &u, err
}
```

**优点**：简单可靠，精确控制  
**缺点**：写完后第一次读需要额外 RTT（主库查询）

#### 方案二：Session Stickiness（会话粘滞）

```go
// 基于用户会话 ID 做路由
// 同一个 user 的读写都打到同一台实例

type StickyRouter struct {
    instances map[string]*sql.DB  // session_key -> db connection
    mutex     sync.RWMutex
}

func (r *StickyRouter) GetDB(sessionKey string) *sql.DB {
    r.mutex.RLock()
    db, ok := r.instances[sessionKey]
    r.mutex.RUnlock()
    
    if ok {
        return db  // 命中 session stickiness
    }
    
    // 没有则分配一个新实例（通常是主库）
    r.mutex.Lock()
    defer r.mutex.Unlock()
    
    // 双重检查
    if db, ok = r.instances[sessionKey]; ok {
        return db
    }
    
    newDB := r.allocateInstance(sessionKey)
    r.instances[sessionKey] = newDB
    return newDB
}

func (r *StickyRouter) allocateInstance(sessionKey string) *sql.DB {
    // 对写操作的 session 分派主库
    // 对读操作的 session 可轮询分配到从库
    return r.masterDB  // 简化的分配逻辑
}
```

**适合场景**：Redis Cluster、ShardingSphere 等中间件自带的粘性路由功能  
**局限**：跨机器部署时需要配合负载均衡器（如 Nginx sticky 模块）

#### 方案三：Binlog 延迟监控 + 自适应路由

```go
// 实时监控主从延迟，延迟超过阈值时自动切换到主库
type LatencyAwareRouter struct {
    masterDB  *sql.DB
    slaves    []*sql.DB
    
    // 从库延迟监控
    replicaLagMu sync.Mutex
    replicaLags  map[int64]time.Duration // server_id -> lag_seconds
}

func (r *LatencyAwareRouter) isReplicaBehindThreshold(slaveID int64, maxLag time.Duration) bool {
    r.replicaLagMu.Lock()
    defer r.replicaLagMu.Unlock()
    
    lag, ok := r.replicaLags[slaveID]
    if !ok || lag > maxLag {
        return true  // 延迟超标
    }
    return false
}

// 定期采集延迟指标
func (r *LatencyAwareRouter) monitorLoop() {
    ticker := time.NewTicker(5 * time.Second)
    for range ticker.C {
        for _, slave := range r.slaves {
            lag := r.getReplicationLag(slave)
            r.replicaLagMu.Lock()
            r.replicaLags[slave.ServerID] = lag
            r.replicaLagMu.Unlock()
        }
    }
}

// 获取从库延迟
func (r *LatencyAwareRouter) getReplicationLag(slave *sql.DB) time.Duration {
    var secondsBehind *int64
    err := slave.QueryRowContext(context.Background(),
        "SHOW SLAVE STATUS").Scan(nil, nil, nil, nil, nil,
        nil, &secondsBehind) // Last_SQL_Error 位置
    if err != nil || secondsBehind == nil || *secondsBehind == 0 {
        return 0
    }
    return time.Duration(*secondsBehind) * time.Second
}

// 查询时根据延迟动态决定
func (r *LatencyAwareRouter) Query(keywords string) []User {
    // 如果是涉及金钱/订单等强一致查询，始终走主库
    if isCriticalQuery(keywords) {
        return r.queryFromMaster(keywords)
    }
    
    // 非关键查询，优先选低延迟的从库
    r.replicaLagMu.Lock()
    bestSlaveIdx := -1
    bestLag := time.Hour
    for i, lag := range r.replicaLags {
        if lag < bestLag {
            bestLag = lag
            bestSlaveIdx = i
        }
    }
    r.replicaLagMu.Unlock()
    
    if bestSlaveIdx >= 0 && bestLag < 100*time.Millisecond {
        return r.queryFromSlave(r.slaves[bestSlaveIdx], keywords)
    }
    
    // 所有从库都延迟太高，降级到主库
    return r.queryFromMaster(keywords)
}
```

#### 方案四：SQL 改写提示（中间件层）

```
对于使用了 ShardingSphere / MyCat 等中间件的方案：

-- 强制走主库写法（中间件识别 /*master*/ 注释）
SELECT /*master*/ * FROM users WHERE id = 1;

-- 强制走某个从库
SELECT /*slave:1*/ * FROM users WHERE id = 1;

中间件层解析 SQL 中的 Hint，直接决定路由目标。
```

#### 方案五：半同步复制（缓解而非消除）

```ini
# MySQL 半同步复制配置
# Master 必须等待至少一个 Slave ACK 后才返回 COMMIT 结果

INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 3000;  # 超时 3s 退化为异步
SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

**效果**：将复制延迟从毫秒级降到极低（因为不是纯异步），但仍无法完全消除  
**代价**：写性能略有下降（需等待 Slave ACK）

### 3. 业务场景分级策略

```
┌───────────────────────────────────────────────┐
│  查询类型         │  路由策略                   │
├──────────────────┼────────────────────────────┤
│  账户余额查询      │ 强制走主库（不能容忍延迟）    │
│  登录/注册验证     │ 写后立即读主库               │
│  商品详情浏览      │ 可以走从库（允许短暂不一致）  │
│  用户个人信息      │ 写后 Session Stickiness     │
│  历史订单查询      │ 走从库即可                  │
│  排行榜/统计数据   │ 走从库或专门的 OLAP 实例     │
└───────────────────────────────────────────────┘
```

---

## 高频追问

### Q1：如何测试系统的读写分离是否正常工作？

```bash
# 方法一：手动制造延迟测试
# 在 Slave 上暂停 SQL 线程
STOP SLAVE SQL_THREAD;

# 此时在 Slave 上查询应该看到旧数据
mysql -h slave_host -e "SELECT * FROM users WHERE id = 1;"

# 恢复后验证
START SLAVE SQL_THREAD;

# 方法二：压测工具
# 用 wrk/wrk2 模拟高并发读写，观察延迟分布

# 方法三：应用内埋点
prometheus_http_request_duration_milliseconds{route="slave"}
prometheus_http_request_duration_milliseconds{route="master"}
# 观察从库 P99 延迟是否正常
```

### Q2：Canal 方案是否能替代读写分离延迟补偿？

**两者解决的问题不同：**
- **读写分离延迟补偿**：解决的是「同一请求中写的东西能不能立即读到」
- **Canal 方案**：解决的是「Cache 和 DB 的一致性」

Canal 不能代替读写分离延迟补偿，因为 Canal 是基于 binlog 的异步订阅，本身就有延迟。要解决同一请求内的即时可见性，必须走主库。

### Q3：Go 标准库 database/sql 能天然支持多数据源切换吗？

**不能直接支持。** `database/sql` 只绑定一个数据源。需要自己封装：

```go
type DualDataSource struct {
    masterPool *sql.DB
    slavePool  *sql.DB
    // ... 路由逻辑
}

// 或者使用更专业的库如 go-sharding
```

主流开源方案：ShardingSphere-Java/Go、MyBatis-Shardingsphere-Golang、自研路由中间件。

---

## 延伸阅读

- [MySQL Replication: Semi-Synchronous Replication](https://dev.mysql.com/doc/refman/8.0/en/replication-semisync.html)
- [Netflix Concurrency Labs: Database Consistency Testing](https://netflixtechblog.com/concurrency-labs-database-consistency-testing-at-netflix-fa7c152bb1e0)
- [Uber RIDE 博客：大规模分布式数据库实践](https://eng.uber.com/distributed-databases/)
