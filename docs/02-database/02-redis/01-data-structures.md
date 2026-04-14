# Redis 数据结构底层实现

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：SDS、ziplist、quicklist、listpack、skiplist、intset、bitmap

## Redis 为什么这么快

Redis 快的原因不只是"内存数据库"，更重要的是**每种数据结构都有精心设计的底层编码**，能用 O(1) 或 O(log n) 完成的绝不用 O(n)。

```
Redis 编码层（Encoding）一览：
==========================

String        → raw / int / embstr
Hash           → ziplist（<512字段） / hashtable
List           → ziplist（<64元素） / quicklist / linkedlist
Set            → intset（全是整数<512） / hashtable
Zset           → ziplist（<128元素） / skiplist + dict
Bitmap         → String 的位操作
HyperLogLog    → 专用字符串编码
Geospatial     → ziplist（早期）→ sorted set（geospatial 2.4+）
```

---

## 1. SDS（Simple Dynamic String）

### 传统 C 字符串的问题

```c
// C 字符串：strlen 获取长度需要 O(n)，因为要遍历到 \0
char buf[100] = "hello";
strlen(buf); // O(n) - 每次调用都要扫描
```

C 字符串的缺陷：
- **长度计算 O(n)**：每次都要遍历到 `\0`
- **不安全**：没有长度记录，缓冲区溢出风险
- **修改代价高**：追加可能触发重新分配 + 复制

### SDS 结构设计

```c
// Redis 源码：sds.h
struct sdshdr {
    int len;        // 已使用字节数（字符串长度）→ O(1) 获取长度
    int free;       // 剩余可用字节数
    char buf[];     // 柔性数组，存放实际字符串
};
```

```
SDS 示例："hello"

| 5 (len) | 5 (free) | h | e | l | l | o | \0 |
                              ↑ buf 起始
strlen(SDS) = sdshdr.len = 5 → O(1) 读取长度！
```

### SDS 的五大优势

| 优势 | 原理 |
|------|------|
| **O(1) 长度获取** | `len` 字段直接记录 |
| **防缓冲区溢出** | `free` 记录剩余空间，空间不够先扩展再写入 |
| **惰性空间释放** | 缩短时只改 `len`，不立即回收 `free` 空间 |
| **二进制安全** | 不依赖 `\0` 判断结尾，可以存二进制数据 |
| **内存预分配** | `free < len` 时，扩展时多分配 1MB 避免频繁重分配 |

```c
// SDS 追加示例
sds s = sdsnew("hello");  // len=5, free=0
s = sdscat(s, " world");  // 自动扩展：检查 free 够不够，不够就 realloc
// 扩展策略：< 1MB 时翻倍扩展，> 1MB 时多分配 1MB
```

### Go 模拟 SDS 思想

```go
type SDS struct {
    Len   int
    Free  int
    Buf   []byte
}

func (s *SDS) Append(p []byte) {
    needed := len(p)
    if s.Free < needed {
        // 扩展策略：小于1MB时翻倍，否则多分配1MB
        grow := needed
        if s.Len+s.Free < 1024*1024 {
            grow = s.Len + needed + s.Len // 翻倍
        } else {
            grow = s.Len + needed + 1024*1024
        }
        newBuf := make([]byte, grow)
        copy(newBuf, s.Buf)
        s.Buf = newBuf
        s.Free = grow - s.Len - needed
    }
    copy(s.Buf[s.Len:], p)
    s.Len += needed
    // free 不变（已包含在新扩展中）
}
```

---

## 2. Hash：ziplist vs hashtable

### ziplist（压缩列表）

适用条件（两个都满足才用 ziplist）：
- 字段数量 `< hash-max-ziplist-entries`（默认 512）
- 所有字段值长度 `< hash-max-ziplist-value`（默认 64 字节）

```c
// ziplist 内存布局：连续内存块，无指针开销
/*
zlbytes(4) | zltail(4) | zllen(2) | entry1 | entry2 | ... | zlend(1)
             ↑                               ↑
         总字节数                          尾部偏移
*/

// entry 结构（变长编码）：
/*
前一个entry长度（1~5字节）| 本entry长度（1~5字节）| content（内容）
*/
```

**ziplist 优点**：无指针开销，内存连续，缓存友好
**ziplist 缺点**：插入/删除 O(n)，数据量大时需要连锁更新

### hashtable（哈希表）

当字段数超过阈值时，Redis 将 Hash 编码切换为 **hashtable**：

```c
// Redis 5.0 源码：dict（字典/哈希表）
typedef struct dict {
    dictType *type;     // 哈希函数、键比较函数
    void *privdata;
    dictht ht[2];       // 两个哈希表（用于 rehash）
    long rehashidx;      // rehash 进度，-1 表示未在进行
} dict;
```

