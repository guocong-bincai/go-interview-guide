# Redis Sentinel vs Redis Cluster：架构对比与选型决策

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：主从复制、故障转移、分片哈希槽、集群模式、哨兵机制

## 核心答案（30 秒版）

**Redis Sentinel** 和 **Redis Cluster** 解决的是不同层次的问题：

| | Sentinel | Cluster |
|---|---|---|
| **定位** | 高可用（HA）**监控+自动故障转移** | **分片+高可用**的完整方案 |
| **数据分布** | 单节点全量存储 | 16384 个 hash slot 分片 |
| **扩展性** | ❌ 只能水平扩容（多实例但每个都是全量副本） | ✅ 直接 scale-out |
| **跨节点操作** | 支持 MULTI/SETNX 等跨 key 事务 | ❌ 不支持，key 必须在同一 slot |
| **容量上限** | 受单机内存限制 | 多节点合计 = 总容量 |

**一句话总结**：不需要分片的场景用 Sentinel；需要水平扩容存大量数据的场景用 Cluster。

---

## 深度展开

### 1. Redis Sentinel 架构

```
Sentinel 高可用架构：

┌───────────┐          ┌───────────┐          ┌───────────┐
│   Master  │◀────────▶│ Sentinel-1│◀────────▶│ Sentinel-2│
│           │ replicas │  (监控)    │ votes    │  (监控)    │
└─────┬─────┘          └───────────┘          └───────────┘
      │                      ▲                       ▲
      │ replicas             │                       │
      ▼                      │                       │
┌───────────┐          ┌─────┴─────┐           ┌─────┴─────┐
│  Slave-1  │          │ Sentinel-3│           │ Client App│
│  (只读)   │          │  (监控)    │           │ (通过 VIP)│
└───────────┘          └───────────┘           └───────────┘

故障转移流程：
1. Sentinel 发现 Master 不可达（主观下线 → ODOWN）
2. 多个 Sentinel 达成一致（SDOWN → ODOWN，需 majority +1 同意）
3. 选举 Leader Sentinel
4. 选择最好的 Slave 作为新 Master（优先选 replica_priority 最高的）
5. 通知其他 Slave 复制新 Master
6. 更新配置并通知客户端
```

**关键参数**：

```go
// sentinel.conf 核心配置
sentinel monitor mymaster 127.0.0.1 6379 2
//                    ↑       ↑        ↑   ↑
//                  name    IP    port quorum（过半数）

sentinel down-after-milliseconds mymaster 30000     // 30s 无响应判定为下线路
sentinel failover-timeout mymaster 180000            // 故障转移超时 3min
sentinel parallel-syncs mymaster 1                   // 同步复制时的并行数（1 更安全）
```

**主观下线 vs 客观下线**：

| 阶段 | 触发条件 | 说明 |
|------|---------|------|
| SDOWN（Subjective Down） | 单个 Sentinel 检测到 Master 超时 | 该 Sentinel 自己的判断 |
| ODOWN（Objective Down） | 多数 Sentinel（quorum 数量）都标记为 SDOWN | 集群层面的确认，开始故障转移 |

### 2. Redis Cluster 架构

```
Cluster 分片架构：

         ┌─────────────────────────────────────────────┐
         │            16384 Hash Slots                 │
         │  0────────────10922────────────16383        │
         │     Slot Range 1      │     Slot Range 2    │
         └───────────────────────┼─────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
       ┌──────────┐      ┌──────────┐      ┌──────────┐
       │ Node A   │      │ Node B   │      │ Node C   │
       │ Slots 0~5460  │      │ Slots 5461~10922 │      │ Slots 10923~16383│
       ├──────────┤      ├──────────┤      ├──────────┤
       │ Master   │      │ Master   │      │ Master   │
       │ Replica  │      │ Replica  │      │ Replica  │
       └──────────┘      └──────────┘      └──────────┘

每个节点管理约 5461 个 slot（16384/3 ≈ 5461）
```

**Hash Slot 计算**：

```go
// CRC16(key) % 16384
import "github.com/crc16"

func hashSlot(key string) int {
	// Redis 使用 CRC16 算法
	return crc16.CRC16([]byte(key)) % 16384
}

// 示例：
// hashSlot("foo") = 3323 → Node A
// hashSlot("bar") = 9134 → Node B
// hashSlot("baz") = 12456 → Node C
```

**MOVED / ASK 重定向**：

```go
// Redis Cluster 客户端必须处理两种重定向：

// 场景1：MOVED —— key 在另一个节点上，永久迁移
// 客户端收到 MOVED 后更新本地路由表
// MOVED <slot> <host:port>
redis > GET foo
MOVED 3323 10.0.0.1:6379  // foo 现在属于 Node A

// 场景2：ASK —— 正在迁移中的临时重定向
// 当节点间迁移 slot 时，旧节点会返回 ASK
// ASK <slot> <host:port>
redis > GET bar
ASK 9134 10.0.0.2:6379  // bar 正在迁移到 Node B，暂时查询这里
// 客户端发送 ASKING 命令后再执行查询
```

### 3. 对比决策矩阵

