# Elasticsearch 写入流程、两阶段查询与深度分页（生产必考）

> 考察频率：★★★★★  难度：★★★★☆
> 关键词：refresh、translog、flush、segment merge、近实时、query phase、fetch phase、from+size、scroll、search_after、脑裂、wait_for_active_shards

## 面试官考察意图

简历写了「用 ES 做搜索/日志分析」就一定会被追问：

1. "ES 是实时的吗？为什么叫近实时？" → 引出 refresh / translog / flush 三段式写入
2. "为什么删除文档后磁盘空间不降？" → 引出 segment 不可变 + merge
3. "ES 深度分页 `from=10000` 为什么报错？" → 引出 query/fetch 两阶段与三种分页方案
4. "集群黄了/红了怎么排查？" → 引出分片分配、脑裂与脑裂防护

**区分度**：能不能把「segment 不可变」这条主线讲透——写入、查询、删除、更新、分页全部由它推导出来。

---

## 核心答案（30 秒版）

| 问题 | 一句话回答 |
|------|-----------|
| ES 是实时的吗？ | **近实时（NRT）**，默认 1s 后可见，靠 `refresh` 把内存 buffer 生成新 segment |
| 写入流程？ | 内存 buffer + translog → `refresh` 生成 segment（可被搜索）→ `flush` 落盘并清空 translog |
| 为什么删数据磁盘不降？ | segment **不可变**，删除只是打 `.del` 标记，空间要等 segment merge 后才真正释放 |
| 查询两阶段？ | 协调节点 **query phase** 收集各分片 TopN 的 docID → **fetch phase** 回各分片取 `_source` |
| 深度分页？ | `from+size` 每片要取 `from+size` 条，`from+size > 10000` 直接拒绝；用 `search_after`（推荐）或 `scroll` |
| 脑裂怎么防？ | 7.x 起由 ES 自动管理 quorum：`(master-eligible 数 / 2) + 1`，不再手配 `minimum_master_nodes` |

---

## 深度展开

### 1. 写入流程：从 client 到磁盘

```
Client ──> 协调节点(Coordinating) ──路由 hash(_routing 默认 _id)──> 主分片 P
                                                                    │
              ① 写内存 buffer（此时不可被搜索）                       │
              ② 追加写 translog（先写日志，WAL 思想，保证不丢）        │
              ③ 同步复制到副本分片 R ────────────────────────────────┘
                                                                    │
              ── 默认每 1s（refresh_interval）──                     
              ④ refresh：buffer 生成一个**新的 segment**（进入文件系统缓存）
                 → 此时该 segment 可被搜索，但还未 fsync 到磁盘 → **近实时**
                                                                    │
              ── translog 达到阈值 / 每 30min ──
              ⑤ flush：把 segment fsync 落盘 + 清空 translog
                                                                    │
              ── 后台 ──
              ⑥ merge：多个小 segment 合并成大 segment，物理删除被标记的文档
```

**三个关键点：**

1. **为什么说「近实时」而不是「实时」**：segment 只有在 `refresh` 之后才可被搜索，默认 1s 延迟。
2. **translog 的作用**：内存 buffer 里的数据在 refresh 前还没进 segment，此时宕机会丢，所以每次写都同步追加 translog，重启时用 translog 恢复。
3. **segment 不可变**：一旦生成就不能改一个字，所以「更新」= 旧文档打删除标记 + 写入新文档，「删除」= 只打 `.del` 标记。

```json
// 写入时可选：让本次写立即可见（谨慎使用，会产生大量小 segment）
PUT /idx/_doc/1?refresh=true

// 从"可见性/持久性"两个维度调优
PUT /idx/_settings
{ "index.refresh_interval": "30s" }      // 提高写入吞吐，代价是可见延迟变大

// 批量导入场景：先关 refresh 和副本，导完再开
PUT /idx/_settings
{ "index.refresh_interval": "-1", "index.number_of_replicas": 0 }
```

### 2. 查询两阶段：query phase + fetch phase

```
Client 请求: from=9900, size=100（第 99 页）
        │
        ▼
协调节点把所有分片当成"散列",每个分片都要返回 from+size = 10000 条候选 ID
        │
   Query Phase：每个分片本地排序，返回 Top 10000 个 (docID, sortValue)
        │
        协调节点归并 5 个分片的 5万条 → 全局排序 → 截取第 9900~10000 条 → 得到 100 个 docID
        │
   Fetch Phase：拿着这 100 个 docID 去对应分片取 _source
        │
        返回结果
```

**这就是 `result window is too large`（默认 `index.max_result_window=10000`）的根因**：分片数越多，协调节点归并的代价越大，内存和 CPU 都是 O(分片数 × (from+size))。

#### 三种深度分页方案对比

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| `from + size` | 每片取 `from+size` 条归并 | 简单、可跳页 | 深分页 O(分片数×(from+size))，有 1万上限 | 前几页、小数据量 |
| `scroll` | 生成快照游标，逐批拉取 | 全量遍历快 | 占用文件句柄/内存，有上下文过期时间，**不能反映实时数据** | 全量导出、离线 ETL |
| `search_after` | 用上一页最后一条的排序值作为下一页起点 | **无深度限制、实时、无状态** | 只能"下一页"，不能跳页 | 主流业务分页（推荐） |

```json
// search_after：必须搭配唯一且确定性的排序键（否则会丢/重）
GET /idx/_search
{
  "size": 100,
  "sort": [
    { "create_time": "desc" },
    { "_id": "asc" }            // 兜底保证全局唯一顺序
  ],
  "search_after": [1699999999000, "doc_9987"]
}
```

