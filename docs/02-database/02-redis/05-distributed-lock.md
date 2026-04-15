# Redis 分布式锁详解

> 面试频率：★★★★★  考察角度：单节点锁、Redlock、Go 实现、15 个坑

---

## 1. 为什么需要分布式锁？

多进程/多机器环境下，进程A和进程B可能同时修改同一份数据：

```
机器1: 订单服务 → 检查库存 → 库存=10 → 扣减1 → 库存=9
机器2: 订单服务 → 检查库存 → 库存=10 → 扣减1 → 库存=9  ← 重复扣减！
```

数据库的行锁只能锁本地进程，分布式场景必须用**分布式锁**。

---

## 2. 单节点 Redis 分布式锁

### 2.1 基本实现：SET NX EX

```go
func AcquireLockRedis(client *redis.Client, key string, ttl time.Duration) (string, error) {
    // 生成唯一 ID，用于安全释放
    lockValue := uuid.New().String()

    // SET key value NX EX ttl 原子操作
    ok, err := client.SetNX(ctx, key, lockValue, ttl).Result()
    if err != nil {
        return "", err
    }
    if !ok {
        return "", errors.New("lock already held")
    }
    return lockValue, nil
}

func ReleaseLockRedis(client *redis.Client, key, lockValue string) error {
    // ❌ 错误写法：先 GET 再 DEL，非原子
    // current := client.Get(key).Val()
    // if current == lockValue {
    //     client.Del(key)
    // }

    // ✅ Lua 脚本保证原子性
    script := `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `
    result, err := client.Eval(ctx, script, []string{key}, lockValue).Int()
    if err != nil {
        return err
    }
    if result == 0 {
        return errors.New("lock not held or expired")
    }
    return nil
}
```

### 2.2 看门狗（Watchdog）自动续期

单靠 EXPIRE 有问题：**持锁进程执行时间 > TTL**，锁自动释放后被其他进程获取，但原进程还在执行 → **数据不一致**。

**解决方案：看门狗自动续期**

```go
type DistributedLock struct {
    client  *redis.Client
    key     string
    value   string
    ttl     time.Duration
    stopCh  chan struct{}
    renewCh chan struct{}
}

// AcquireLockWithWatchdog 获取锁并启动看门狗自动续期
func AcquireLockWithWatchdog(ctx context.Context, client *redis.Client, key string, ttl time.Duration) (*DistributedLock, error) {
    lockValue := uuid.New().String()

    ok, err := client.SetNX(ctx, key, lockValue, ttl).Result()
    if err != nil {
        return nil, err
    }
    if !ok {
        return nil, errors.New("lock acquisition failed")
    }

    dl := &DistributedLock{
        client:  client,
        key:     key,
        value:   lockValue,
        ttl:     ttl,
        stopCh:  make(chan struct{}),
        renewCh: make(chan struct{}, 1),
    }

    // 看门狗 goroutine：自动续期 ttl/3
    go dl.watchdog()

    return dl, nil
}

func (dl *DistributedLock) watchdog() {
    ticker := time.NewTicker(dl.ttl / 3)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // 续期：如果锁还被自己持有
            script := `
                if redis.call("GET", KEYS[1]) == ARGV[1] then
                    return redis.call("PEXPIRE", KEYS[1], ARGV[2])
                else
                    return 0
                end
            `
            dl.client.Eval(context.Background(), script, []string{dl.key}, dl.value, dl.ttl.Milliseconds())
        case <-dl.stopCh:
            return
        }
    }
}

func (dl *DistributedLock) Unlock() error {
    close(dl.stopCh)
    script := `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `
    dl.client.Eval(context.Background(), script, []string{dl.key}, dl.value)
    return nil
}
```

---

## 3. Redlock（红锁）算法

单节点 Redis 锁存在**单点故障**风险。Redlock 通过多数节点共识解决：

### 3.1 算法原理

```
获取锁：
1. 获取当前时间 T1
2. 依次向 N 个 Redis 节点获取锁（SET key value NX EX ttl）
3. 计算耗时：T2 - T1
4. 成功条件：成功节点数 >= N/2 + 1 且 耗时 < TTL
5. 有效期 = TTL - 耗时

释放锁：
向所有节点发送 DEL 命令（不管是否成功获取）
```

### 3.2 Go 实现 Redlock

