# etcd 原理与实战

> 考察频率：★★★★☆  优先级：P1
> 关键词：Raft、MVCC、Watch、Lease、分布式锁、选主、boltdb

## 面试官考察意图

这道题考察候选人对 **etcd 的理解深度**，不只是会用 API，而是要理解它为什么这样设计。高级工程师的回答会涵盖：
- etcd 如何在有限硬件上实现强一致性（CAP 取舍）
- Watch 机制底层是 gRPC stream + MVCC 组合拳，缺一不可
- 分布式锁依赖 Lease + Revision 实现公平性，这是 etcd 的独门绝技
- 选主不是简单的多数派投票，有脑裂风险需要防范

初级工程师往往只能说出"基于 Raft 协议"，然后就没了。

---

## 核心答案（30 秒版）

**etcd = Raft 一致性协议 + boltdb 持久化存储 + MVCC 多版本控制 + gRPC Watch 推送。**

核心能力一句话：
- **服务注册/配置存储**：Lease 租约 + key 的 TTL 自动过期
- **分布式锁**：用 Revision 大小保证公平排队
- **选主**：基于 Raft Leader 选举，Go 标准库 `clientv3` 直接支持
- **Watch 变更推送**：借助 MVCC 做到增量推送，不丢消息

---

## 深度展开

### 1. 整体架构

```
                    ┌─────────────────────────────────────────┐
                    │              etcd 集群                  │
                    │                                          │
  Client ──gRPC──▶ │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
  (Go/Java/Python) │  │  Node1  │  │  Node2  │  │  Node3  │  │
                    │  │ (Leader)│◀─│(Follower│◀─│(Follower│  │
                    │  └────┬────┘  └────┬────┘  └────┬────┘  │
                    │       │            │            │       │
                    │       ▼            ▼            ▼       │
                    │   ┌─────────────────────────────────┐   │
                    │   │         boltdb 存储引擎          │   │
                    │   │   (WAL + Snapshots + MVCC)      │   │
                    │   └─────────────────────────────────┘   │
                    └─────────────────────────────────────────┘
```

**三层职责：**
| 层级 | 组件 | 职责 |
|------|------|------|
| 网络层 | gRPC + Raft | 节点间通信、Leader 选举、日志复制 |
| 逻辑层 | MVCC + Lease | 多版本并发控制、租约管理、Revision 生成 |
| 存储层 | boltdb | key-value 持久化存储，支持 MVCC |

---

### 2. Raft 选主与日志复制

#### Leader 选举

```
时间线：
─────────────────────────────────────────────────────────▶

[ Node A (Candidate) ]  ── 发起选举 ──▶  收到多数派投票 ──▶  成为 Leader
                                  │              │
                                  │              ▼
                                  │        term=3, commitIndex=100
                                  ▼
                           选主超时 (150~300ms)
                           随机 jitter 避免平票
```

**选举流程：**
1. Follower 超过 election timeout（随机 150~300ms）未收到 Leader 心跳 → 转为 Candidate
2. Candidate 自增 term，给自己投一票
3. 向其他节点发 `RequestVote`，要求投票
4. 节点比较 term，term 更大或相同时 log 更长则投票
5. 获得多数派票数 → 成为 Leader，开始心跳

**防止脑裂的关键：**
- 一个任期 (term) 最多一个 Leader
- 节点不认可 term 更小的 Leader 发来的消息
- 网络分区时，旧 Leader 无法获得多数派承认，自动降级

#### 日志复制

```
Client 请求： PUT key="name" value="bincai"
        │
        ▼
   Leader 写入本地 WAL
        │
        ▼
   并行发送给所有 Follower（AppendEntries RPC）
        │
        ├── Follower1 ──写入WAL──▶ 回复 OK
        ├── Follower2 ──写入WAL──▶ 回复 OK
        └── Follower3 ──超时/失败──▶ 重试
        │
        ▼
   收到多数派确认（包括自己）──▶ commit
        │
        ▼
   异步通知 Follower commit 位置
   回复 Client 成功
```

**关键点：**
- 日志必须落 WAL 才能被确认，宕机可恢复
- `commitIndex` 由 Leader 推进，不是 Follower 主动推进
- 线性一致性：Leader 确认前必须等待多数派持久化

---

### 3. boltdb 存储引擎