**hashtable 优势**：O(1) 单次操作，插入/查找极快
**hashtable 劣势**：每个 entry 有额外的 dictEntry 指针开销

### 编码转换触发条件

```bash
# Redis 配置
hash-max-ziplist-entries 512   # 字段数 > 512 → 转为 hashtable
hash-max-ziplist-value 64       # 任意字段值 > 64字节 → 转为 hashtable

# 查看 key 的编码类型
OBJECT ENCODING user:1001
# 输出：ziplist 或 hashtable
```

### 实战经验：Hash 节省内存

```go
// 反面教材：每个字段都存成 String（独立 key）
SET user:1001:name "zhangsan"    // "user:1001:name" → 额外 20+ 字节 key
SET user:1001:age "28"            // "user:1001:age" → 额外 key 开销
SET user:1001:city "beijing"

// 正确做法：用 Hash（多个字段共享一个 key）
HSET user:1001 name "zhangsan" age "28" city "beijing"
// 只存储一个 key "user:1001"，3个字段紧凑存在 ziplist 中
// 内存节省约 40-60%（减少 key 重复存储）
```

---

## 3. List：ziplist vs quicklist vs linkedlist

### 编码演进历史

```
Redis 3.0 之前：ziplist（<64元素）+ linkedlist（≥64）
Redis 3.2+：quicklist（默认，ziplist 的链表）
```

### quicklist（快速列表）

quicklist = **多个 ziplist 通过双向指针串联**：

```
quicklist 结构：
[ziplist A] ←→ [ziplist B] ←→ [ziplist C] ←→ ...

每个 ziplist 默认最多 8KB（可配置 list-max-ziplist-size -2）
```

**为什么不用单个大 ziplist**？→ ziplist 插入 O(n)，太大会成瓶颈
**为什么不用纯 linkedlist**？→ 每个节点有独立指针开销，内存浪费
**quicklist 折中**：每个节点是紧凑的 ziplist，节点间通过指针串联

```bash
# 配置每个 ziplist 节点的最大大小
# list-max-ziplist-size -2 = 默认 8KB 每个节点
# list-compress-depth 1 = 两端各 1 个 ziplist 不压缩（热点数据）
CONFIG SET list-max-ziplist-size -2
CONFIG SET list-compress-depth 1
```

---

## 4. Set：intset vs hashtable

### intset（整数集合）

当 Set 满足以下条件时，使用 intset（连续内存，节省空间）：
- 元素全部是整数
- 元素数量 `< set-max-intset-entries`（默认 512）

```c
// intset 内存布局
/*
encoding(4字节) | length(4字节) | contents（连续整数数组）
                 ↑ 元素个数
*/

// 编码方式（encoding 决定每个元素占多少字节）
INTSET_ENC_INT16 = 2字节（-32768 ~ 32767）
INTSET_ENC_INT32 = 4字节（全部 int32）
INTSET_ENC_INT64 = 8字节（全部 int64）

// 自动升级：添加 > 当前 encoding 范围的数时，整体升级
// 只能升级不能降级！
```

**intset 优势**：内存极紧凑（无指针），缓存友好
**intset 劣势**：插入/删除需要 O(n) 挪动元素，只能存整数

### intset 升级示例

```
初始：encoding=INT16, contents=[1, 2, 3]
添加 40000 → 超出 INT16 范围
升级：encoding=INT32, contents=[1, 2, 3, 40000]
```

### 节省内存实战：合理使用 intset

```go
// 反面：存储 user:1001:tags 为 Set，存了大量字符串标签
SADD user:1001:tags "golang" "redis" "mysql" "kafka"
// 如果标签很多（>512个）→ hashtable，内存开销大

// 正面：如果标签是数字 ID，用 intset 存储
SADD item:tags:1001 1 2 3 4 5 6
// <512 个整数 → intset，极省内存
// 一个 intset 存 512 个 int64 = 4KB
// 而 hashtable 存 512 个字符串 key ≈ 50KB+
```

---

## 5. Zset：ziplist vs skiplist + dict

### ziplist（有序列表编码）

当元素数量 `< zset-max-ziplist-entries`（默认 128）时使用：
- 紧凑连续内存，无指针开销
- 内存占用约为 skiplist 的 **1/3~1/5**

```bash
# ziplist 编码的 Zset：按分数排序，元素和分数交替存储
ZRANGE zset:1 0 -1 WITHSCORES
# 存储：[elem1, score1, elem2, score2, ...]
```

### skiplist + dict（跳表 + 字典）

当元素数量超过阈值时使用：

```c
// Redis Zset 底层结构
typedef struct zset {
    dict *dict;       // 字典：member → score，O(1) 查找分数
    zskiplist *zsl;   // 跳表：按分数排序，O(log n) 按序遍历
} zset;
```

