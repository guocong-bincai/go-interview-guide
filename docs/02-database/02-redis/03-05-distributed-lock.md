# Redis 分布式锁

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：SETNX、Redlock、原子性、过期时间、可重入、异步哨兵

## 为什么需要分布式锁

单机锁（Mutex）在多进程/多机器环境下完全失效：

```go
// 单机锁：多进程/多机器下失效
var mu sync.Mutex
mu.Lock()  // 只在单机进程内有效
// 机器 A 和机器 B 上的 goroutine 都会进入临界区！
```

分布式锁需要满足三个基本特性：
1. **互斥**：任意时刻只有一个客户端能持有锁
2. **不死锁**：即使某客户端崩溃，锁也要能自动释放（TTL）
3. **可重入**：同一客户端可重复获取锁

---

## 基础实现：SETNX + EXPIRE

### 第一版：先 SET 后 EXPIRE

```go
func lockV1(lockKey string, expire time.Duration) bool {
    // 问题：SET 和 EXPIRE 之间可能崩溃，导致死锁！
    result, _ := redis.SetNX(lockKey, "locked", 0).Result()
    if result {
        redis.Expire(lockKey, expire)
    }
    return result
}
```

### 第二版：SETNX + 设置过期时间（原子）

```go
func lockV2(lockKey, requestID string, expire time.Duration) bool {
    // Lua 脚本保证 SET + 过期时间原子执行
    script := `
        if redis.call("SET", KEYS[1], ARGV[1], "EX", ARGV[2], "NX") then
            return 1
        else
            return 0
        end
    `
    result, _ := redis.Eval(script, []string{lockKey}, requestID, int(expire.Seconds())).Result()
    return result == 1
}

func unlockV2(lockKey, requestID string) error {
    // 释放锁：只能释放自己持有的锁
    script := `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `
    _, err := redis.Eval(script, []string{lockKey}, requestID).Result()
    return err
}
```

---

## 完整分布式锁：Go + Redisson 风格

### 锁结构设计

```go
type RedisLock struct {
    client    *redis.Client
    key       string
    value     string        // 唯一标识（UUID + 线程ID），用于可重入和释放
    expire    time.Duration
    leaseTime time.Duration // Redisson 的最大持有时间（续期）
}

// 生成唯一锁持有者ID
func generateLockValue() string {
    return fmt.Sprintf("%s-%d-%d",
        uuid.New().String(),
        os.Getpid(),
        goin.Maxroutineid(),
    )
}
```

### 获取锁（可重试 + 阻塞/非阻塞）

```go
func (l *RedisLock) TryLock(ctx context.Context, retry int, retryInterval time.Duration) (bool, error) {
    for i := 0; i <= retry; i++ {
        ok, err := l.tryAcquire(ctx)
        if err != nil {
            return false, err
        }
        if ok {
            return true, nil
        }

        if i < retry {
            select {
            case <-ctx.Done():
                return false, ctx.Err()
            case <-time.After(retryInterval):
                // 继续重试
            }
        }
    }
    return false, nil
}

func (l *RedisLock) tryAcquire(ctx context.Context) (bool, error) {
    // SET key value NX PX milliseconds
    result, err := l.client.SetNX(ctx, l.key, l.value, l.expire).Result()
    if err != nil {
        return false, fmt.Errorf("redis setnx failed: %w", err)
    }
    return result, nil
}
```

### 释放锁（原子性检查 + 删除）

```go
func (l *RedisLock) Unlock(ctx context.Context) error {
    // 必须检查 value 是否是自己的（Lua 脚本保证原子性）
    script := `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `
    result, err := l.client.Eval(ctx, script, []string{l.key}, l.value).Result()
    if err != nil {
        return fmt.Errorf("redis unlock failed: %w", err)
    }
    if result == 0 {
        return fmt.Errorf("锁已过期或被其他客户端持有: key=%s", l.key)
    }
    return nil
}
```

