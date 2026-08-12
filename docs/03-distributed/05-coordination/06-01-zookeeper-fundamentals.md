# ZooKeeper 原理与实战：选举、会话、分布式协调

> 考察频率：★★★★☆  优先级：P1
> 关键词：ZAB 协议、Leader/Follower/Observed、Session、EPHEMERAL 节点、Watch 机制、与 etcd 对比

---

## 面试官考察意图

这道题考察候选人对 **ZooKeeper 底层原理**的理解。初级只会用 API（创建临时节点拿分布式锁），高级要能讲清楚 **ZAB 协议的两种模式、Session 的心跳保活机制、EPHEMERAL 节点与 Watch 的组合拳如何构成 ZK 的核心能力、以及为什么 ZK 强一致而 etcd 也强一致却适合不同的场景**。这是"大厂必问，但很多人答不深"的经典题。

---

## 核心答案（30 秒版）

**ZooKeeper = ZAB 一致性协议 + 内存数据结构 + ZNode 层级命名空间。**

三大核心能力：
- **分布式锁：** EPHEMERAL 临时节点保证客户端宕机自动释放
- **Leader 选举：** ZAB 协议 Two-Phase Commit，过半存活即可选主
- **配置同步：** WATCHER 事件驱动，变更实时推送

**ZK vs etcd：**
| 维度 | ZooKeeper | etcd |
|------|-----------|------|
| 一致性协议 | ZAB（类 Paxos） | Raft |
| 数据模型 | 树形 ZNode | KV store |
| Watch | 一次性触发（需反复注册） | 持续监听（Revision 增量推送） |
| 适用场景 | Hadoop 生态、大规模集群协调 | K8s、轻量配置中心、服务发现 |
| Go 原生支持 | ⚠️ 官方库已归档，社区维护 | ✅ go.etcd.io/etcd/client/v3 |

---

## 深度展开

### 一、ZAB 协议详解

ZAB（Zookeeper Atomic Broadcast）是 ZooKeeper 的**原子消息广播协议**，包含两种模式：

#### 1. 恢复模式（Recovery Phase）→ Leader 选举

```
1. 每个节点启动时进入 LOOKING 状态，认为自己是 Leader
2. 发送 ElectionPacket（包含 selfId、zxid、electionEpoch）
3. 收到其他节点的包后比较：
   - zxid 大的优先（事务序号越大越新）
   - zxid 相同时，selfId 大的优先（服务器 ID 做决胜局）
4. 统计选票，得票过半的成为 Leader
5. Leader 选出后，所有 Follower 切换到 FOLLOWING 状态
```

**关键点：**
- **FOLLOWING > OBSERVING > LOOKING** 状态优先级，一旦看到 FOLLOWING 就停止竞争
- Zxid（ZooKeeper Transaction Id）是全局单调递增的事务编号，用来判断谁是最新数据

#### 2. 广播模式（Broadcast Phase）→ 数据更新

Leader 将更新提案分发给所有 Follower：

```
Client → Proposal(P) → Leader
                        │
                   ┌────┴────┐
                   ▼         ▼
              Follower 1  Follower 2  ... Follower N
                   │         │              │
                   ▼         ▼              ▼
                ACK        ACK            ACK
                   │         │              │
                   └────┬────┘──────────────┘
                        ▼
                  过半 ACK 达成 → Commit → Notify 所有节点
```

**ZAB 保证了什么？**
- **原子性：** 一个 proposal 要么在所有节点上提交，要么都不提交
- **顺序性：** 全局单线程执行，所有节点看到的修改顺序一致
- **持久性：** commit 后立即落盘

---

### 二、ZNode 数据结构

ZooKeeper 的数据结构是**树形命名空间**，每个节点叫 ZNode：