```go
type RedLock struct {
    clients []*redis.Client
    n       int // 总节点数
}

func NewRedLock(addrs []string) (*RedLock, error) {
    clients := make([]*redis.Client, len(addrs))
    for i, addr := range addrs {
        clients[i] = redis.NewClient(&redis.Options{Addr: addr})
    }
    return &RedLock{clients: clients, n: len(addrs)}, nil
}

func (rl *RedLock) Lock(ctx context.Context, key string, ttl time.Duration) (string, error) {
    lockValue := uuid.New().String()
    successCount := 0
    var successClients []*redis.Client

    // 并发向所有节点获取锁
    var wg sync.WaitGroup
    for _, client := range rl.clients {
        wg.Add(1)
        go func(c *redis.Client) {
            defer wg.Done()
            ok, _ := c.SetNX(ctx, key, lockValue, ttl).Result()
            if ok {
                atomic.AddInt32((*int32)(unsafe.Pointer(&successCount)), 1)
                successClients = append(successClients, c)
            }
        }(client)
    }
    wg.Wait()

    quorum := rl.n/2 + 1
    if successCount < quorum {
        // 获取锁失败，释放已获取的锁
        for _, c := range successClients {
            c.Del(ctx, key)
        }
        return "", errors.New("failed to acquire redlock")
    }

    return lockValue, nil
}
```

### 3.3 Redlock 的争议

> **Martin Kleppmann（分布式系统大牛）对 Redlock 的批评：**
> - 5 个 Redis 节点时钟漂移可能导致锁失效
> - Redlock 不是**线性一致性**的，无法作为分布式系统的唯一时钟源
> - 如果要严格分布式锁，建议用 **ZooKeeper / etcd** 的一致性算法

**实际工程选择**：

| 场景 | 推荐方案 |
|------|----------|
| 简单场景：单机 Redis + 高可用 | SET NX EX + 看门狗 |
| 严格分布式锁（数据一致性要求高） | etcd / Consul（Raft 共识） |
| 允许短暂不一致（幂等控制） | Redis 单节点锁足够 |

---

## 4. 分布式锁的 15 个坑

### 4.1 锁过期了任务还没执行完

→ **用看门狗续期**（上面已讲）

### 4.2 释放了别人的锁

→ **用唯一 ID 标识锁持有者，DEL 前先检查**（Lua 脚本）

### 4.3 主从切换导致锁丢失

```text
Client A 获取 Redis Master 锁
Master 宕机，Slave 晋升为 Master（数据未同步）
Client B 从新 Master 获取同一把锁成功
→ 两客户端同时持有同一把锁！
```

→ **用 Redlock** 或选择 etcd / ZK

### 4.4 误把可重入锁当不可重入锁用

可重入锁实现：

```go
type ReentrantLock struct {
    client *redis.Client
    key    string
    ctx    context.Context
}

func (l *ReentrantLock) Lock() error {
    // 通过 ThreadLocal 或 Context 传递锁计数
    if l.isHolding() {
        l.incrHoldCount()
        return nil
    }

    ok, _ := l.client.SetNX(l.ctx, l.key, "1", 30*time.Second).Result()
    if !ok {
        return errors.New("lock not available")
    }
    l.setHolding(1)
    return nil
}
```

### 4.5 锁的 key 设计不规范

```
❌  lock:order:12345        ← 冲突风险高
✅  lock:order:12345:uuid   ← 加唯一标识
```

### 4.6 并发量大时大量请求失败

→ 考虑**退火策略**（随机延迟重试）+ **消息队列**代替锁

---

## 5. 面试高频追问

**Q：分布式锁和数据库行锁的区别？**
> 分布式锁是跨进程的锁，用于多机部署场景；数据库行锁是进程内的锁，用于单机数据库场景。分布式锁可以用 Redis/etcd/ZK 实现。

**Q：为什么用 SETNX 而不是先 GET 再 SET？**
> GET + SET 不是原子操作，可能出现并发问题（两个进程同时 GET 到空值，都 SET 成功）。SETNX 是 Redis 原子命令，保证只有一个进程能成功获取锁。

**Q：Redlock 真的安全吗？**
> 有争议。Martin Kleppmann 指出了时钟漂移问题。工程实践中，如果对锁的严格性要求极高，建议用 etcd/Consul；如果允许短暂的不一致，单节点 + 看门狗足够。

**Q：如何实现一个可重入的分布式锁？**
> 在 Redis 中用 Hash 结构存储 `{owner: uuid, counter: n}`，每次 Lock 时 counter+1，每次 Unlock 时 counter-1，归零时删除 key。
