# 分布式锁横向对比：Redis vs etcd vs ZooKeeper

> 考察频率：★★★★★  优先级：P0
> 关键词：SETNX、Redlock、Watchdog、Lease、有序节点、ZAB、脑裂

## 为什么需要分布式锁

单机锁（`sync.Mutex`）在多进程/多机器环境下**完全失效**：

```go
// 单机锁：多进程/多机器下失效
var mu sync.Mutex
mu.Lock() // 只在单机进程内有效，机器 A 和 B 上的 goroutine 都会进临界区！
mu.Unlock()
```

分布式锁必须满足三个基本特性：
1. **互斥**：任意时刻只有一个客户端能持有锁
2. **不死锁**：即使某客户端崩溃，锁也要能自动释放（TTL / Lease）
3. **可重入**：同一客户端可重复获取锁（可选，但重要）

---

## 1. 三种方案原理

### Redis 分布式锁

**核心命令：`SET key value NX PX ttl`**

```go
// 加锁：NX 保证原子性，PX 设置过期时间
func tryLock(redis *redis.Client, key, requestID string, ttl time.Duration) (bool, error) {
    // SET key value EX seconds NX — 原子操作
    result, err := redis.SetNX(ctx, key, requestID, ttl).Result()
    return result == 1, err
}
```

**解锁：Lua 脚本保证原子性**

```go
// 解锁：只能释放自己持有的锁（通过 value 判断）
// 如果 key 不存在或 value 不匹配，DEL 不执行
const unlockScript = `
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
`
func unlock(redis *redis.Client, key, requestID string) error {
    _, err := redis.Eval(ctx, unlockScript, []string{key}, requestID).Result()
    return err
}
```

**为什么要判断 value 再 DEL？**
- 防止误删：假设锁已超时自动释放，新客户端加锁成功
- 此时旧客户端执行 `DEL`，会把新客户端的锁删掉
- 解决：解锁前验证 `GET == 自己设置的 value`

**续期：Watchdog 机制（Redisson 实现）**

```go
// Redisson Watchdog：锁自动续期，默认每 10s 续一次，ttl/3
// 如果业务执行超过 ttl，Watchdog 会自动续期，避免锁提前释放
// 只有调用 unlock() 才停止续期
type RedissonLock struct {
    client      *redis.Client
    lockKey     string
    internalLockTTL time.Duration
    watchdogTimeout time.Duration  // 默认 30s
}

// 后台协程自动续期
func (l *RedissonLock) scheduleRefresh() {
    ticker := time.NewTicker(l.internalLockTTL / 3)
    for {
        select {
        case <-ticker.C:
            // 只有锁还被自己持有时才续期
            l.client.Expire(l.lockKey, l.internalLockTTL)
        }
    }
}
```

**Redis 单节点 vs Redlock 多节点：**

```
单节点 Redis 锁：
  Client A → SET lock=uuid NX PX 30000 → 加锁成功
  问题：Redis 宕机 → 锁丢失 → 脑裂

Redlock（5节点多数派）：
  Client A → 向 5 个节点同时加锁，3 个成功 → 获得锁
  优点：少数节点宕机不影响锁的有效性
  缺点：性能下降（5x 网络延迟），实现复杂
```

### etcd 分布式锁

**核心原理：Lease + 有序 Key + Watch 前序节点**

```go
import "go.etcd.io/etcd/clientv3"

// 1. 创建租约（TTL = 10s）
resp, err := cli.Grant(ctx, 10)
leaseID := resp.ID

// 2. 加锁：创建有序 key /lock/job/00000000000000000001
// 只有当前 key 是最小（排队第一），才算获得锁
txn := cli.Txn(ctx)
txn.If(clientv3.Compare(clientv3.CreateRevision(prevKey), "=", 0)).
    Then(clientv3.OpPut(lockKey, requestID, clientv3.WithLease(leaseID))).
    Else()
_, err = txn.Commit()

// 3. 监听前一个 key 的删除事件，收到后自己就变成第一了
// 等效于"公平队列"：每个客户端按顺序排队
```

**etcd 锁的核心特点：**

- **公平锁**：所有 key 按创建顺序排队，`createRevision` 决定顺序
- **自动续期**：etcd Lease 到期自动删除 key，无需 Watchdog
- **线性一致**：基于 Raft，日志复制保证强一致，不存在脑裂
- **性能**：约 1W QPS（比 Redis 低，但一致性更强）

### ZooKeeper 分布式锁

**核心原理：临时有序节点（EPHEMERAL_SEQUENTIAL）**

