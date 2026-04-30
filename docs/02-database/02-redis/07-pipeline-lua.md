# Redis Pipeline 与 Lua 脚本

> 考察频率：★★★★☆  优先级：P1

## TODO（待填写）

## 1. Pipeline 原理
- [ ] RTT（Round Trip Time）是 Redis 性能瓶颈的根源
- [ ] Pipeline 批量发送命令，一次性读取所有响应
- [ ] Pipeline vs 逐条执行：RTT 从 N 次降到 1 次，吞吐量提升示例（benchmark 数据）
- [ ] Pipeline 的局限：命令无原子性、中间命令失败不回滚
- [ ] Go 中使用 `go-redis` 的 Pipeline 代码示例

## 2. Lua 脚本原子性
- [ ] 为什么 Lua 是原子的：Redis 单线程执行，Lua 执行期间不处理其他命令
- [ ] `EVAL` vs `EVALSHA`（脚本缓存）
- [ ] 典型场景：原子性 CAS（先 GET 再 SET 的竞态问题用 Lua 解决）
- [ ] 超卖问题的 Lua 实现（完整 Go 代码）
- [ ] 分布式限流的 Lua 实现（令牌桶 / 滑动窗口）

## 3. MULTI/EXEC 事务 vs Lua 对比
| 维度 | MULTI/EXEC | Lua |
|------|-----------|-----|
| 原子性 | 部分（命令错误不回滚） | 完全原子 |
| 错误处理 | 单条命令失败其他继续 | 脚本中止 |
| 性能 | 较差（一来一回） | 较好 |
| 适用场景 | 简单批量 | 复杂条件判断 |

- [ ] Redis 事务「不支持回滚」的根本原因（设计取舍：性能 vs 一致性）
- [ ] WATCH 乐观锁机制：CAS 的 Redis 实现

## 4. 生产注意事项
- [ ] Lua 脚本执行时间过长（> lua-time-limit 5000ms）会阻塞 Redis
- [ ] Cluster 模式下 Lua 的限制：所有 key 必须在同一 slot（用 `{tag}` 强制分配）
- [ ] 脚本调试：`redis-cli --ldb` 本地调试模式

## 高频追问
- [ ] Redis 事务与数据库事务的本质区别？
- [ ] 为什么 Redis Cluster 下 Lua 脚本受限？
- [ ] Pipeline 和 Lua 能组合用吗？
