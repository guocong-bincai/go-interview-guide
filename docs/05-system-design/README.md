# 05 · 系统设计

> 考察频率：★★★★★  优先级：P0

## 文章清单

### 01-patterns · 架构模式
- [✅] [CQRS 模式：读写分离、Event Sourcing 结合](01-patterns/06-01-cqrs.md)
- [✅] [事件驱动架构、Outbox Pattern](01-patterns/07-02-event-driven.md)
- [✅] [Saga 在业务中的落地](02-scenarios/09-01-saga.md)
- [✅] [DDD 战略设计：领域、限界上下文、聚合根](01-patterns/01-03-ddd.md)
- [⏳] [DDD 战术设计：实体、值对象、聚合、领域服务、Repository](01-patterns/02-05-ddd-tactical-design.md)
- [⏳] [业务建模方法论：Event Storming、核心域/支撑域、服务边界](01-patterns/08-06-business-modeling.md)

### 02-scenarios · 高频设计题
- [✅] [秒杀系统设计：预减库存、异步下单、防超卖](01-seckill/03-01-seckill.md)
- [✅] [短链系统：发号器、跳转、高可用](02-short-url/19-02-short-url.md)
- [✅] [IM 系统：消息投递、离线消息、已读未读](02-scenarios/10-02-im.md)
- [✅] [Feed 流：推模式 vs 拉模式 vs 推拉结合](04-feed/04-04-feed.md)
- [✅] [分布式 ID：Snowflake、Leaf、UUIDv7 对比](05-distributed-id/13-05-distributed-id.md)
- [✅] [限流系统：令牌桶、滑动窗口、分布式限流](02-scenarios/10-02-im.md)
- [✅] [搜索系统：分词、倒排索引、搜索建议](07-search/15-07-search.md)
- [✅] [支付系统：幂等、对账、资金安全](08-payment/16-08-payment.md)

### 03-cache · 缓存架构
- [✅] [缓存与数据库一致性策略 + 多级缓存架构（Cache-Aside/Write-Through/延迟双删）](01-cache-consistency/01-01-cache-consistency.md)

### 04-cdn · CDN 架构
- [✅] [CDN 架构设计：回源策略、缓存失效与热点分发（PULL/PUSH/预热/BFCache）](04-cdn/01-01-cdn-architecture.md)

### 06-api-gateway · API 网关
- [✅] [API 网关设计：鉴权/路由/限流/熔断一体化（JWT/动态路由/灰度发布）](06-api-gateway/01-01-api-gateway.md)

### 07-websocket · 长连接架构
- [✅] [WebSocket 长连接架构：百万级并发在线用户实战（心跳/水平扩展/离线消息）](07-websocket/01-01-websocket-scale.md)

### 08-file-upload · 文件传输
- [✅] [大文件上传/下载系统设计：分片/断点续传/秒传（MD5指纹/Ranger Header）](08-file-upload/01-01-large-file-upload.md)

### 09-deep-pagination · 分页优化
- [✅] [深度分页优化：MySQL OFFSET 瓶颈与替代方案（游标分页/延迟关联/覆盖索引）](09-deep-pagination/01-01-deep-pagination.md)

### 03-capacity · 容量估算
- [✅] [信封估算法：QPS / 存储 / 带宽快速估算](03-capacity/11-01-back-of-envelope.md)
- [✅] [性能指标体系：P99/P999、可用性 SLA](03-capacity/12-02-performance-indicators.md)