```
/zk/                    ← root
├── /zk/config          ← PERSISTENT（永久节点，重启仍在）
│   ├── /zk/config/flag1
│   └── /zk/config/flag2
├── /zk/leader          ← EPHEMERAL（临时节点，断开自动删除）
├── /zk/service/user    ← PERSISTENT_SEQUENTIAL（永久有序节点）
│   ├── user_0000000001
│   ├── user_0000000002
│   └── user_0000000003
└── /zk/watcher-topic   ← EPHEMERAL_SEQUENTIAL（临时有序节点）
    ├── topic_0000000004
    └── topic_0000000005
```

| 节点类型 | 持久性 | 有序性 | 典型用途 |
|---------|--------|--------|---------|
| **PERSISTENT** | 重启后保留 | ❌ | 配置存储 |
| **EPHEMERAL** | 客户端断开自动删除 | ❌ | 分布式锁、服务实例注册 |
| **PERSISTENT_SEQUENTIAL** | 重启后保留 | ✅ 按序号递增 | 有序队列、分布式 ID |
| **EPHEMERAL_SEQUENTIAL** | 客户端断开自动删除 | ✅ 按序号递增 | Leader 选举（最小号当选） |

**每个 ZNode 包含的数据：**
```go
// ZNode 元数据
struct Stat {
    int64  ctime     // 创建时间
    int64  mtime     // 修改时间
    int32  version   // 数据版本号
    int32  cversion  // 子节点版本变化数
    int32  aversion  // ACL 变化次数
    int64  ephemeralOwner  // EPHEMERAL 节点的所有者 sessionId
}
```

---

### 三、分布式锁实现

```go
// 基于 ZK 临时有序节点实现公平分布式锁
func acquireLock(zkConn *zk.Conn, path string) error {
    // 1. 创建临时有序节点
    lockPath, err := zkConn.Create(
        path+"/lock-",   // parent
        []byte{},        // data
        zk.FlagEphemeral|zk.FlagSequence,  // 标志位
        zk.WorldACL(zk.PermAll),
    )
    if err != nil { return err }
    
    // 2. 获取父目录下所有子节点并按序号排序
    children, _, err := zkConn.Children(path)
    if err != nil { return err }
    
    // 3. 判断自己是不是最小的节点（获得锁）
    sort.Strings(children)
    mySuffix := strings.Replace(lockPath, path+"/", "", 1)
    myIndex := sort.SearchStrings(children, mySuffix)
    
    if myIndex == 0 {
        // 我是最小的，拿到锁！
        fmt.Println("Acquired lock:", lockPath)
        return nil
    }
    
    // 4. 等待前一个节点（排前面的同学解锁通知）
    prev := children[myIndex-1]
    done := make(chan struct{})
    zkConn.AddWatch(path+"/"+prev, zk.EventNodeDeleted)  // 监听前一个节点
    
    <-done  // 阻塞等待前一个节点被删除（锁释放）
    return acquireLock(zkConn, path)  // 递归重试
}
```

**流程图解：**
```
Client A 创建 /lock-A0000000001  → 是最小的，获得锁 ✓
Client B 创建 /lock-B0000000002  → 不是最小，等待 A0000000001
Client C 创建 /lock-C0000000003  → 不是最小，等待 B0000000002

A 完成操作，删除节点 →
  B 的 Watch 收到 EventNodeDeleted → B 获得锁
C 的 Watch 收到...（等 B 完成后再轮到自己）

✅ 公平性：严格按创建顺序获取锁
```

---

### 四、Watch 机制

```
Client A ──▶ Register Watch on /config ──▶ ZK Server
                                        │
                                   Client B 修改 /config
                                        │
                                  ZK Server ──▶ Trigger Watch Event ──▶ Client A
                                              │                            │
                                          Version +1                     读到新数据
```

**ZK Watch 的关键特征：**

| 特性 | 说明 |
|------|------|
| **一次性触发** | 每次 Watch 只触发一次，读完后需要重新注册 |
| **服务端推送** | 变更发生时主动推送给 Watcher，不需要轮询 |
| **顺序保证** | Watch 事件的发送顺序和事务执行顺序一致 |
| **最终一致性** | 不同客户端收到 Watch 事件的顺序可能不同（网络延迟）|

