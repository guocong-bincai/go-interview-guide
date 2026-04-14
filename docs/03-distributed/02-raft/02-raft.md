# Raft 协议

> 考察频率：★★★★★  难度：★★★★☆

## 概述

Raft 是分布式一致性协议，通过 Leader 选举和日志复制实现容错。

### 与 Paxos 的区别

| 对比 | Raft | Paxos |
|------|------|-------|
| 可理解性 | 更容易理解 | 难以实现 |
| Leader | 强 Leader | 弱 Leader |
| 成员变更 | 支持 | 难以实现 |
| 工程实现 | 大量使用 | 理论为主 |

Raft 把一致性分解为三个子问题：
1. **Leader 选举**
2. **日志复制**
3. **成员变更**

---

## Leader 选举

### 角色

| 角色 | 说明 |
|------|------|
| **Leader** | 接收客户端请求，负责日志复制 |
| **Follower** | 被动响应请求，参与投票 |
| **Candidate** | 发起选举的 Follower |

### 任期（Term）

- 任期是递增的整数
- 每个任期最多一个 Leader
- 节点通过 Term 判断消息新旧

### 选举过程

```
Follower 超时 → 变成 Candidate → 发起选举 → 获得多数票 → 成为 Leader
```

1. **初始状态**：所有节点都是 Follower
2. **选举超时**：Follower 在 `150-300ms` 随机超时后变为 Candidate
3. **发起投票**：Candidate 自增 Term，给自己投票，向其他节点请求投票
4. **投票规则**：
   - 先到先得（每个 Term 一票）
   - 投票给 Term 不小于自己的，且日志比自己新的
5. **选举结果**：
   - 获得多数票 → 成为 Leader
   - 收到其他 Leader 的心跳 → 变回 Follower
   - 选举超时 → 重新发起选举

### 追问：为什么随机超时？

> 避免多个 Candidate 同时发起选举导致票数分散。随机化使得大多数情况下只有一个节点先超时并赢得选举。

---

## 日志复制

### 日志结构

```
index: 1    2    3    4    5
term:  1    1    1    2    2
cmd:   PUT  GET  PUT  SET  DEL
--------------------------------
Log entries
```

每个日志条目包含：
- **index**：日志索引
- **term**：任期号
- **command**：操作命令

### 复制流程

```
Client → Leader → AppendEntries RPC → 复制到多数节点 → Commit → 响应 Client
```

1. Leader 接收客户端请求
2. Leader 将命令写入本地日志
3. Leader 并发发送 `AppendEntries RPC` 给所有 Follower
4. Follower 写入日志后回复成功
5. Leader 收到**多数派**成功响应后，提交（Commit）日志
6. Leader 通知 Follower 已提交
7. Leader 响应客户端

### 一致性保证

Raft 通过 `AppendEntries` 的一致性检查保证：
- Leader 和 Follower 的日志在相同 index 的 term 相同
- 之前的日志也都相同

```
如果 Follower 日志不一致：
  Leader 从最新日志往前找，发送 PrevLogIndex/PrevLogTerm
  Follower 验证通过才接收，否则拒绝
```

### 日志压缩（Snapshot）

日志会无限增长，需要压缩：
- 定期生成 Snapshot（快照）
- 快照包含：最后提交的 index、term、状态数据
- 快照之前的日志可以丢弃

---

## 成员变更

### 联合共识（Joint Consensus）

当添加/移除节点时，可能出现双 Leader 脑裂。

Raft 使用**两阶段变更**：

1. **Joint Configuration**：新旧配置共存
   - 收到成员变更请求后，Leader 写入 `C_old,new`
   - 所有操作需要新旧配置都多数同意
2. **切换到新配置**：完成后写入 `C_new`

### 单节点变更（Simplified）

每次只添加或移除一个节点，避免联合共识的复杂性。

---

## Go 实现要点

### 核心数据结构

```go
type State int
const (
    Follower State = iota
    Candidate
    Leader
)

type LogEntry struct {
    Term    int
    Index   int
    Command interface{}
}

type Raft struct {
    mu        sync.Mutex
    peers     []string
    me        int
    state     State
    currentTerm int
    votedFor   int
    log        []LogEntry

    // 持久化需要：lastIncludedIndex, lastIncludedTerm
    // Volatile:
    commitIndex int
    lastApplied int

    // Leader 专用
    nextIndex  []int
    matchIndex []int

    // Channel
    applyCh    chan ApplyMsg
    heartbeat  time.Duration
    electionTimeout time.Duration
}
```

### RPC 要点

```go
// RequestVote RPC
type RequestVoteArgs struct {
    Term         int
    CandidateId  int
    LastLogIndex int
    LastLogTerm  int
}

// AppendEntries RPC
type AppendEntriesArgs struct {
    Term         int
    LeaderId     int
    PrevLogIndex int
    PrevLogTerm  int
    Entries      []LogEntry
    LeaderCommit int
}
```

### Leader 发送心跳

```go
func (rf *Raft) broadcastHeartbeat() {
    for i := range rf.peers {
        if i == rf.me {
            continue
        }
        go func(peer int) {
            args := &AppendEntriesArgs{
                Term:         rf.currentTerm,
                LeaderId:     rf.me,
                PrevLogIndex: rf.getLastLogIndex(),
                PrevLogTerm:  rf.getLastLogTerm(),
                Entries:       []LogEntry{},
                LeaderCommit: rf.commitIndex,
            }
            var reply AppendEntriesReply
            if rf.sendAppendEntries(peer, args, &reply) {
                // 处理回复
            }
        }(i)
    }
}
```

---

## 常见面试问题

### Q：Raft 如何保证线性一致性？

> 通过 Leader 的全序广播实现。所有写请求必须经过 Leader，Leader 在提交日志前确保之前所有日志都已提交。这样所有节点看到的操作顺序是一致的。

### Q：Leader 挂了怎么办？

> Follower 在 election timeout 后发起选举。如果之前的 Leader 恢复，会发现 term 更小，主动降为 Follower，新 Leader 会重置它的日志。

### Q：如何保证不丢数据？

> 日志必须复制到**多数派**才算提交。即使 Leader 挂了，之前的日志一定存在于至少一个多数派节点中，新 Leader 一定包含最新的日志。

### Q：Raft 和 etcd 的关系？

> etcd 是 Raft 的工程实现，用于分布式键值存储。Kubernetes 使用 etcd 作为存储。