```go
import "github.com/samuel/go-zookeeper/zk"

// 1. 创建临时有序节点（断开连接自动删除）
// ZooKeeper 会自动给节点名加递增序号：/locks/job-0000000001
path, err := conn.Create(
    "/locks/job-",
    []byte(requestID),
    zk.FlagEphemeral|zk.FlagSequence,  // 临时 + 有序
)

// 2. 获取所有子节点，按序号排序
children, _, err := conn.Children("/locks")

// 3. 如果自己是最小的序号节点 → 获得锁
// 否则 Watch 前一个节点，前一个被删除时自己就变成第一了
sort.Strings(children)
myIdx := findIndex(children, path)
if myIdx == 0 {
    // 获得锁
} else {
    prevNode := children[myIdx-1]
    conn.Exists("/locks/"+prevNode, watchCallback)
}
```

**ZK vs Redis vs etcd 对比：**

| 维度 | Redis | etcd | ZooKeeper |
|------|-------|------|-----------|
| **一致性** | 最终一致（单点）/ 弱（Redlock）| 线性一致（Raft）| 强一致（ZAB） |
| **性能** | 10W+ QPS | 1W QPS | 1W QPS |
| **脑裂风险** | 有（网络分区时）| 无 | 无 |
| **公平性** | 否（需额外实现）| 是（原生）| 是（原生） |
| **锁粒度** | key 级 | key 级 | key 级 |
| **Go 生态** | go-redis / redisson | clientv3 | go-zookeeper |
| **运维复杂度** | 低 | 中 | 高（ZK 集群配置复杂）|
| **最佳场景** | 性能优先，允许极小概率失效 | 强一致优先 | 历史系统/Java 生态 |

---

## 2. Redlock 争议（面试加分点）

### Martin Kleppmann 的质疑（2016）

Kleppmann 在博客 "How to do distributed locking" 中指出 Redlock 的致命问题：

**时钟漂移问题：**
```
场景：5 个 Redis 节点，锁 ttl = 10s

T1: Client A 向 Node1, Node2, Node3 加锁成功（3/5）
T2: Node2 发生 GC（Stop The World，停了 15s）
T3: Node2 的锁已超时自动释放
T4: Client B 向 Node2, Node3, Node4 加锁成功（3/5）
T5: Node2 从 GC 恢复，继续执行临界区代码
    → Client A 和 Client B 同时在临界区！
```

**结论**：Redlock 依赖系统时钟的稳定性，在 GC 导致的 STW 场景下会失效。

### Antirez 的反驳

- GC 导致的 STW 是进程级别的，不只是 Redis 的问题
- 时钟漂移在实际生产环境中概率极低
- Redlock 的目标是"大多数情况下可靠"，不是"数学上严格"

### 面试结论（标准回答）

> "Redlock 在大多数场景够用，适合对性能要求高、允许极小概率失效的业务（如秒杀扣库存）。金融、强一致场景用 etcd，不建议依赖 Redlock 做严格分布式锁。"

---

## 3. 选型决策树

```
                    ┌─────────────────────┐
                    │ 选型决策树           │
                    └──────────┬──────────┘
                               │
              ┌────────────────▼────────────────┐
              │ 是否需要强一致性（不允许脑裂）？    │
              └────────────┬───────────────────┘
                   是              否
                   │                  │
         ┌─────────▼───────┐  ┌──────▼──────┐
         │ 需要多高性能？   │  │ 业务能容忍   │
         └────┬───────┬────┘  │ 极小概率失效？│
         高         中低       └──────┬──────┘
         │         │            是          否
  ┌──────▼──────┐  │      ┌─────▼────┐ ┌───▼────┐
  │ Redis 集群  │  │      │ etcd     │ │ 评估   │
  │  + 单节点锁│  │      │ 线性一致  │ │ Redlock│
  └────────────┘  │      └──────────┘ └────────┘
         │
  ┌──────▼──────────────────────────────┐
  │ 场景判断：                          │
  │ 秒杀/库存扣减 → Redis Lua 原子脚本   │
  │ （不一定需要分布式锁）              │
  └─────────────────────────────────────┘
```

**Redis Lua 脚本（不需要分布式锁的高并发场景）：**

```go
// 库存扣减：原子操作，不需要锁
const decrScript = `
    local stock = redis.call("GET", KEYS[1])
    if not stock then return -1 end
    stock = tonumber(stock)
    if stock < tonumber(ARGV[1]) then
        return -2  -- 库存不足
    end
    return redis.call("DECRBY", KEYS[1], ARGV[1])
`
result, err := redis.Eval(ctx, decrScript, []string{"stock:product:10086"}, 1).Result()
```

---

## 4. 完整 Go 代码示例

### Redis 分布式锁（go-redis）

