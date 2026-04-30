# 连接池原理与调优：database/sql、Redis、gRPC

> 考察频率：★★★★★  优先级：P0

## TODO（待填写）

## 1. 连接池的本质
- [ ] 为什么需要连接池（TCP 握手 + TLS + 数据库认证耗时 ~50ms）
- [ ] 连接池核心参数：最大连接数、最大空闲数、连接最大生命周期、空闲超时
- [ ] 连接池状态机：idle → inuse → closed

## 2. database/sql 连接池（Go 重点）
- [ ] `MaxOpenConns`：最大打开连接数（超过则等待）
- [ ] `MaxIdleConns`：最大空闲连接数（多余连接关闭）
- [ ] `ConnMaxLifetime`：连接最长生命周期（防数据库踢掉空闲连接）
- [ ] `ConnMaxIdleTime`：空闲连接超时（Go 1.15+）
- [ ] 生产推荐配置值及计算公式：`MaxOpenConns = DB最大连接数 / 实例数`
- [ ] 完整 Go 代码示例：配置 + 监控 `db.Stats()`

## 3. 连接泄漏排查（P0 高频问题）
- [ ] 症状：`db.Stats().InUse` 持续增长，`WaitCount` 告警
- [ ] 根因一：忘记 `rows.Close()` 或 `stmt.Close()`
- [ ] 根因二：事务未提交/回滚（`defer tx.Rollback()`）
- [ ] 根因三：`QueryRow` 后未调用 `Scan`
- [ ] 排查 SOP：pprof goroutine dump → 找阻塞在 `(*DB).conn` 的 goroutine

## 4. Redis 连接池（go-redis）
- [ ] `PoolSize`：连接池大小（默认 CPU 核数 × 10）
- [ ] `MinIdleConns`：最小空闲连接（预热）
- [ ] `PoolTimeout`：等待连接超时（区别于命令超时）
- [ ] 连接健康检查：PING 心跳配置

## 5. gRPC 连接复用
- [ ] gRPC 基于 HTTP/2，天然多路复用，一个连接承载所有请求
- [ ] 什么时候需要多个连接（负载均衡 pick_first vs round_robin）
- [ ] `ClientConn` 的生命周期管理

## 6. 连接池满了怎么办（生产处理链路）
- [ ] 现象：响应超时、`context deadline exceeded`
- [ ] 处理：熔断 → 降级（走缓存）→ 扩容（加 DB 实例/读写分离）

## 高频追问
- [ ] `MaxOpenConns` 设太大会怎样？设太小会怎样？
- [ ] Go 的 `database/sql` 连接池是并发安全的吗？
- [ ] 为什么 Redis 连接池满比 MySQL 连接池满更危险？
