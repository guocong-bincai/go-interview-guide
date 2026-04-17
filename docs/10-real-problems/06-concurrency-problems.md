# 并发编程实战问题

> 考察频率：★★★★★  优先级：P0

## TODO（待填写）

## 1. goroutine 池设计
- [ ] 为什么需要 goroutine 池（控制并发数，防止资源耗尽）
- [ ] 核心结构：任务队列 + worker goroutine + 动态扩缩容
- [ ] 实现一个简单的 goroutine 池（完整 Go 代码）
- [ ] ants 库的核心设计，什么场景用

## 2. 并发安全的本地缓存
- [ ] sync.Map 适合什么场景（读多写少）
- [ ] 分片锁（sharding map）实现：16/32 个分片，减少锁粒度
- [ ] 带 TTL 的缓存实现（定时清理 vs 懒删除）
- [ ] 完整 Go 代码示例

## 3. 优雅关闭（Graceful Shutdown）
- [ ] 为什么需要优雅关闭（K8s rolling update，不丢请求）
- [ ] 信号处理：SIGTERM / SIGINT
- [ ] WaitGroup 等待所有请求处理完
- [ ] 超时强制退出（给 30 秒，超了就强关）
- [ ] 完整 Go 代码示例

## 4. 并发任务编排
- [ ] errgroup：并发执行多任务，任一失败全部取消
- [ ] semaphore：控制最大并发数
- [ ] pipeline 模式：多阶段流水线处理
- [ ] 完整 Go 代码示例