etcd 的存储引擎是 **boltdb**（BoltDB），它是 LMDB 的 Go 实现，具备以下特性：

#### MVCC 实现原理

```go
// boltdb 每个 key 存储多个版本，格式如下：
// key + revision → value
//
// revision 结构：
// main: 每次事务递增
// sub:  事务内的操作序号
//
// key="name" at revision=3:2 → value="bincai"
// key="name" at revision=5:0 → value="xiaogao"  (更新后旧版本依然保留)
//
// compaction 时删除旧版本
```

**MVCC 的好处：**
- 读不阻塞写，写不阻塞读（快照读）
- Watch 可以精确获取变更版本，不丢消息
- 历史版本可查询（`WithRev` 参数）

#### 存储结构

```
boltdb page 结构：
┌──────────┬──────────┬──────────┐
│  meta    │  freelist │   data   │
│  page    │   page    │  pages   │
└──────────┴──────────┴──────────┘

data pages 组织为 B+ 树：
- 叶子节点：key-value 对（按 bucket 分组）
- 非叶子节点：索引指针

WAL（Write-Ahead Log）：
每次写操作先追加 WAL，再写 boltdb
宕机后通过 WAL 重放恢复
```

---

### 4. Watch 机制（核心亮点）

这是面试中最容易挖深的点。

#### 原理：gRPC Stream + MVCC 版本号

```go
// 客户端发起 Watch 请求
watcher := client.Watch(ctx, "service/order/", clientv3.WithPrefix())

// etcd 内部处理流程：
// 1. 创建 watcher 对象，记录 startRev（从哪个版本开始看）
// 2. 把 watcher 挂到对应的 MVCC 桶上
// 3. 每次 kv 变更，MVCC 递增 revision
// 4. 推送协程扫描变化的 revision，发送给客户端

for {
    select {
    case wresp := <-watcher:
        for _, ev := range wresp.Events {
            // ev.Type: PUT / DELETE
            // ev.Kv.Key, ev.Kv.Value
            // ev.Kv.ModRevision: 本次修改的版本号
            fmt.Printf("%s %s @ rev=%d\n", ev.Type, ev.Kv.Key, ev.Kv.ModRevision)
        }
    }
}
```

**为什么 Watch 不丢消息？**

这是 MVCC 的精妙之处：
- 每次 Put/Delete 都递增全局 revision（单调递增）
- Watch 请求带上 `startRev`，告诉 etcd"我要从第 N 个版本开始看"
- 即使客户端重启，从上次已知 revision 重新 Watch，依然能拿到中间的变更

**与 Redis Pub/Sub 的区别：**

| 维度 | etcd Watch | Redis Pub/Sub |
|------|-----------|---------------|
| 可靠性 | ✅ MVCC 保证，可重放 | ❌ 纯内存，断连丢失 |
| 增量 | ✅ 只推变更 | ❌ 全量推送 |
| 历史 | ✅ 支持从任意版本开始 | ❌ 无历史 |
| 复杂度 | 高（依赖 MVCC） | 低（纯内存） |

#### 生产经验：Watch 丢消息的坑

```go
// ❌ 错误：直接在主循环里 Watch，没有处理重连
for {
    rch := client.Watch(ctx, key)
    for wresp := range rch {
        // 处理消息...
    }
    // 如果 channel 关闭，说明连接断了，需要重连
    time.Sleep(time.Second) // 避免疯狂重试
}

// ✅ 正确：处理 watch channel 关闭，自动重连
func watchWithReconnect(cli *clientv3.Client, key string) {
    for {
        rch := cli.Watch(ctx, key)
        for wresp := range rch {
            if wresp.Err() != nil {
                log.Printf("Watch error: %v", wresp.Err())
                break
            }
            for _, ev := range wresp.Events {
                handleEvent(ev)
            }
        }
        // channel 关闭，重连
        time.Sleep(100 * time.Millisecond)
    }
}
```

---

### 5. Lease 租约机制

**核心概念：** Lease 是一段时间（TTL），关联到一组 key，过期后所有 key 自动删除。

