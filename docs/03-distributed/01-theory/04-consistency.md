# 分布式一致性

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：强一致、线性一致、顺序一致、因果一致、最终一致

## 一致性全景图

```
强一致性强
    │
    ├── 线性一致性（Linearizability）
    │       └── 操作按全局时钟排序，客户端感知即时
    │
    ├── 顺序一致性（Sequential Consistency）
    │       └── 各节点看到操作顺序一致，但不保证全局时钟顺序
    │
    └── 因果一致性（Causal Consistency）
            └── 只保证有因果关系的操作顺序
                │
                └── 最终一致性（Eventual Consistency）
                        └── 不保证顺序，只保证最终收敛
```

---

## 1. 线性一致性（Linearizability）

### 定义

**最严格的一致性模型**。所有操作像是按一个全局时钟排序，对外部观察者来说，读能读到最近一次写的结果。

> "看起来像是只有一个副本"——Maurice Herlihy & Jeannette Wing

### 典型场景

- **分布式锁**（etcd、Zookeeper）
- **唯一性约束**（用户 ID 去重）
- **配置文件更新**

### Go 验证：线性一致性测试

```go
// 使用 go-play/linearizability 进行线性化验证
package test

import (
	"context"
	"sync/atomic"
	"testing"
	"time"

	"github.com/rel当作/linearizability"
)

// 模拟一个简单的分布式锁服务
type DistributedLock struct {
	held    atomic.Bool
	ownerID int
}

func (l *DistributedLock) Lock(ctx context.Context, id int) bool {
	if l.held.CompareAndSwap(false, true) {
		return true
	}
	select {
	case <-ctx.Done():
		return false
	case <-time.After(10 * time.Millisecond):
		return l.Lock(ctx, id)
	}
}

func (l *DistributedLock) Unlock(ctx context.Context, id int) bool {
	if l.held.CompareAndSwap(true, false) {
		return true
	}
	return false
}

// 线性化测试：验证锁操作的线性一致性
func TestLockLinearizability(t *testing.T) {
	lock := &DistributedLock{}

	// 定义操作模型
	lockOp := linearizability.NewOperation(
		func(input interface{}) interface{} {
			id := input.(int)
			ctx, cancel := context.WithTimeout(context.Background(), time.Second)
			defer cancel()
			ok := lock.Lock(ctx, id)
			return ok
		},
		func(output interface{}) interface{} {
			return output
		},
		time.Millisecond,
	)

	unlockOp := linearizability.NewOperation(
		func(input interface{}) interface{} {
			id := input.(int)
			ctx, cancel := context.WithTimeout(context.Background(), time.Second)
			defer cancel()
			ok := lock.Unlock(ctx, id)
			return ok
		},
		func(output interface{}) interface{} {},
		time.Millisecond,
	)

	// 运行线性化检查
	model := linearizability.NewModel()
	model.AddReadOperation(lockOp)
	model.AddWriteOperation(unlockOp)

	history := []linearizabilitymodels.History{}
	// ... 运行并发测试收集历史 ...

	ok := linearizability.Validate(model, history)
	if !ok {
		t.Fatal("Lock is not linearizable")
	}
}
```

### 时间复杂度

线性一致性读/写 = **O(RTT)**（一次网络往返）

---

## 2. 顺序一致性（Sequential Consistency）

### 定义

所有进程看到的**操作全局顺序**一致，且该顺序与各进程的本地程序顺序一致。

### vs 线性一致性的区别

```go
// 线性一致性要求：先写先读
// 顺序一致性允许：只要各节点看到顺序一致即可

// 例子：两个节点的操作
NodeA: W(x)  // 写 x=1
NodeB: R(x)  // 读 x（可能读到旧值，因为没要求全局时钟顺序）

// 线性一致性：NodeB 必须读到 x=1
// 顺序一致性：只要 NodeA 和 NodeB 对 W(x) 的顺序达成一致即可
```

---

## 3. 因果一致性（Causal Consistency）

### 定义

只保证**有因果关系**的操作顺序一致，无因果关系的操作可以并发。

### 典型系统

- **Cassandra**（使用向量时钟追踪因果关系）
- **DynamoDB**

### 向量时钟（Vector Clock）示例

