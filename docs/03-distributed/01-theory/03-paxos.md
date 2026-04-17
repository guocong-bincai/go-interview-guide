# Paxos 算法

> 考察频率：★★★☆☆  难度：★★★★★
> 关键词：半数机制、Proposer、Acceptor、Learner、两阶段提交

## 概述

Paxos 是 Leslie Lamport 提出的**分布式共识算法**，用于在分布式系统中对某个值达成一致。Raft 是 Paxos 的简化与工程化版本。

### 核心角色

| 角色 | 职责 |
|------|------|
| **Proposer** | 提出提案（值），发起投票 |
| **Acceptor** | 接收提案，投票决定是否接受 |
| **Learner** | 学习被选定的值，通知客户端 |

> **注意**：每个节点可以同时担任多个角色。

---

## 算法流程

Paxos 分为**两个阶段**，类似两阶段提交：

### 第一阶段：Prepare（准备）

```
Proposer 选择一个提案编号 N，向半数以上的 Acceptor 发送 Prepare(N) 请求
```

**Acceptor 收到 Prepare(N) 后**：
- 如果 `N > 任何已响应的 Prepare 请求编号`：承诺不再接受编号小于 N 的提案，回复 Promise
- 如果 `N <= 已响应的某个 Prepare 请求编号`：忽略该请求

### 第二阶段：Accept（接受）

```
Proposer 收到多数 Acceptor 的 Promise 后：
  - 如果有 Proposer 已接受的值 → 选择其中编号最大的值作为提案值
  - 如果没有 → 自由选择提案值
  → 向所有 Acceptor 发送 Accept(N, Value)
```

**Acceptor 收到 Accept(N, Value) 后**：
- 如果未承诺不接收（即未响应过编号 > N 的 Prepare）：接受该提案，回复 Accepted
- 否则忽略

### 达成共识

当 Proposer 收到**半数以上**的 Accepted 回复时，提案被选定（Chosen）。

---

## Go 代码实现（简化版）

```go
package paxos

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

// Proposal 代表一个提案
type Proposal struct {
	Number int
	Value  string
}

// Acceptor Paxos 中的接受者
type Acceptor struct {
	mu            sync.Mutex
	id            int
	promisedNum   int              // 已承诺的最大提案编号
	acceptedNum   int              // 已接受的提案编号
	acceptedValue string           // 已接受的提案值
}

// Promise 承诺不再接受编号小于 n 的提案
func (a *Acceptor) Prepare(n int) (promised bool, acceptedNum int, acceptedValue string) {
	a.mu.Lock()
	defer a.mu.Unlock()

	if n >= a.promisedNum {
		a.promisedNum = n
		promised = true
		acceptedNum = a.acceptedNum
		acceptedValue = a.acceptedValue
	}
	return
}

// Accept 接受提案
func (a *Acceptor) Accept(n int, value string) (accepted bool) {
	a.mu.Lock()
	defer a.mu.Unlock()

	if n >= a.promisedNum {
		a.promisedNum = n
		a.acceptedNum = n
		a.acceptedValue = value
		return true
	}
	return false
}

// Proposer Paxos 中的提案者
type Proposer struct {
	mu       sync.Mutex
	id       int
	n        int       // 当前提案编号
	v        string    // 当前提案值
	acceptors []*Acceptor
}

// NewProposer 创建提案者
func NewProposer(id int, acceptors []*Acceptor) *Proposer {
	return &Proposer{id: id, acceptors: acceptors}
}

// generateProposalNumber 生成全局唯一递增提案编号
// 实际实现中需要保证全局唯一，通常用 (节点ID << 48) | (term << 32) | 序列号
func (p *Proposer) generateProposalNumber(term int) int {
	return (p.id << 48) | (term << 32) | rand.Int()
}

// RunPaxos 运行 Paxos 算法的单轮共识
func (p *Proposer) RunPaxos(value string, term int) (chosen bool, chosenValue string) {
	p.mu.Lock()
	p.n = p.generateProposalNumber(term)
	p.v = value
	proposalNum := p.n
	p.mu.Unlock()

	// ========== 第一阶段：Prepare ==========
	promises := 0
	highestAcceptedNum := -1
	highestAcceptedValue := ""

	for _, acc := range p.acceptors {
		promised, acceptedNum, acceptedValue := acc.Prepare(proposalNum)
		if promised {
			promises++
			if acceptedNum > highestAcceptedNum {
				highestAcceptedNum = acceptedNum
				highestAcceptedValue = acceptedValue
			}
		}
	}

	// 未获得多数票，直接失败
	if promises <= len(p.acceptors)/2 {
		return false, ""
	}

	// ========== 第二阶段：Accept ==========
	// 如果有已接受的值，选择编号最大的；否则用自己提议的值
	finalValue := value
	if highestAcceptedNum > 0 {
		finalValue = highestAcceptedValue
	}

	accepts := 0
	for _, acc := range p.acceptors {
		if acc.Accept(proposalNum, finalValue) {
			accepts++
		}
	}

	if accepts > len(p.acceptors)/2 {
		return true, finalValue
	}
	return false, ""
}
```

---

## Multi-Paxos（工业用法）

原始 Paxos 只对单个值达成共识。实际使用中是对**一系列值**（日志条目）达成共识，即 Multi-Paxos：

| 优化 | 说明 |
|------|------|
| **省略 Prepare 阶段** | 有了稳定的 Leader 后，直接发 Accept |
| **Leader 唯一** | 避免多个 Proposer 竞争，提高效率 |
| **日志空洞** | 允许某些条目暂时不达成共识 |

> **Raft 本质上是简化版的 Multi-Paxos**，将日志索引作为提案编号。

---

## 对比 Raft vs Paxos

| 对比项 | Raft | Paxos |
|--------|------|-------|
| 可理解性 | ✅ 强 | ❌ 难 |
| Leader | ✅ 强依赖 | 可选 |
| 日志复制 | ✅ 清晰 | ❌ 模糊 |
| 成员变更 | ✅ Joint Consensus +渤海 | ❌ 未定义 |
| 工程实现 | etcd、TiKV、CockroachDB | Chubby（Google） |

---

## 常见追问

**Q：Paxos 和 Raft 的本质区别是什么？**

> 本质都是**过半机制**保证共识，区别在于：
> - **Raft** 将问题拆成 Leader 选举、日志复制、成员变更三个子问题，更容易实现
> - **Paxos** 是纯数学证明，直接给出一致性保证，但不关心如何落地
> 打个比方：Raft 是"工程规范"，Paxos 是"数学定理"。

**Q：Paxos 会不会有活锁？**

> 可能。两个 Proposer 交替提出更高编号的提案，导致都无法获得多数票。这就是为什么 Raft 使用**随机 election timeout** 来避免同时发起选举，也是为什么实际系统多使用 Multi-Paxos + 唯一 Leader 来避免这个问题。

**Q：为什么 Chubby 选择 Paxos 而不是 Raft？**

> Chubby 比 Raft 论文早几年，而且 Google 内部先实现 Paxos 再遇到 Raft。但新系统（如 etcd、TiKV）普遍选择 Raft，因为更易理解和实现。