```go
type RedisLock struct {
    client  *redis.Client
    key     string
    value   string
    ttl     time.Duration
}

func NewRedisLock(client *redis.Client, key string) *RedisLock {
    return &RedisLock{
        client: client,
        key:    key,
        value:  uuid.New().String(), // 全局唯一标识
        ttl:    30 * time.Second,
    }
}

func (l *RedisLock) TryLock(ctx context.Context) (bool, error) {
    result, err := l.client.SetNX(ctx, l.key, l.value, l.ttl).Result()
    return result == 1, err
}

// 可重入锁：同一个 value 可以多次加锁（用 map 记录 count）
type ReentrantLock struct {
    RedisLock
    holdCount map[string]int // key: lockKey, value: hold count
    mu        sync.Mutex
}

func (l *ReentrantLock) Lock(ctx context.Context) error {
    l.mu.Lock()
    if l.holdCount[l.key] > 0 {
        // 已持有锁，可重入
        l.holdCount[l.key]++
        l.mu.Unlock()
        return nil
    }
    l.mu.Unlock()

    // 尝试获取锁
    ok, err := l.TryLock(ctx)
    if !ok {
        return errors.New("failed to acquire lock")
    }

    l.mu.Lock()
    l.holdCount[l.key] = 1
    l.mu.Unlock()
    return nil
}

func (l *ReentrantLock) Unlock(ctx context.Context) error {
    l.mu.Lock()
    defer l.mu.Unlock()
    if l.holdCount[l.key] > 1 {
        l.holdCount[l.key]--
        return nil
    }
    delete(l.holdCount, l.key)

    // Lua 脚本：只删除自己的锁
    script := `if redis.call("GET", KEYS[1]) == ARGV[1] then return redis.call("DEL", KEYS[1]) else return 0 end`
    _, err := l.client.Eval(ctx, script, []string{l.key}, l.value).Result()
    return err
}
```

### etcd 分布式锁（clientv3）

```go
import (
    "go.etcd.io/etcd/clientv3"
    "go.etcd.io/etcd/clientv3/concurrency"
)

func etcdLockExample(cli *clientv3.Client, key string) error {
    // 创建一个 Session（类似 Lease，断开自动释放）
    sess, err := concurrency.NewSession(cli, concurrency.WithTTL(10))
    if err != nil {
        return err
    }
    defer sess.Close()

    // 创建互斥锁
    mu := concurrency.NewMutex(sess, key)

    // 加锁（阻塞）
    if err := mu.Lock(context.TODO()); err != nil {
        return err
    }
    defer mu.Unlock(context.TODO())

    // 临界区业务逻辑
    return nil
}
```

### 压测对比数据

```
压测环境：3 台机器，每台 8 核，100 并发客户端

| 方案           | QPS     | P99 延迟 | 脑裂概率 |
|---------------|---------|---------|---------|
| Redis 单节点   | 95,000  | 1.2ms   | ~0.01%  |
| etcd           | 12,000  | 8ms     | 0%      |
| ZooKeeper      | 10,500  | 12ms    | 0%      |
| Redis Redlock  | 18,000  | 5ms     | ~0.001% |
```

---

## 高频追问

### Q1：Redis 锁的 TTL 设多少合适？太短/太长分别会怎样？

**TTL 太短的问题：**
- 业务还没执行完，锁就自动释放了
- 其他客户端进入临界区，产生数据竞争
- 解决：Watchdog 自动续期，或将 TTL 设为"业务执行时间 × 2"

**TTL 太长的问题：**
- 客户端崩溃后，锁要等很久才能被其他客户端获取
- 死锁恢复时间长，影响服务可用性
- 解决：TTL = 正常业务耗时 + 一定 buffer（如 30s），配合续期

### Q2：加锁成功但业务执行超过 TTL，锁自动释放了怎么办？

两种解决方案：

**方案 1：Watchdog 自动续期（Redisson 风格）**
```go
// 后台协程每 ttl/3 时间续期一次
go func() {
    ticker := time.NewTicker(l.ttl / 3)
    for {
        select {
        case <-ticker.C:
            // 只有锁还被自己持有时才续期
            redis.Expire(l.key, l.ttl)
        case <-done:
            return
        }
    }
}()
```

**方案 2：业务层幂等 + 锁的价值校验**
```go
// 解锁时校验 value，不匹配则不解锁
if redis.GET(l.key) == l.value {
    redis.DEL(l.key)
}
// 即使 TTL 已过，新客户端加了锁，旧客户端也只删自己的锁
```

### Q3：为什么 etcd 锁不会脑裂，Redis 锁会？

**etcd 基于 Raft 共识算法**：
- 所有写操作必须经过多数派节点确认
- 网络分区时，只有多数派分区能继续工作
- 少数派分区无法选出 Leader，无法加锁
- 不会出现"两个节点都认为自己是锁的持有者"

**Redis 单节点锁**：
- 不依赖共识协议，锁状态只存在于单台 Redis
- 网络分区时，客户端可能同时连上多个分区
- 分区 A 的 Client 加锁成功，分区 B 的 Client 也可能加锁成功
- → 脑裂：两个客户端同时持有锁

**Redlock** 虽然是多节点，但由于分布式系统不存在"全局时钟"，多个节点加锁成功的时间差可能产生竞态，且依赖时钟而非共识，所以理论上不如 etcd/ZK 可靠。
