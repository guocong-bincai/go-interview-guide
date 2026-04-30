# 分布式锁横向对比：Redis vs etcd vs ZooKeeper

> 考察频率：★★★★★  优先级：P0

## TODO（待填写）

## 1. 三种方案原理速览

### Redis 分布式锁
- [ ] `SET key value NX PX ttl` 加锁原子操作
- [ ] Lua 脚本解锁（判断 value + DEL 原子化）
- [ ] 续期（Watchdog 机制，Redisson 实现）
- [ ] Redis 单点 vs Redlock（多节点）

### etcd 分布式锁
- [ ] 基于 Lease + 有序 Key（`/lock/prefix/seq`）
- [ ] 公平锁：Watch 前一个 key，有序排队
- [ ] Lease 到期自动释放，基于 Raft 线性一致，不存在脑裂

### ZooKeeper 分布式锁
- [ ] 临时有序节点（EPHEMERAL_SEQUENTIAL）
- [ ] Watch 前一个节点实现公平队列
- [ ] Session 超时自动删除节点

## 2. 核心维度对比表
| 维度 | Redis | etcd | ZooKeeper |
|------|-------|------|----------|
| 一致性 | 最终（单点）/ 弱（Redlock） | 强（线性一致） | 强（ZAB） |
| 性能 | 最高（10W+ QPS） | 中（1W QPS） | 中（1W QPS） |
| 脑裂风险 | 有（网络分区时） | 无 | 无 |
| 公平性 | 否（需额外实现） | 是（原生） | 是（原生） |
| 运维复杂度 | 低 | 中 | 高 |
| Go 生态 | go-redis | clientv3 | go-zookeeper |

## 3. Redlock 争议（面试加分点）
- [ ] Martin Kleppmann 的质疑：时钟漂移导致多节点同时持锁
- [ ] Antirez 的反驳：实践中时钟漂移概率极低
- [ ] 结论：Redlock 在大多数场景够用，金融/强一致场景用 etcd

## 4. 选型决策树
- [ ] 高性能 + 允许极小概率失效 → Redis 单节点锁
- [ ] 强一致 + 中等性能 → etcd
- [ ] 历史系统已有 ZK + Java 生态 → ZooKeeper（不建议新项目）
- [ ] 秒杀/库存扣减 → Redis Lua 脚本（不需要分布式锁）

## 5. 完整 Go 代码示例（三套）
- [ ] Redis 分布式锁（go-redis + Lua）
- [ ] etcd 分布式锁（clientv3 + Lease）
- [ ] 压测对比数据

## 高频追问
- [ ] Redis 锁的 TTL 设多少合适？太短太长分别会怎样？
- [ ] 加锁成功但业务执行超过 TTL，锁自动释放了怎么办？
- [ ] 为什么 etcd 锁不会脑裂，Redis 锁会？