```go
// 创建租约，TTL=10秒
leaseResp, err := cli.Grant(ctx, 10)
leaseID := leaseResp.ID

// key 关联租约
cli.Put(ctx, "config/feature-flags", "enabled", clientv3.WithLease(leaseID))

// 自动续约
ch, err := cli.KeepAlive(ctx, leaseID)
go func() {
    for {
        select {
        case resp := <-ch:
            if resp == nil {
                fmt.Println("租约已过期，需要重新注册")
                return
            }
            fmt.Printf("续约成功，TTL=%d\n", resp.TTL)
        }
    }
}()

// 撤销租约（提前删除所有关联 key）
cli.Revoke(ctx, leaseID)
```

**典型使用场景：**
1. **服务注册**：心跳续约，实例下线自动注销
2. **配置管理**：配置有过期时间，防止配置挂起
3. **分布式锁**：锁依赖租约防止死锁

---

### 6. 分布式锁（基于 Lease + Revision）

这是 etcd 最有价值的功能之一，**Redlock 的基础**。

```go
// Go + etcd 分布式锁完整实现
import (
    "context"
    "fmt"
    "time"

    "go.etcd.io/etcd/client/v3"
    "go.etcd.io/etcd/client/v3/concurrency"
)

func distributedLockExample(cli *clientv3.Client) {
    // 创建 session（自动持有租约，退出时自动释放）
    session, err := concurrency.NewSession(cli, concurrency.WithTTL(30))
    if err != nil {
        panic(err)
    }
    defer session.Close()

    // 创建互斥锁
    mu := concurrency.NewMutex(session, "my-lock")

    ctx := context.Background()

    // 加锁（阻塞等待）
    if err := mu.Lock(ctx); err != nil {
        panic(err)
    }
    fmt.Println("获得锁，执行临界区操作")
    // ... 业务逻辑 ...

    // 解锁
    if err := mu.Unlock(ctx); err != nil {
        panic(err)
    }
    fmt.Println("释放锁")
}

// 可重入锁（同一进程多次获得锁）
func reentrantLock(cli *clientv3.Client) {
    session, _ := concurrency.NewSession(cli)
    defer session.Close()

    mu := concurrency.NewMutex(session, "reentrant-lock")

    // 第一次加锁
    mu.Lock(context.Background())
    // 第二次加锁（同一个 keyRev，可以重入）
    err := mu.TryLock(context.Background())
    if err != nil {
        fmt.Println("无法重入：已被其他持有")
    }
    defer mu.Unlock(context.Background())
}
```

**公平性实现原理：**
```
客户端列表（按 revision 排序）：
Revision=5: Client A（最早请求）
Revision=6: Client B
Revision=7: Client C

获得锁的条件：
1. 没有比当前客户端更小 revision 的锁持有者
2. 自己是当前 revision 中最小的（或唯一）
```

**与 Redis 分布式锁对比：**

| 维度 | etcd 锁 | Redis 锁 |
|------|--------|---------|
| 公平性 | ✅ revision 自然排序，天然公平 | ⚠️ 需要 Lua 脚本模拟 |
| TTL 安全 | ✅ 自动续期 (KeepAlive) | ⚠️ 需要 watchdog |
| 性能 | ~1000 QPS/节点 | ~10万 QPS/节点 |
| 一致性 | CP（强一致） | AP（主从异步） |
| Redlock | ✅ 原生支持 | ⚠️ 争议较大 |

---

### 7. 选主（Leader Election）

```go
// 使用 etcd 的 election 实现选主
func leaderElectionExample(cli *clientv3.Client) {
    session, _ := concurrency.NewSession(cli)
    defer session.Close()

    election := concurrency.NewElection(session, "my-election")

    ctx := context.Background()

    // 注册为候选人（发起选举）
    err := election.Campaign(ctx, "candidate-1")
    if err != nil {
        panic(err)
    }

    // 当前是否是 Leader
    resp, err := election.Leader(ctx)
    if err != nil {
        fmt.Println("当前无 Leader")
    } else {
        fmt.Printf("当前 Leader: %s, term=%d\n", string(resp.Kvs[0].Value), resp.Header.Revision)
    }

    // 监控 Leader 变化
    go func() {
        for {
            ch := election.Observe(ctx)
            for resp := range ch {
                fmt.Printf("Leader 变更: %s\n", string(resp.Kvs[0].Value))
            }
            time.Sleep(time.Second)
        }
    }()

    // 退出选举（主动让贤）
    election.Resign(ctx)
}
```

---

