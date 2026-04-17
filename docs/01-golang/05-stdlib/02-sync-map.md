# sync.Map 与并发安全 Map

> 考察频率：★★★★☆  优先级：P1

## TODO（待填写）

- [ ] sync.Map 底层结构（read map + dirty map 双 map 设计）
- [ ] 读写流程：为什么读不加锁（atomic.Value + read map）
- [ ] 适用场景：读多写少、key 集合稳定；不适合写多场景
- [ ] 与 map+RWMutex 的性能对比（benchmark 数据）
- [ ] 替代方案：分片锁 map（sharding）实现思路
- [ ] 高频追问：为什么 Go 原生 map 不是并发安全的？
