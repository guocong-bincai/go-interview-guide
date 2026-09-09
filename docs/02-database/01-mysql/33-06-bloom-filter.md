# 布隆过滤器（Bloom Filter）在缓存穿透防护中的应用

> 考察频率：★★★★☆  难度：★★★★☆
> 关键词：布隆过滤器、哈希函数、误判率、Redisson、缓存穿透、位图压缩

## 面试官考察意图

面试官通常从这条链提问：

1. "如果黑客一直请求不存在的数据 ID，你的系统会怎样？" → 引出缓存穿透
2. "普通的空值缓存方案有什么缺陷？" → 引出需要更专业的防护
3. "听说过布隆过滤器吗？它是什么原理？" → 看候选人是否有进阶知识
4. "布隆过滤器有缺点吗？" → 区分度题，看是否真正理解

**区分度**在于能否说清布隆过滤器的数学原理（为什么会有误判）、如何计算合适的参数、以及 Go / Redis 中实际怎么用的。

---

## 核心答案（30 秒版）

| 问题 | 一句话回答 |
|------|-----------|
| **布隆过滤器是什么？** | 一种空间极省的概率型数据结构，判断「元素一定不存在」或「可能存在」|
| **怎么防缓存穿透？** | 将合法业务 ID 全部插入布隆过滤器，请求先查过滤器→不在直接返回空（不查 DB） |
| **布隆过滤器有什么缺点？** | ❌ 不能删除元素（除非用 Counting Bloom Filter）；❌ 存在误判率（说有但实际没有） |
| **Go 语言有现成的实现吗？** | 有：`github.com/bits-and-blooms/bloom-go`（纯 Go）+ `github.com/redisson/redisson-go`（Redis 分布式版） |
| **误判率多大？** | 默认 ~1%~3%，可通过调整 bit 数组长度和哈希函数数量控制 |

---

## 深度展开

### 1. 布隆过滤器的底层原理

```
布隆过滤器的组成：
├── 一个很长的二进制位数组（bit array），初始全为 0
├── K 个独立的哈希函数
└── 支持 Add 和 Contains 两种操作
```

**Add 操作流程：**

```
bloom.Add("user:10001")

步骤：
1. h1("user:10001") = 3    → bit[3] = 1
2. h2("user:10001") = 15   → bit[15] = 1  
3. h3("user:10001") = 42   → bit[42] = 1

结果：bit 数组变为 [0,0,0,1,0,0,...,0,1,...,0,1,...]
```

**Contains 查询流程：**

```
bloom.Contains("user:10001")  → true  （所有对应的 bit 都是 1）
bloom.Contains("user:99999")  → false （至少有一个 bit 是 0）
bloom.Contains("user:88888")  → true  （碰巧三个 bit 都被其他 key 设置了 → 误判！）
```

### 2. 误判率公式与参数选择

```
误判率 p ≈ (1 - e^(-kn/m))^k

其中：
  k = 哈希函数数量
  m = bit 数组总位数
  n = 预计存储的元素数量
  e = 自然常数

最佳哈希函数数：k = ln(2) × m/n ≈ 0.693 × m/n

最优 bit 数：m = -n × ln(p) / (ln(2)^2) ≈ 1.44 × n × log2(1/p)
```

**常见场景推荐配置：**

| 预期元素数 | 允许误判率 | 所需 bit 数 | 哈希函数数 |
|-----------|----------|------------|----------|
| 1000 万 | 1% | ~96 Mbits (~12 MB) | 7 |
| 1 亿 | 0.1% | ~192 Mbits (~24 MB) | 10 |
| 10 亿 | 0.01% | ~960 Mbits (~120 MB) | 14 |

**关键结论**：
- 布隆过滤器的内存占用仅约 **每个元素 10~20 bits**（取决于误判率要求）
- 相比直接存 Hash Set，节省了几十倍的空间
- Redis 的布隆过滤器模块仅需约 **13.5MB** 就能存储 1 亿个元素的 1% 误判率过滤器

### 3. Redis 布隆过滤器实战（Redis 4.0+ / RedisBloom）

```bash
# 安装 RedisBloom 模块后使用

# 创建布隆过滤器（预估 100 万元素，误判率 0.01）
BF.RESERVE items_filter 0.0001 1000000

# 添加合法的业务 ID
BF.ADD items_filter 10001
BF.ADD items_filter 10002
BF.MADD items_filter 10003 10004 10005 ...

# 查询时先检查（注意 BF.EXISTS 返回 1=存在/可能，0=一定不存在）
BF.EXISTS items_filter 10001    → 1  （确实存在）
BF.EXISTS items_filter 99999    → 0  （一定不存在 → 直接拦截，不查 DB！）
BF.EXISTS items_filter 88888    → 1  （可能被误判 → 仍需查 DB）
```

### 4. Go + Redis 完整示例