### 8. 性能与局限

**etcd 性能上限（3 节点集群）：**
- 写 QPS：约 **1,000~3,000**（强一致，需要 Raft 多数派确认）
- 读 QPS：约 **10,000~30,000**（线性读），**100,000+**（快照读）
- 存储：单 key 不超过 **1MB**（建议 10KB 以内）

**为什么不适合做消息队列？**
1. 写路径：每次写 → WAL → Raft 复制 → boltdb 写入 → 通知客户端，链路太长
2. 容量有限：boltdb 是 KV 存储，不是 LSM-tree，存储成本高
3. Watch 扩展性差：万个 Watcher 同时监控一个 key，etcd 会成为瓶颈
4. 正确的选择：Kafka / RocketMQ 处理消息，etcd 存储配置和选主

---

### 9. 与 Zookeeper 对比

| 维度 | etcd | Zookeeper |
|------|------|----------|
| 一致性模型 | Raft（强一致）| ZAB（类 Paxos）|
| API | gRPC + JSON（易用）| 自定义协议（复杂）|
| 语言生态 | Go 官方 client，生态好 | Java 为主 |
| 健康检查 | Lease TTL | 临时节点 + 心跳 |
| 分布式锁 | 原生 NewMutex | 需要 Curator |
| 运维复杂度 | 低（单二进制）| 高（JVM + ZooKeeper）|
| K8s 集成 | 原生（k8s 使用 etcd）| 需要额外集成 |
| 适用场景 | 配置中心、服务注册、选主 | 传统 Java 微服务 |

---

## 高频追问

### Q1: etcd 的 lease 过期了，但业务还没处理完怎么办？

**答：** Lease 本身不提供续期之外的延长能力。正确做法是：
1. **主动续期（KeepAlive）**：在业务处理周期内持续调用 KeepAlive，不让租约过期
2. **设计与业务分离**：把 lease TTL 设为业务容忍的最大处理时间 + 缓冲
3. **Redlock 模式**：用 etcd 锁的 TTL 替代纯 lease，锁自动保证不会无限持有

```go
// 业务处理中持续续期
ctx := context.Background()
for {
    select {
    case <-time.After(3 * time.Second):
        // 每 3 秒续一次（TTL=10秒，有足够缓冲）
        cli.KeepAliveOnce(ctx, leaseID)
    case <-业务完成信号:
        // 业务完成，释放锁
        mu.Unlock(ctx)
        return
    }
}
```

### Q2: 怎么防止 etcd 选主时的脑裂？

**答：** etcd 的 Raft 实现天然防止脑裂：
- 一个 term 最多一个 Leader
- Leader 必须获得多数派（>50%）确认才能工作
- 网络分区时，少数派节点无法获得多数派，无法当选 Leader
- 旧 Leader 在新 term 下自动降级，不接受写请求

### Q3: etcd 读请求能否线性一致？

**答：** 默认情况下，etcd 支持两种读：
1. **串行化读（Default）**：Leader 直接读本地状态，可能读到旧值（不是线性一致）
2. **线性读（Serializable）**：通过 Leader 拿到 `revision`，读指定版本的数据，保证线性一致

```go
// 线性读（Serializable）
resp, err := cli.Get(ctx, key, clientv3.WithSerializable())
// 或
resp, err := cli.Get(ctx, key, clientv3.WithRev(leaderRev))
```

### Q4: etcd 挂了 2 台节点，集群还能用吗？

**答：** 取决于节点数和存活节点数：
- **3 节点**：挂 1 台 → 2 台存活 → **可用**（多数派 2/3）
- **3 节点**：挂 2 台 → 1 台存活 → **不可用**（无法形成多数派）
- **5 节点**：挂 2 台 → 3 台存活 → **可用**（多数派 3/5）
- 建议生产环境使用 **5 节点**或 **3 节点+仲裁**

---

## 延伸阅读

| 资料 | 链接 |
|------|------|
| etcd 官方文档 | https://etcd.io/docs/ |
| etcd clientv3 Go SDK | https://github.com/etcd-io/clientv3 |
| Raft 论文 | https://raft.github.io/raft.pdf |
| boltdb 存储原理 | https://github.com/etcd-io/bbolt |
| TiDB 对比 etcd | https://pingcap.com/blog/tidb-vs-etcd |