**为什么同时用两个数据结构？**

| 操作 | dict | skiplist |
|------|------|----------|
| ZRANGE（按序遍历） | O(n) 太慢 | **O(log n)** 极快 |
| ZSCORE（查某成员分数） | **O(1)** 极快 | O(n) 太慢 |
| ZADD（添加成员） | O(1) | O(log n) |
| ZREM（删除成员） | O(1) | O(log n) |

两种数据结构**共享同一份数据**（member 和 score），修改时同步更新。

### 跳表原理

```
跳表层（简化）：

Level 3:  ──────────────────────────────→ [∞]
Level 2:  ───────────→ [10]──────────────→ [∞]
Level 1:  ──→ [5]────→ [10]────→ [20]───→ [∞]
Level 0:  →[3]→[5]→[8]→[10]→[15]→[20]→[25]→ [∞]

Level 0：原始链表，有序
Level 1：每隔 1 个节点抽一个
Level 2：每隔 2 个节点抽一个
Level 3：每隔 4 个节点抽一个

查找：从高层向低层查找，时间复杂度 O(log n)
```

### Zset 内存分析

```go
// Zset 内存对比：10000 个元素

// ziplist 编码（<128 元素时）
// 每个 entry ≈ 40 字节（member + score + 指针）
// 10000 元素 × 40B = 400KB

// skiplist + dict 编码
// dictEntry ≈ 72字节（key指针+value+next+编码） × 10000 = 720KB
// skiplistNode ≈ 32字节（header+tail+level[16]） + 数据
// 总计 ≈ 1.5MB+

// 结论：元素少时 ziplist 省内存，元素多时 skiplist 省时间
```

---

## 6. Bitmap：位图

Bitmap 不是独立数据类型，是 String 的位操作扩展。

```go
// 场景：用户每日签到（365天）
// 普通做法：SADD user:1001:sign:2026 1 2 3 ... 365  → 365 个元素
// Bitmap 做法：SETBIT user:1001:sign:2026 day-1 1     → 1 bit/天

// 设置：2026 年 1 月 1 日签到（day=0，即 bit 位置 0）
SETBIT user:1001:sign:2026 0 1

// 检查：2026 年 1 月 5 日是否签到
GETBIT user:1001:sign:2026 4

// 统计 2026 年 1 月签到次数（bit 0-30，共 31 天）
BITCOUNT user:1001:sign:2026 0 30
```

### Bitmap 内存计算

```
一年 365 天 = 365 bits = 46 bytes ≈ 0.05KB

对比普通 String：
SADD user:sign:2026 1 2 3 ... 365 → 365 × 20~50字节 ≈ 10~20KB

节省内存：约 200~400 倍！
```

### 实际应用场景

| 场景 | Bitmap 使用方式 |
|------|----------------|
| 用户每日签到 | `SETBIT user:sign:{year} {dayOfYear} 1` |
| 统计 DAU（日活） | 每日一个 key，`BITOP OR` 合并多天 |
| 用户在线状态 | `SETBIT online:{minute} {userID} 1` |
| 布隆过滤器 | 多哈希函数的 bitmap |

---

## 编码转换规则汇总

```
Hash:
  ziplist → hashtable：字段数 > 512 或 某字段值 > 64B
  
List:
  ziplist → quicklist：元素数 > 64 或 某元素 > 64B

Set:
  intset → hashtable：元素数 > 512 或 某元素非整数

Zset:
  ziplist → skiplist：元素数 > 128 或 某member > 64B

编码转换：自动触发，只能从节省内存的编码转为高性能编码
         只能升级不能降级（除非手动执行 OBJECT ENCODING 重建）
```

---

## 面试高频追问

**Q1: Redis 的 dict 和 Go 的 map 有什么区别？**
→ Redis dict 是自己实现的哈希表，有 rehash 机制（渐进式rehash），保证查询稳定 O(1)。Go 的 map 也有扩容机制，但 Redis 的 rehash 是双哈希表渐进式迁移，更适合长连接场景。

**Q2: ziplist 什么时候会变成 hashtable？**
→ ziplist 是内存紧凑结构，但操作是 O(n)。当字段数超过 512（或某字段值超过 64 字节）时，Redis 自动转为 hashtable 以保证 O(1) 操作性能。这是内存和时间平衡的结果。

**Q3: 为什么 Zset 同时用 skiplist 和 dict？**
→ skiplist 保证按分数排序遍历 O(log n)，dict 保证按 member 查分数 O(1)。两者共享数据，内存开销约翻倍，但性能最优。实际可以只用 skiplist（或只用 dict），Redis 选择了同时使用的折中方案。
