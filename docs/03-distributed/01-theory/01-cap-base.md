# CAP 定理与 BASE 理论

> 考察频率：★★★★☆  难度：★★★☆☆

## CAP 定理

分布式系统最多只能同时满足以下**两个**特性：

| 特性 | 说明 |
|------|------|
| **Consistency（一致性）** | 所有节点在同一时刻看到相同的数据 |
| **Availability（可用性）** | 每个请求都能在有限时间内得到响应 |
| **Partition Tolerance（分区容错）** | 系统在网络分区时仍能运行 |

**核心结论**：网络分区不可避免，因此 C 和 A 无法同时保证，只能在 C 和 A 之间权衡。

### CAP 三选二？

> 这里的"三选二"是指在发生网络分区时，只能在 C 和 A 之间选择。

```
        CA（不存在）
           ↑
     可用性分区容错
    /                \
   C                C
分区容错            可用性
```

### 实际系统选择

| 系统 | 选 C 还是 A | 典型场景 |
|------|------------|---------|
| Redis Cluster | A（最终一致） | 缓存、分布式锁 |
| Zookeeper | C（强一致） | 配置管理、选举 |
| Cassandra | A（最终一致） | 日志、时序数据 |
| etcd | C（强一致） | 服务发现、配置中心 |
| MongoDB | 可配置 | 文档存储 |

### 追问：CP 和 AP 的实际表现

**CP 系统（如 etcd）**：
- Leader 挂掉后，集群不可用，直到新 Leader 选出
- 保证强一致，但可能拒绝服务

**AP 系统（如 Cassandra）**：
- 任意节点都可响应
- 数据可能暂时不一致，最终会同步

---

## BASE 理论

是对 CAP 中 AP 的延伸，核心思想是**牺牲强一致，保证可用性，最终一致**。

| 缩写 | 全称 | 说明 |
|------|------|------|
| **B**asically **A**vailable | 基本可用 | 允许部分节点故障 |
| **S**oft state | 软状态 | 数据状态可以暂时不一致 |
| **E**ventually consistent | 最终一致 | 经过一段时间后达到一致 |

### BASE vs CAP

| 对比 | CAP | BASE |
|------|-----|------|
| 一致性 | 强一致 | 弱一致 |
| 可用性 | 放弃 P 时不可用 | 始终可用 |
| 追求 | 原子性、隔离性 | 可用性、性能 |
| 典型场景 | 金融、事务 | 互联网业务 |

---

## 分布式系统一致性问题

### 一致性级别

| 级别 | 说明 | 典型系统 |
|------|------|---------|
| 强一致 | 任何时刻都是一致的 | Zookeeper、etcd |
| 线性一致 | 操作的先后顺序是全局确定的 | Google Spanner |
| 顺序一致 | 操作顺序在每个节点一致 | 分布式锁 |
| 因果一致 | 保持因果关系的顺序 | Cassandra |
| 最终一致 | 允许暂时不一致，最终达到一致 | 大多数 NoSQL |

### 实际选型建议

```
读多写少 + 允许短暂不一致 → Redis 主从 / Cassandra
强一致 + 高可用 → etcd / Consul
金融、订单等核心业务 → 2PC / TCC
```

---

## PACELC 模型

CAP 定理的扩展：发生分区（P）时，必须在延迟（Latency）和一致性（C）之间权衡。

```
PA/EL：分区时优先可用 + 低延迟（AP 系统）
PC/EL：分区时优先一致 + 高延迟（CP 系统）
PA/EC：分区时优先可用 + 一致性降级
PC/EC：分区时优先一致 + 延迟可控
```

| 系统 | CAP | PACELC | 典型场景 |
|------|-----|--------|---------|
| Cassandra | AP | PA/EL | 低延迟写入，分区容忍 |
| DynamoDB | AP | PA/EL | 全球分布式 |
| etcd | CP | PC/EC | 强一致配置中心 |
| HBase | CP | PC/EC | 强一致存储 |
| Kafka | CP | PC/EC | 消息可靠 |
| Redis Cluster | AP | PA/EL | 缓存 |

**追问**：PACELC 有什么用？
> CAP 只考虑分区时，但很多系统在**正常情况下**也需要在延迟和一致性间权衡。PACELC 更完整地描述了系统的实际行为。比如 DynamoDB 在无分区时也提供 eventually consistent（低延迟）和 strongly consistent（高延迟）两种读模式。

---

## 实际系统 CAP 深度分析

### etcd（CP + 高可用）

```go
// etcd 使用 Raft 协议，选举期间不可用
// 读请求可以走 Lease 或线性读

// 线性读：保证读取到最新已提交的数据
resp, err := client.Get(ctx, key, clientv3.WithLinearRead())
```

**特点**：
- Leader 选举期间（约几百毫秒）服务不可用
- 读性能好（可本地读），写需 Leader
- 适合配置管理、服务发现等强一致场景