---

## Watch Dog 自动续期（防止锁提前释放）

### 问题

如果任务执行时间超过锁的 TTL，锁会自动过期，导致其他客户端获取锁。

### 解决方案：Watch Dog 机制

```go
// Redisson 的续期机制：锁持有期间，后台线程自动续期
func (l *RedisLock) startWatchDog() {
    ticker := time.NewTicker(l.leaseTime / 3) // 每 1/3 TTL 续期一次
    go func() {
        for {
            select {
            case <-ticker.C:
                // 续期脚本：只有锁仍被自己持有时才续期
                script := `
                    if redis.call("GET", KEYS[1]) == ARGV[1] then
                        return redis.call("PEXPIRE", KEYS[1], ARGV[2])
                    else
                        return 0
                    end
                `
                result, _ := l.client.Eval(context.Background(),
                    script, []string{l.key}, l.value, int(l.leaseTime.Milliseconds())).Result()
                if result == 0 {
                    return // 锁已释放，停止续期
                }
            }
        }
    }()
}
```

**TTL 设置策略**：
- `expire`（TTL）：锁最大持有时间，任务超时后自动释放
- `leaseTime`（Watch Dog 续期间隔）：默认 TTL / 3
- 如果不需要 Watch Dog（任务时间固定），设置 `leaseTime = expire` 即可

---

## Redlock 算法（多节点分布式锁）

### 单机 Redis 锁的问题

如果 Redis Master 宕机，而锁还没同步到 Slave，可能导致锁丢失。

### Redlock 思路

在 **N 个独立 Redis 节点** 上获取锁，超过 N/2+1 节点成功才算获取成功。

```go
// Redlock: 在 N 个节点上获取锁
func (r *RedLock) Lock(ctx context.Context, resource, value string, ttl time.Duration) error {
    N := len(r.nodes)
    quorum := N/2 + 1
    start := time.Now()
    timeout := time.Duration(N) * time.Second // 总超时

    var lockedNodes []string

    for _, node := range r.nodes {
        if time.Since(start) > timeout {
            break
        }

        ok, err := tryLock(node, resource, value, ttl)
        if ok {
            lockedNodes = append(lockedNodes, node.addr)
        }
    }

    if len(lockedNodes) < quorum {
        // 获取失败，释放已获取的锁
        for _, node := range lockedNodes {
            tryUnlock(node, resource, value)
        }
        return fmt.Errorf("无法获取超过半数锁：%d/%d", len(lockedNodes), quorum)
    }

    // 校验 TTL 是否足够（防止网络延迟导致有效时间减少）
    elapsed := time.Since(start)
    if elapsed > ttl/2 {
        // TTL 剩余不足，可以选择重试
        ttl = ttl - elapsed
    }
    return nil
}
```

### Redlock 的争议

| 争议方 | 观点 |
|--------|------|
| **Martin Kleppmann** | Redlock 存在争议，不推荐使用，建议用 Zookeeper 或 etcd |
| **Redis 作者 Antirez** | Redlock 在特定网络条件下是安全的，适用于对可靠性要求不极端的场景 |

**现实选型建议**：
- 电商秒杀、库存扣减：单节点 Redis + Watch Dog 足够
- 金融交易、对可靠性极高：使用 etcd/Consul 的分布式锁
- Redlock：争议较大，生产中使用需谨慎评估

---

## 生产常见问题与解决方案

### 1. 锁过期，任务未执行完

**问题**：任务 A 获取锁，30s TTL，但任务需要 60s，30s 后锁被释放，任务 B 获取锁，AB 同时执行。

**解决方案**：
- Watch Dog 自动续期（Redisson 方式）
- 任务时间预估 + 设置足够长的 TTL（保守策略）
- 任务拆分为多个子任务，每个子任务独立加锁

### 2. 可重入锁实现

