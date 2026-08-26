# 一致性哈希（Consistent Hashing）：原理、虚拟节点与 Go 实现

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：hash环、虚拟节点、数据迁移量、cache分布、负载均衡

## 核心答案（30 秒版）

**一致性哈希的核心思想**：将存储节点和 key 都映射到一个 $2^{32}$ 大小的 hash 环上，key 沿顺时针方向找到第一个节点作为归属。**添加/删除一个节点时，只有该节点附近的 key 需要重新路由，而不是所有 key**。

为解决单节点负载不均问题，引入**虚拟节点（virtual node）**——每个真实节点对应多个 hash 位置，分散在 hash 环上。典型做法是 100~200 个虚拟节点/真实节点。

---

## 深度展开

### 1. 为什么不用普通取模？

```
普通取模（N 个节点，key.hash() % N）的问题：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

初始状态：3 个节点 A(0)、B(1)、C(2)
  key=10 → 10%3 = 1 → B  ✓
  key=20 → 20%3 = 2 → C  ✓
  key=30 → 30%3 = 0 → A  ✓

新增节点 D：
  key=10 → 10%4 = 2 → D  ✗→ 从 B 变成 D（全乱了）
  key=20 → 20%4 = 0 → A  ✗→ 从 C 变成 A（全乱了）
  key=30 → 30%4 = 2 → D  ✗→ 从 A 变成 D（全乱了）

结论：增加 N 个节点，平均有 (N-1)/(N+1) 的比例需要迁移！
      当节点数从 3 变到 4 时，约 67% 的数据需要迁移！
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 2. Consistent Hashing 工作原理

```
hash 环（范围 0 ~ 2^32-1）：

     节点A        节点B          节点D
    ↙─────╲           ╲       ╱
   │  k1  │  k2   k3 │  k4  │
   │      │           │      │
    ╲─────┴───────────┴──────┘← hash 环（首尾相连）

规则：key 的 hash 值顺时针找第一个遇到的节点。

例：key=k2 的 hash 值落在 A 和 B 之间 → 顺时针走 → 命中 B
例：key=k5 的 hash 值在 D 之后（接近起点） → 绕回 → 命中 A

加入新节点 C（落在 A 和 B 之间）：
     节点A        节点C      节点B          节点D
    ↙─────╲         ╲               ╲       ╱
   │  k1  │   k2    │  k3   k4      │  k5  │
   │      │          │              │      │
    ╲─────┴──────────┴──────────────┴──────┘

影响范围：只有原来分配给 B 的 k2 被"截流"到新节点 C。
k3 仍在 B，k4 仍在 B —— 只有少量 key 需要迁移。
```

### 3. 虚拟节点的作用

```
没有虚拟节点（4 个节点分布在环上）：
        A              C
     ╱──────╲            ╲──────────╲
    │        │            │          │
    │        │            │          │
     ╲───────╲────────────╲──────────╱← 环上空隙大
                            B    D
问题：节点可能集中在某个区间，导致负载严重不均。

有虚拟节点（每节点 150 个虚拟节点）：
     A1  A48  A96      C23  C107
    ╱│╲╱│╲╱│╲  ╱│╲╱│╲  ╱│╲╱│╲  ...
   大量均匀散落的点填满整个环
   
结论：虚拟节点越多，负载分布越均匀。
推荐值：100~200 个/vnode。实际生产多用 150。
```

### 4. Go 语言实现

```go
package consistenthash

import (
	"crypto/md5"
	"sort"
	"strconv"
)

// HashFunction 自定义哈希函数签名
type HashFunction func(data []byte) uint32

// ConsistentHash 一致性哈希结构
type ConsistentHash struct {
	hashFunc      HashFunction       // 哈希函数
	sortedKeys    []uint32           // hash 环上的关键点（有序）
	virtualNodes  map[uint32]string  // hash值 → 真实节点名
	keyToNode     map[string]uint32  // 反向索引：key → hash值
}

const defaultVirtualNodes = 150

// New 创建一致性哈希实例
func New(hashFunc HashFunction, virtualNodes int) *ConsistentHash {
	if virtualNodes <= 0 {
		virtualNodes = defaultVirtualNodes
	}
	return &ConsistentHash{
		hashFunc:      hashFunc,
		virtualNodes:  make(map[uint32]string),
		keyToNode:     make(map[string]uint32),
	}
}

// DefaultHash 默认使用 md5，返回 uint32 范围
func DefaultHash(data []byte) uint32 {
	sum := md5.Sum(data)
	return uint32(sum[0])<<24 | uint32(sum[1])<<16 |
		uint32(sum[2])<<8 | uint32(sum[3])
}

// Add 添加节点
func (c *ConsistentHash) Add(key string) {
	for i := 0; i < defaultVirtualNodes; i++ {
		hash := c.hashFunc([]byte(strconv.Itoa(i) + key))
		c.virtualNodes[hash] = key
	}
	c.updateSortedKeys()
}