```go
// Go 里最常见的 search_after 封装
func Page(ctx context.Context, cli *elasticsearch.Client, after any) ([]Doc, []any, error) {
    body := map[string]any{
        "size": 100,
        "sort": []any{map[string]any{"create_time": "desc"}, map[string]any{"_id": "asc"}},
    }
    if after != nil {
        body["search_after"] = after
    }
    // ... 执行请求，取最后一条的 sort 值作为下一页游标返回
    return docs, lastSort, nil
}
```

### 3. 集群脑裂与分片分配

**脑裂（split brain）**：网络分区导致两个 master 同时认为自己是主，各自维护集群状态，写入互相覆盖。

```
防护机制（7.x+ 全自动，不再需要 discovery.zen.minimum_master_nodes）：
  ES 7.x 采用「类 Raft」的选举：候选主节点需要拿到 (master-eligible / 2) + 1 票才能当选。
  例如 3 个 master-eligible：需要 2 票；2 个：需要 2 票（等于没有容错，所以要用奇数个）。
```

**生产配置基线：**

```
master-eligible 节点数：至少 3 个且为奇数（专用 master 节点，内存 4~8G 即可）
node.roles: [master]              # 专用主节点，不存数据不查询
discovery.seed_hosts: [...]       # 7.x 初始选举种子（仅集群首次启动必需）
cluster.initial_master_nodes: ... # 只用于首次 bootstrap，之后必须删除，否则可能重新组集群
```

**集群健康色排查：**

| 颜色 | 含义 | 常见原因 | 处理 |
|------|------|----------|------|
| green | 所有主副分片正常 | - | - |
| yellow | 主分片正常，**副本未分配** | 单节点集群、副本数 > 数据节点数 | 加节点或 `number_of_replicas: 0` |
| red | **有主分片未分配** | 节点宕机、磁盘水位超限、分片损坏 | `GET _cluster/allocation/explain` 定位 |

```bash
# red/yellow 第一步一定是看分配失败原因
GET _cluster/allocation/explain
GET _cat/indices?v&health=yellow
GET _cat/shards?v | grep UNASSIGNED

# 磁盘水位触发的只读（常见生产事故）
GET _cat/allocation?v
# low(85%) → 不再分配新分片；high(90%) → 尝试迁走；flood(95%) → 索引置为只读
```

### 4. 写入可靠性与一致性（Go 生产配置）

```go
// 关键参数：一次写入要不要等足够的活跃分片
// wait_for_active_shards=1（默认主分片即可）/ "all" / 数字
// 一致性语义：ES 只提供"写多数派成功"的弱保证，不提供读已提交级别的事务语义

// 生产建议：批量写 + 不强制 refresh + 用响应中的 failures 字段判断部分失败
res, err := cli.Bulk(ctx, &buf)
if err != nil { return err }
var br esapi.BulkResponse
json.Unmarshal(res.Body, &br)   // 注意：HTTP 200 也可能含逐条 failures
if br.Errors {
    // 处理 429（限流）/ 409（版本冲突）等
}
```

**高频踩坑：**

- **不要 `refresh=true` 逐条写**：每次 refresh 都生成新 segment，segment 过多导致查询变慢、merge 压力大。批量导入用 `refresh=-1`，导入完手动 `_refresh`。
- **translog durability**：默认 `request`（每次请求 fsync），提高吞吐可改 `async`，代价是宕机可能丢最近约 5s 数据（`sync_interval`）。
- **`429 TOO_MANY_REQUESTS`**：写队列满，需要降速 + 扩容 + 缩短单个 bulk 大小。
- **版本冲突 409**：并发更新同一文档，用 `if_seq_no` / `if_primary_term` 做乐观锁。

---

## 高频追问

### Q1：ES 更新文档是「原地修改」吗？为什么更新比 MySQL 慢？

不是。`_update` 的真实流程是：查出旧文档 → 内存里合并字段 → **把旧文档标记删除** → 写入一条**全新文档**。因此一次更新 = 一次「写 + 逻辑删」，成本远高于 MySQL 的原地页内更新。这也是为什么 ES 不适合做频繁更新的主业务存储。

### Q2：为什么删除后 `_cat/indices` 的 `store.size` 不下降？

因为 segment 不可变，删除只在 `.del` 文件打标记，`_cat/indices` 统计的是 segment 的物理大小。只有后台 merge 把旧 segment 合并时才会真正丢弃被标记的文档。可以 `POST /idx/_forcemerge?max_num_segments=1` 手动触发（**只对只读索引做**，force merge 是重 I/O 操作）。

### Q3：单分片大小和分片数量怎么定？

经验值：**单分片 10~50GB**（搜狗/ES 官方建议 20~40GB 是常见基线），分片数 = 数据总量 / 单分片目标大小，再对齐数据节点数的整数倍。分片过多会带来：集群状态膨胀、查询协调开销变大、merge 线程争抢。日志类场景可配合 ILM（索引生命周期）滚动 + 冷热分层。

### Q4：ES 和 MySQL 双写一致性怎么保证？

推荐**不让业务双写**：以 MySQL 为唯一事实源，用 **Canal / Debezium 订阅 binlog** 异步写 ES；线上查询走 ES，写后立即读走 MySQL（读写分离兜底）。若必须双写，则用本地消息表 / 事务消息保证「MySQL 成功 → MQ → ES」，并做对账补偿。

---

## 延伸阅读

- [Elasticsearch 官方文档 - Near real time](https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html)
- [Elasticsearch 官方文档 - Paginate search results](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html)
- [Elasticsearch 官方文档 - Cluster allocation explain](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-allocation-explain.html)