```go
package main

import (
    "context"
    "log"
    "github.com/redis/go-redis/v9"
    "github.com/bits-and-blooms/bloom-go"
)

// === 方案一：本地布隆过滤器（适合单实例）===

func localBloomFilterExample() {
    // 预估 1000 万用户 ID，允许 1% 误判率
    bf := bloom.NewWithEstimates(10_000_000, 0.01)
    
    // 初始化：将所有合法的 user_id 加入过滤器
    for id := int64(1); id <= 10_000_000; id++ {
        bf.Add([]byte(fmt.Sprintf("%d", id)))
    }
    
    // 查询流程
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    
    userID := 12345 // 模拟用户输入
    
    if !bf.Check([]byte(fmt.Sprintf("%d", userID))) {
        // 一定不存在，直接返回空（不查 DB！）
        log.Println("非法请求，布隆过滤器已拦截")
        return
    }
    
    // 可能存在 → 正常走缓存查询
    val, err := rdb.Get(ctx, fmt.Sprintf("user:%d", userID)).Result()
    if err == redis.Nil {
        // Cache MISS → 查 DB
        user := queryDB(userID)
        rdb.Set(ctx, fmt.Sprintf("user:%d", userID), user, 30*time.Minute).Err()
    } else {
        // Cache HIT
        log.Printf("User found in cache: %s\n", val)
    }
}

// === 方案二：分布式布隆过滤器（多实例共享）===

// 通过 Redis 的 BF.EXISTS 命令实现分布式布隆过滤器
type DistributedBloomFilter struct {
    rdb *redis.Client
    key string
}

func NewDistributedBloomFilter(rdb *redis.Client, key string) *DistributedBloomFilter {
    return &DistributedBloomFilter{rdb: rdb, key: key}
}

func (b *DistributedBloomFilter) Add(id int64) error {
    _, err := b.rdb.BFAdd(context.Background(), b.key, strconv.FormatInt(id, 10)).Result()
    return err
}

func (b *DistributedBloomFilter) Exists(id int64) (bool, error) {
    exists, err := b.rdb.BFExists(context.Background(), b.key, strconv.FormatInt(id, 10)).Result()
    if err != nil {
        return false, err
    }
    return exists == 1, nil // Redis 返回 1 表示可能存在/一定存在，0 表示一定不存在
}

func (b *DistributedBloomFilter) QueryUserID(userID int64) (string, error) {
    exists, err := b.Exists(userID)
    if err != nil {
        return "", err
    }
    if !exists {
        return "", ErrNotExists // 布隆过滤器说一定不存在 → 不查 DB
    }
    // 可能存在 → 正常查缓存再查 DB
    return getFromCacheOrDB(userID)
}
```

### 5. 布隆过滤器 vs 空值缓存对比

| 维度 | 空值缓存（null TTL） | 布隆过滤器 |
|------|-------------------|-----------|
| **内存占用** | 每个不存在的 key 都占一个 slot | ~10~20 bits / element |
| **拦截效果** | 防止重复查 DB，但不减少首次请求 | 首次就拦截，完全不查 DB |
| **适用场景** | 偶发的不合法请求 | 恶意大量探测不存在的 key |
| **维护成本** | 零 | 需要定期重建（数据变化时）|
| **误判** | 无 | 存在（但可控） |
| **删除元素** | 简单（SET EX 过期即可） | 困难（不支持精确删除） |

---

## 高频追问

### Q1：布隆过滤器能删除元素吗？

**标准的布隆过滤器不支持删除。** 因为多个元素的 bit 位可能重叠，删一个会影响其他的判断。

如果需要删除功能，可用 **Counting Bloom Filter**：将 bit 改为计数器。

```go
// Counting Bloom Filter 原理：
// 每个位置是一个 4-bit 或 8-bit 计数器，而不是 1 bit
// Add: 对应位置的计数器 +1
// Remove: 对应位置的计数器 -1（只减到 0）
// Contains: 检查所有对应位置 > 0

// Go 实现：
import "github.com/seiflotfy/cuckoofilter"

cf := cuckoofilter.New(1000000)
cf.Insert("user:10001")

// 可以删除
cf.Remove("user:10001")

// 查询
cf.Contains("user:10001")  // false（已被移除）
```

### Q2：布隆过滤器更新频繁怎么办？（比如商品库经常增减）

**方案一：定时全量重建**

```go
// 每天凌晨低峰期重建整个布隆过滤器
func rebuildBloomEveryDay() {
    bf := newBloomFromDB() // 从数据库读取全量数据重建
    publishToAllInstances(bf)
}
```

**方案二：双布隆过滤器切换**

```go
var bf1, bf2 *BloomFilter
var active atomic.Int32 // 0=bf1, 1=bf2

func check(userID int64) bool {
    if active.Load() == 0 {
        return bf1.Check(userID) && bf2.Check(userID)
        // 两个都认为是存在才放行
    }
    return bf2.Check(userID)
}
```

### Q3：除了防缓存穿透，布隆过滤器还有哪些应用场景？

1. **反垃圾邮件**：邮箱列表预先导入布隆过滤器，快速判断黑名单
2. **爬虫去重**：URL 队列去重，避免重复爬取
3. **磁盘寻址优化**：SSD 上判断数据是否存在于缓存层
4. **Kubernetes Pod 调度**：快速判断某个节点是否已调度了某些工作负载
5. **数据库优化**：判断索引页是否在 Buffer Pool 中，决定是否需要磁盘 IO

---

## 延伸阅读

- [Bloom Filters by Example — Martin Kersten](https://www.bu.edu/files/2013/11/bloomfilters.pdf)
- [RedisBloom Documentation](https://redis.io/docs/stack/bloom/)
- [GitHub: bits-and-blooms/bloom-go](https://github.com/bits-and-blooms/bloom-go)