// updateSortedKeys 维护 sortedKeys 数组有序性
func (c *ConsistentHash) updateSortedKeys() {
	c.sortedKeys = make([]uint32, 0, len(c.virtualNodes))
	for k := range c.virtualNodes {
		c.sortedKeys = append(c.sortedKeys, k)
	}
	sort.Slice(c.sortedKeys, func(i, j int) bool {
		return c.sortedKeys[i] < c.sortedKeys[j]
	})
}

// Get 获取 key 对应的节点
func (c *ConsistentHash) Get(key string) string {
	if len(c.sortedKeys) == 0 {
		return ""
	}
	
	hash := c.hashFunc([]byte(key))
	idx := sort.Search(len(c.sortedKeys), func(i int) bool {
		return c.sortedKeys[i] >= hash
	})
	
	// 超出范围 → 绕回到环的起点
	if idx == len(c.sortedKeys) {
		idx = 0
	}
	
	return c.virtualNodes[c.sortedKeys[idx]]
}

// Remove 移除节点（清理其所有虚拟节点）
func (c *ConsistentHash) Remove(key string) {
	removedKeys := make([]uint32, 0)
	for h, n := range c.virtualNodes {
		if n == key {
			removedKeys = append(removedKeys, h)
		}
	}
	for _, h := range removedKeys {
		delete(c.virtualNodes, h)
	}
	c.updateSortedKeys()
}
```

### 5. Go stdlib `container/heap` 进阶实现（性能更优）

标准库提供了 `sort.Search` 用于二分查找，但在频繁增删场景下，建议使用平衡树或跳表替代排序切片：

```go
// 使用 go-libconsul-hashring 等成熟库：
// go get github.com/buraksezer/consistent

import "github.com/buraksezer/consistent"

type serverConfig struct {
	host string
	port int
}

func (s *serverConfig) String() string {
	return fmt.Sprintf("%s:%d", s.host, s.port)
}

func main() {
	c := consistent.New(
		func(obj interface{}) string { // 自定义 hasher
			s := obj.(*serverConfig)
			return s.String()
		},
		consistent.DefaultNumVirtualNodes(150), // 150 vnode/node
	)

	// 添加节点
	nodes := []*serverConfig{
		{"10.0.0.1", 8080},
		{"10.0.0.2", 8080},
		{"10.0.0.3", 8080},
	}
	for _, n := range nodes {
		c.Add(n)
	}

	// 查询
	cluster := c.Get("user-123")
	fmt.Printf("user-123 -> %v\n", cluster)
}
```

### 6. 常见应用场景

| 场景 | 说明 |
|------|------|
| **分布式缓存** | Memcached/Redis 集群根据 key 路由到不同节点 |
| **分布式分片存储** | 按 key 路由到不同的数据库分片 |
| **CDN 节点选择** | 用户域名解析到最近的 CDN 节点 |
| **区块链 P2P 网络** | 确定区块/交易归属哪个节点处理 |
| **微服务路由** | 根据请求 ID 路由到固定后端实例（会话保持） |

### 7. 面试高频追问

**Q：虚拟节点个数怎么确定？**

> 一般选 100~200。太少的话，hash 环上分布不均匀，负载倾斜严重；太多的话，内存占用增大，遍历开销变大。经验值是 150。可以通过测试模拟不同 vnode 数量下的负载方差来调优。

**Q：节点故障时，数据会丢失吗？**

> 一致性哈希本身不保证冗余——故障节点的 key 会迁移到下一个节点。如果需要高可用，需要在应用层做副本（比如 Redis Cluster 的主从复制），或者用 CRDT/Paxos 等共识算法保证多副本一致。

**Q：一致性哈希的时间复杂度是多少？**

> 查询：O(log N)，N 为虚拟节点总数（用二分查找）。添加/删除单个节点：O(V log V)，V 为总虚拟节点数（需要重新排序）。如果节点很少（比如 <1000），性能不是瓶颈。对于海量节点，可以用 B 树或跳表替代排序切片。

---

## 面试话术

**Q：如何把数据均匀地分发到 N 台服务器上？**

> 最直观的做法是取模：`hash(key) % N`。但问题是，新增服务器后几乎所有 key 都需要重新定位，迁移成本巨大。生产上应该用**一致性哈希**——把服务器和 key 都映射到一个 hash 环上，key 顺时针找一个最近的服务器。这样只影响相邻的一小部分数据。再配合**虚拟节点**（每个真实节点映射 150 个 hash 位置），就能保证负载均衡。像 Redis Cluster、Memcached 都在用这个方案。

🗣️ **记忆口诀**：**"取模迁移全乱，哈希环只动隔壁，加虚拟节点保均衡"**

---

*参考实现：[geektutu/geecache](https://github.com/geektutu/geecache/tree/master/day4-consistent-hash) · [stathat/consistent](https://github.com/stathat/consistent)*

[🏠 首页](../../../README.md) · [📦 分布式系统](../README.md) · [💬 理论基础](../../docs/03-distributed/01-theory/README.md)