```
┌──────────────────────┬──────────────┬──────────────┐
│     需求维度         │   Sentinel   │   Cluster    │
├──────────────────────┼──────────────┼──────────────┤
│ 数据量小（<10GB）     │  ✅ 推荐     │  ⚠️ 过杀    │
│ 需要水平扩容          │  ❌ 不支持    │  ✅ 支持     │
│ 跨 key 原子操作       │  ✅ 支持     │  ❌ 不支持    │
│ 简单运维              │  ✅ 更简单    │  较复杂      │
│ 读写分离              │  手动切换    │  自动(readonly)│
│ Multi Key 事务        │  ✅ 支持     │  ❌ 不支持    │
│ Pipeline 批量操作     │  ✅ 支持     │  跨 slot 失败 │
└──────────────────────┴──────────────┴──────────────┘
```

### 4. Go 连接 Redis Cluster 示例

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/redis/go-redis/v9"
)

func main() {
	rdb := redis.NewClusterClient(&redis.ClusterOptions{
		Addrs: []string{
			":6379", ":6380", ":6381",
			":6382", ":6383", ":6384",
		},
		MaxRedirects:   8,           // MOVED/ASK 最大重定向次数
		RouteByLatency: true,        // 延迟最低的路由
		RouteRandomly:  false,       // 随机路由（适合读负载分散）
		PoolSize:       20,          // 连接池大小
	})

	ctx := context.Background()
	
	// Cluster 模式下，key 必须落在同一个 slot 才能做多 key 操作
	err := rdb.Set(ctx, "user:1:profile", "value1", 0).Err()
	if err != nil {
		log.Fatal(err)
	}
	
	val, err := rdb.Get(ctx, "user:1:profile").Result()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(val)

	// Set TTL on multiple keys — Redis Cluster 只允许同 slot！
	// MSET user:1:name Alice user:1:email alice@example.com  ✅ OK
	// MSET user:1:name Alice user:2:name Bob                ❌ CROSSSLOT
}
```

### 5. Redis Cluster 的常见陷阱

| 陷阱 | 说明 | 解决方案 |
|------|------|---------|
| **Pipeline + 跨 slot** | Pipeline 中涉及不同 slot 的 key 会失败 | 改用 MGET/MSET，或者确保所有 key 在同一个 slot |
| **SCAN 分页** | SCAN 是游标式迭代，不是精确分页 | 连续调用 SCAN cursor 直到返回 0 |
| **Lua 脚本** | Lua 只能在同一个 slot 执行 | 脚本中涉及的 key 必须全部在同一 slot |
| **Slots 分配不均** | 新增节点后某些节点 slot 过多 | 手动重新平衡：`CLUSTER REBALANCE` |
| **大 Key 问题** | cluster 不能像单机那样优雅淘汰大 key | 使用 `UNLINK`（异步删除）替代 `DEL` |

---

## 面试高频追问

**Q：Sentinel 故障转移期间会发生什么？**

> 转移过程约几秒到几十秒。Master 宕机后：① Sentinel 发现并判定为 ODOWN；② 选出 Leader Sentinel；③ 选择最优 Slave 提升为 Master；④ 修改其余 Slave 的配置指向新 Master；⑤ 更新配置文件。**转移期间客户端可能短暂读到旧 Master 的数据或写入失败**。Go 客户端会自动重试 MOVED 重定向。

**Q：Redis Cluster 怎么实现负载均衡？**

> ① 读取请求可以设置 `RouteByLatency` 路由到延迟最低的节点；② 设置 `RouteRandomly` 随机路由分散读压力；③ 写请求严格按照 slot 路由到对应节点，无法主动分流。**如果某个 slot 数据过大导致热点，可以考虑手动重新分片**（`redis-trib reshard`）。

**Q：为什么 Cluster 不支持 MULTI/EXEC 事务？**

> 因为事务涉及多个 key，而不同 key 可能分布在不同的 slot 对应的节点上。跨节点的事务需要两阶段提交（2PC），Redis 没有实现这套机制。如果需要跨 slot 的原子操作，要么把 key 放到同一个 slot（用 `{}` 簇设计如 `user:{1000}:name` 和 `user:{1000}:email`），要么用分布式锁 + 应用层保证一致性。

---

## 面试话术

**Q：你们怎么用 Redis？用了 Sentinel 还是 Cluster？**

> 我们根据业务规模来定。**数据量小且不需要分片的场景用 Sentinel**——比如缓存系统、分布式锁，一个 Master + 几个 Slave，Sentinel 监控故障转移即可。但如果要存**大量会话数据或时序数据**，就用 **Redis Cluster**——16384 个 hash slot 分布在多个节点上，可以直接水平扩容。需要注意的是 Cluster 不支持跨 slot 的多 key 操作，所以 key 设计时要考虑 slot 归属，同类数据用 `{}` 簇在一起。

🗣️ **记忆口诀**：**"Sentinel 保命不扩容量，Cluster 能扩不能事务"**

---

*官方文档：[Redis Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/) · [Redis Cluster Spec](https://redis.io/docs/latest/operate/oss_and_stack/architecture/cluster/)*

[🏠 首页](../../../README.md) · [📦 分布式系统](../README.md) · [⚙️ 协调服务](../../docs/03-distributed/05-coordination/README.md)
