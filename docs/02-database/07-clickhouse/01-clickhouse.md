# ClickHouse 核心原理与 OLAP 选型

> 考察频率：★★★☆☆  优先级：P1

## TODO（待填写）

## 1. 列式存储原理
- [ ] 行存 vs 列存：OLAP 场景下列存为什么快（IO 少、压缩率高、向量化友好）
- [ ] ClickHouse 数据组织：Part → Column → Mark
- [ ] 压缩编码：Delta/DoubleDelta/Gorilla/ZSTD

## 2. MergeTree 引擎族（核心）
- [ ] `MergeTree`：基础引擎，主键索引（稀疏索引）
- [ ] `ReplacingMergeTree`：去重（异步，不保证实时）
- [ ] `SummingMergeTree`：预聚合
- [ ] `AggregatingMergeTree`：精确聚合
- [ ] 主键 vs ORDER BY 的区别
- [ ] Partition 分区设计（按月/天）

## 3. 向量化执行
- [ ] SIMD 指令（SSE/AVX）批量处理列数据
- [ ] 为什么 ClickHouse 单核性能远超 MySQL
- [ ] 多核并行：一个 SELECT 自动利用所有 CPU

## 4. 与其他 OLAP 选型对比
| 维度 | ClickHouse | TiFlash | Apache Doris | ES |
|------|-----------|---------|-------------|-----|
| 写入延迟 | 中（批量好） | 低 | 中 | 低 |
| 查询性能 | 极高 | 高 | 高 | 中 |
| SQL 兼容 | 高 | 高 | 高 | 低 |
| 实时性 | 秒级 | 秒级 | 秒级 | 秒级 |

## 5. 典型使用场景
- [ ] 日志分析（替代 ELK 中 ES 的分析角色）
- [ ] 监控指标存储（替代 Prometheus 长期存储）
- [ ] 用户行为分析（漏斗/留存/路径）
- [ ] 广告效果统计（实时 CTR/CVR）

## 6. Go 接入实战
- [ ] clickhouse-go v2 SDK 使用
- [ ] 批量写入最佳实践（`InsertBatch`）
- [ ] 连接池配置

## 高频追问
- [ ] ClickHouse 为什么不适合高并发点查？
- [ ] `ReplacingMergeTree` 删除数据为什么不是立即生效的？
- [ ] Kafka → ClickHouse 的实时数据管道怎么设计？
