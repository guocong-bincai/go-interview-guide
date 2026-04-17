# etcd 原理与实战

> 考察频率：★★★★☆  优先级：P1

## TODO（待填写）

- [ ] etcd 架构：基于 Raft，boltdb 存储，MVCC 版本控制
- [ ] Watch 机制：如何实现配置变更推送（gRPC stream）
- [ ] 分布式锁：基于 lease + revision 实现公平锁
- [ ] 选主：基于 campaign，如何防止脑裂
- [ ] 性能上限：写 QPS 约 1 万，为什么不适合做消息队列
- [ ] 与 Zookeeper 的对比：CAP 取舍、API 易用性、Go 生态
- [ ] 高频追问：etcd 的 lease 到期但业务没处理完怎么办（续期）？