### Cassandra（AP + 可调一致性）

```yaml
# Cassandra 一致性级别可配置
# ONE：最低延迟，单节点确认即可
# QUORUM：多数节点确认，强一致
# ALL：全部节点确认，最强一致

# 可用性：任一节点可写
# 延迟：取决于一致性级别
```

**选型决策**：
```go
// 写强一致场景：写 ALL，读 ONE
// 读强一致场景：写 QUORUM，读 QUORUM
// 低延迟场景：写 ONE，读 ONE（最终一致）
```

### Redis Cluster（AP + 最终一致）

```go
// Redis Cluster 写入流程
// 1. 计算 key 所在槽位
// 2. 转向槽位对应的 Master
// 3. Master 写入后立即返回
// 4. 异步复制到 Slave

// 故障场景：Master 宕机，Slave 晋升需数秒
// 这期间写入会失败（可用性降级）
// 但不会脑裂（最小槽位设计）
```

### Spanner（CP + 高可用）

Google Spanner：TrueTime（GPS + 原子钟）实现线性一致性。
- 写入需 Paxos 多数派确认
- 读取可本地读（快照隔离）
- 跨数据中心延迟 10-15ms

---

## Go 工程中的 CAP 实践

### 注册中心选型

| 组件 | CAP | Go 客户端 | 适用场景 |
|------|-----|---------|---------|--------|
| etcd | CP | clientv3 | K8s、服务发现、配置中心 |
| Consul | CP | consul api | 服务发现、健康检查 |
| Eureka | AP | aws-sdk | Spring Cloud 微服务 |
| Nacos | CP/AP 可配置 | nacos-sdk | 阿里云原生 |

```go
// etcd 强一致读取
cli, _ := clientv3.New(clientv3.Config{
    Endpoints:   []string{"localhost:2379"},
    DialTimeout: 5 * time.Second,
})

// 线性读（强一致）
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
resp, err := cli.Get(ctx, key, clientv3.WithSerializable())
```

### 分布式锁选型

| 锁类型 | CAP | 实现 | 适用场景 |
|--------|-----|------|---------|
| Redis SETNX | AP | Redisson | 性能优先，允许少量锁失效 |
| etcd 分布式锁 | CP | clientv3 concurrency | 强一致，金融级 |
| ZK 分布式锁 | CP | curator | 强一致，选举场景 |

```go
// etcd 分布式锁（强一致）
mutex, _ := concurrency.NewMutex(cli, "/my-lock")
mutex.Lock(ctx)
// 业务逻辑
mutex.Unlock(ctx)
```

---

## 面试话术

**Q：为什么分布式系统不能同时保证 C 和 A？**

> 因为网络分区不可避免（机架断电、交换机故障）。当发生分区时，如果要保证强一致，需要所有节点同步确认后才能响应，这期间服务不可用；如果要保证可用，就需要接受分区期间的数据不一致。所以 CAP 告诉我们：P（分区容错）是必须面对的现实，在此基础上只能选择 C 或 A。

**Q：Redis Cluster 是 AP 还是 CP？**

> Redis Cluster 是 AP 系统。它采用异步复制，Master 写入后立即返回，不等待 Slave 同步。当 Master 故障时，可能丢失少量数据（主从同步有延迟）。但它保证了可用性，任意节点都能响应请求。这里有个细节：如果配置了 `wait` 命令让 Master 等待 Slave 确认，可以提高一致性，但会增加延迟，本质上是在 C 和 L（Latency）之间权衡，符合 PACELC 模型。

**Q：如何设计一个兼顾一致性和可用性的系统？**

> 可以采用**分离读写**的思路：写操作走强一致路径（如 Raft 共识），读操作可以本地读取（可能读到旧数据）。或者采用**分层策略**：核心业务（订单、支付）走 CP，保证数据不错；非核心业务（评论、点赞）走 AP，保证可用。另外可以设置"一致性窗口"，允许在窗口内数据暂时不一致，超时后强制同步。

**Q：CAP 和 BASE 的关系是什么？**

> CAP 描述的是分布式系统在网络分区时的**二选一困境**；BASE 是对这个困境的**工程解法**。CAP 告诉你要么选 C 要么选 A，BASE 告诉你选 A 时怎么通过软状态和最终一致来让系统实际可用。互联网业务 99% 的场景都是 AP + BASE，只有涉及钱的事情才需要 CP。

**Q：什么场景必须用 CP 系统？**

> 三类场景：1）金融交易（订单、支付），数据不一致直接导致资损；2）分布式协调（选主、分布式锁），脑裂会导致严重后果；3）元数据管理（K8s、配置中心），数据错乱会导致全站故障。etcd/Consul/Zookeeper 都是 CP 系统，代价是写入需 Leader 确认，Leader 挂了要等选举。