**Go 端使用示例：**
```go
ch := make(chan zk.Event, 1)
conn.RegisterWatch("/config", ch)

for {
    event := <-ch
    if event.Type == zk.EventNodeDataChanged {
        data, stat, _ := conn.Get("/config")
        fmt.Println("Config changed:", string(data))
        
        // Watch 是一次性的，必须重新注册！
        conn.RegisterWatch("/config", ch)
    }
}
```

**常见陷阱：**
- 漏注册导致错过后续变更（最常见的线上问题）
- Watch 回调中阻塞太久导致积压（应改为异步处理）
- 并发修改频繁时 Watch 风暴（应加本地缓存+防抖）

---

### 五、Session 机制

ZooKeeper 客户端与服务端之间有一个 **Session** 生命周期：

```
Session 建立
    │
    ▼
┌────────────┐    超时未续约    ┌─────────────┐
│ Active     │ ──────────────▶ │ Session     │
│ （心跳正常）│                 │ Expired     │
│            │ ◀────────────── │ （自动销毁） │
│            │    立即关闭      └─────────────┘
└────────────┘
    ▲
    │ 定期心跳（默认 40s timeout / sessionTimeoutMs / 2）
```

**Session 参数：**
```go
dialTimeout := 5 * time.Second     // 连接超时
sessionTimeout := 40 * time.Second // 会话超时（单位毫秒）

conn, _, err := zk.Connect(servers, sessionTimeout, zk.WithDialTimeout(dialTimeout))
```

- 客户端每隔 `sessionTimeout / 2` 发送心跳
- 超过 `sessionTimeout` 没收到心跳 → Server 认为 Session 过期
- 所有 EPHEMERAL 节点在 Session 过期时自动删除

**注意：** 实际心跳间隔可以在客户端手动配置，推荐设为 sessionTimeout 的一半以下。

---

### 六、ZK vs etcd 深度对比

| 维度 | ZooKeeper | etcd |
|------|-----------|------|
| 一致性协议 | ZAB（Two-Phase Commit） | Raft（Log Replication） |
| 数据模型 | 树形 ZNode（有子节点概念） | 扁平 KV store |
| Watch 模型 | 一次性触发，需重复注册 | 持续监听，Revision 差分推送 |
| 性能写 | 中等（ZAB 两阶段开销大） | 高（Raft 日志追加快） |
| 性能读 | 纯内存读取，极快 | 纯内存读取，极快 |
| 容量上限 | 百万级 ZNode，GB 级数据 | 百万级 key，GB 级数据 |
| Go SDK 状态 | 官方 zk/go-zk 已归档 | go.etcd.io/etcd/client/v3 活跃维护 |
| 主要使用者 | Netflix、Uber、LinkedIn | CoreOS、Kubernetes、Cloudflare |
| 运维复杂度 | 高（需要经验） | 低（自带运维工具） |
| 脑裂处理 | Observer 角色不参与选举，避免分裂投票 | Raft 天然无脑裂 |

**选型建议：**
- **Hadoop 生态项目**（Kafka、HBase、Storm）→ ZooKeeper（生态绑定）
- **K8s 环境或新项目** → etcd（Go 原生、API 更简洁、Watch 更好用）
- **只需简单分布式锁** → Redis 锁足够，无需引入重型协调服务
- **需要配置热更新** → etcd Watch 体验远优于 ZK

---

## 面试话术模板

> "我们用 etcd 做服务发现和配置管理，因为它比 ZK 的 Watch 更友好——ZK 是一次性的，每次读完要重新注册；etcd 用 Revision 增量推送，可以一直监听着。但如果做 Leader 选举，ZK 的 EPHEMERAL_SEQUENTIAL 天然适合。本质上两者都是 CP 系统，ZAB 和 Raft 都保证过半可用。"

---

📌 **扩展阅读：**
- Apache Curator（ZK 客户端的最佳实践封装）
- Raft 论文 vs ZAB 论文对比
- ZK Java Client 源码中的 Watch 注册逻辑