```go
package causal

import (
	"fmt"
	"sort"
)

// VectorClock 向量时钟
type VectorClock map[string]int

// Merge 合并两个向量时钟
func (vc VectorClock) Merge(other VectorClock) {
	for node, val := range other {
		if vc[node] < val {
			vc[node] = val
		}
	}
}

// HappenedBefore 判断 a 是否 happened-before b
func (vc VectorClock) HappenedBefore(other VectorClock) bool {
	atLeastOne := false
	for node, aVal := range vc {
		bVal := other[node]
		if aVal > bVal {
			return false // a 在某些维度更新，不可能在 b 之前
		}
		if aVal < bVal {
			atLeastOne = true
		}
	}
	// 检查 other 独有的节点
	for node, bVal := range other {
		if _, ok := vc[node]; !ok && bVal > 0 {
			atLeastOne = true
		}
	}
	return atLeastOne
}

// Concurrent 判断两个向量时钟是否并发
func (vc VectorClock) Concurrent(other VectorClock) bool {
	return !vc.HappenedBefore(other) && !other.HappenedBefore(vc)
}

func Example() {
	vc1 := VectorClock{"A": 1, "B": 0}
	vc2 := VectorClock{"A": 1, "B": 1}

	fmt.Printf("vc1 happened-before vc2: %v\n", vc1.HappenedBefore(vc2)) // true
	fmt.Printf("vc2 happened-before vc1: %v\n", vc2.HappenedBefore(vc1)) // false
	fmt.Printf("concurrent: %v\n", vc1.Concurrent(vc2))                  // false
}
```

---

## 4. 最终一致性（Eventual Consistency）

### 定义

不保证操作顺序，允许短暂不一致，但经过**有限时间**后系统收敛到一致。

### 典型实现

| 系统 | 一致性保证 | 冲突处理 |
|------|-----------|---------|
| Cassandra | 最终一致 | Last-Write-Wins / 自定义合并 |
| DynamoDB | 最终一致 | 矢量时钟冲突解决 |
| Riak | 最终一致 | CRDT |
| Redis Cluster | 最终一致 | 主从异步复制 |

### 冲突处理策略

```go
// Last-Write-Wins（最常用）
type LWWRegister struct {
	value      string
	timestamp  int64
}

func (r *LWWRegister) Merge(other LWWRegister) {
	if other.timestamp > r.timestamp {
		r.value = other.value
		r.timestamp = other.timestamp
	}
}

// CRDT（Conflict-free Replicated Data Types）无冲突数据类型
// 示例：G-Counter（只增计数器）
type GCounter struct {
	mu     sync.Mutex
	nodeID string
	count  map[string]int64 // 每节点计数
}

func (c *GCounter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.count[c.nodeID]++
}

func (c *GCounter) Merge(other GCounter) {
	c.mu.Lock()
	defer c.mu.Unlock()
	for node, val := range other.count {
		if c.count[node] < val {
			c.count[node] = val
		}
	}
}

func (c *GCounter) Sum() int64 {
	c.mu.Lock()
	defer c.mu.Unlock()
	var total int64
	for _, v := range c.count {
		total += v
	}
	return total
}
```

---

## 5. 实际场景选型

| 场景 | 推荐一致性级别 | 实现方案 |
|------|--------------|---------|
| 分布式锁 | 线性一致 | etcd/Redis RedLock |
| 库存扣减 | 强一致（事务） | 数据库事务 / TCC |
| 用户资料 | 最终一致 | DynamoDB / Cassandra |
| 消息通知 | 最终一致 | 消息队列 |
| 配置下发 | 顺序一致 | Zookeeper |
| 秒杀库存 | 强一致 | Redis 原子操作 + 数据库 |

---

## 面试话术

**Q：线性一致性和顺序一致性有什么区别？**

> 线性一致性要求操作按**真实时间**排序，所有客户端都能感知到"先写先读"；顺序一致性只要求各节点看到的**全局操作顺序一致**，但不保证这个顺序和真实时间一致。比如：节点 A 先写入 x=1，节点 B 后写入 x=2，线性一致性保证所有节点看到的顺序是 x=1 → x=2，而顺序一致性只要求所有节点看到的顺序一致（可以是 x=2 先，也可以 x=1 先，但必须统一）。

**Q：如何实现线性一致的分布式锁？**

> 有三种主流方案：1）**etcd**：基于 Raft 协议 + TTL 租约，KV 操作原子化，适合 Kubernetes 生态；2）**Zookeeper**：基于 ZAB 协议 + 临时有序节点，CP 系统；3）**Redis RedLock**：基于多个独立 Redis 节点多数派写入，有争议（RedLock 不是线性一致的）。生产环境推荐 etcd 或 Zookeeper。

**Q：CP 和 AP 系统在实际业务中怎么选？**

> 看业务对"数据一致性"和"可用性"的容忍度。比如**金融交易、订单系统**选 CP（etcd、Zookeeper），因为数据不一致是灾难性的；**社交 Feed、评论系统**选 AP（Cassandra、DynamoDB），因为短暂不一致用户感知不强，但服务不可用会被投诉。现实中很多系统是混合的：核心链路用 CP，非核心用 AP。