```go
// 可重入锁：用 ThreadLocal 记录持有次数
type ReentrantLock struct {
    client    *redis.Client
    key       string
    value     string
    threads   map[int64]*Counter // pid → count
    mu        sync.Mutex
}

type Counter struct {
    count int
}

func (l *ReentrantLock) Lock(ctx context.Context) error {
    pid := goin.Goid()

    l.mu.Lock()
    if c, ok := l.threads[pid]; ok {
        c.count++
        l.mu.Unlock()
        return nil // 可重入，直接返回
    }
    l.mu.Unlock()

    // 第一次获取
    ok, _ := l.client.SetNX(ctx, l.key, l.value, l.expire).Result()
    if !ok {
        return fmt.Errorf("获取锁失败")
    }

    l.mu.Lock()
    l.threads[pid] = &Counter{count: 1}
    l.mu.Unlock()
    return nil
}

func (l *ReentrantLock) Unlock(ctx context.Context) error {
    pid := goin.Goid()

    l.mu.Lock()
    c := l.threads[pid]
    if c == nil {
        l.mu.Unlock()
        return nil
    }
    c.count--
    if c.count > 0 {
        l.mu.Unlock()
        return nil // 还有重入次数，不释放
    }
    delete(l.threads, pid)
    l.mu.Unlock()

    // 只有完全释放时才 DEL
    return l.unlockActual(ctx)
}
```

### 3. 集群/Sentinel 模式下的分布式锁

```go
// Redis Sentinel 模式下的锁
type SentinelLock struct {
    addrs []string // Sentinel 地址列表
}

func (l *SentinelLock) Lock(ctx context.Context, key, value string, ttl time.Duration) error {
    // 遍历所有 Sentinel 节点
    for _, addr := range l.addrs {
        client := redis.NewClient(&redis.Options{Addr: addr})

        // 向 master 节点获取锁
        ok, err := client.SetNX(ctx, key, value, ttl).Result()
        if ok {
            return nil // 获取成功
        }
    }
    return fmt.Errorf("无法在 Sentinel 集群中获取锁")
}
```

---

## 分布式锁 vs 本地锁对比

| 维度 | 本地锁（sync.Mutex） | 分布式锁（Redis） |
|------|---------------------|----------------|
| 作用域 | 单进程内 | 多进程/多机器 |
| 性能 | 无网络开销，极快 | 有网络延迟 |
| 可靠性 | 进程崩溃时自动释放 | 需要 TTL + Watch Dog |
| 功能 | 基础互斥 | 可重入、公平锁、读写锁 |
| 实现 | Go runtime 保证 | 需要自己处理网络异常 |

---

## 面试高频追问

**Q1: SETNX 和 SET NX PX 有什么区别？**
→ `SETNX` 是单独的 key 不存在才设置命令（SET if Not eXists），没有设置过期时间的能力。如果获取锁后进程崩溃，无法自动释放。`SET key value NX PX ms` 原子地完成两件事：加锁 + 设置过期时间，是分布式锁的标准写法。

**Q2: 锁自动过期后，任务还在执行怎么办？**
→ 两种方案：① Watch Dog 机制（Redisson 实现），后台线程自动续期；② 任务执行前评估最坏耗时，TTL 设置为「预估时间 × 1.5」或更长。如果任务无法预估时间，用 Watch Dog。

**Q3: Redlock 和单节点 Redis 锁哪个更推荐？**
→ 大多数场景用单节点 Redis + Watch Dog 足够，Redlock 实际使用较少（有争议），且运维复杂度高。如果对可靠性要求极高，用 etcd/Consul 的分布式锁，它们基于 Raft 协议天然保证强一致性。

**Q4: 分布式锁的性能优化？**
→ ① 使用连接池而非每次新建连接；② 合理设置 TTL，避免锁长时间占用；③ 使用 Lua 脚本减少网络往返；④ 对一致性要求不高的场景，可以用分段锁（Redis 的多 key）替代全局锁。
