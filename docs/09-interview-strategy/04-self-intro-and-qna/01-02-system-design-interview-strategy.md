# 系统设计面试：从思路到落地的完整策略

> 考察频率：★★★★★（高级岗必考）  难度：★★★★☆
> 适用：技术二面/三面，考察架构设计能力

---

## 核心答案（30 秒版）

系统设计面试题没有"标准答案"，考察的是**系统化思考 + 权衡取舍 + 沟通能力**。用五步框架：**明确需求 → 估算规模 → 高层架构 → 组件详细设计 → 扩展优化**。每个步骤都要和面试官保持互动，别一个人闷头讲 20 分钟。Go 工程师的系统设计题常围绕：**高并发服务、消息队列方案、缓存架构、分布式 ID 生成**。

---

## 1. 通用五步法

### Step 1：明确需求（2-3 分钟）

```
不要一上来就画图！先搞清楚边界：

功能需求：
- 这个系统要做什么？（用户故事）
- 哪些是必须有的（MVP），哪些可以后续加？

非功能需求：
- QPS / 日活量级？
- 一致性要求？（强一致 vs 最终一致）
- 可用性目标？（99.9% vs 99.99%）
- 数据增长趋势？
- 延迟要求？

关键追问示例：
"这个接口要求的 P99 延迟是多少？"
"如果流量突然翻 10 倍，能接受降级吗？"
"读多还是写多？大概的比例？"
```

### Step 2：规模估算（3-5 分钟）

```
这是区分初级和中高级工程师的关键！

计算模板：

假设：日均请求 1 亿次，每次请求返回 2KB 数据

① QPS：1亿 / (24*3600) ≈ 1157 QPS，考虑峰值系数 10x → 约 1.2 万 QPS
② 存储量：每条记录 1KB × 1亿 = 100GB/天 → 一年约 36.5TB
③ 带宽：2KB × 1亿 = 200GB/天 → 约 23MB/s 平均带宽
④ Redis 缓存命中率预估 90%，需要缓存的数据量 ≈ 20GB

结论：
- QPS 不高但存储量大 → 需要考虑数据库水平分片
- 数据增长快 → 冷热分离设计
```

### Step 3：高层架构（2-3 分钟）

```
画出一张包含核心组件的架构图，不追求细节：

Client → Load Balancer → API Gateway → [Service A] [Service B]
                              ↕           ↕
                         Redis Cluster  MySQL Sharded DB
                              ↕
                          Message Queue
                              ↓
                        [Data Pipeline]

边画边说：为什么选择这些组件，不用那些组件的理由是什么。
```

### Step 4：组件详细设计（8-10 分钟）

```
选 2-3 个核心组件深入讨论，而不是每块都浅尝辄止：

重点展开方向：
- 数据库表结构设计 + 索引策略
- 缓存设计与失效策略
- 服务间通信方式（REST/gRPC/MQ）
- 分布式锁或一致性方案
- Go 代码层面的关键实现
```

### Step 5：扩展与优化（3-5 分钟）

```
主动提出扩展点，展现前瞻性：

- "如果流量再翻 10 倍，瓶颈会出现在哪？怎么解决？"
- "单点故障怎么处理？"
- "数据迁移时如何做到零停机？"
- "如何监控和告警？"
```

---

## 2. Go 工程师高频系统设计真题

### 【🔥🔥🔥🔥🔥】短链接生成系统

**需求**：设计一个短链接服务，支持将长 URL 转为短码，访问时重定向。

```
✅ Go 工程师参考答案要点：

【需求澄清】
- QPS 预估：日均 100 万次请求，峰值可能 5 万 QPS
- 写入少读多，读写比约 1:1000
- 短码长度：6位 Base62 编码可覆盖 57 亿条

【方案选择】
- 不使用查表法（百万级短码映射表查询太慢）
- 使用自增 ID + Base62 编码转换（最简洁）
- Go 实现：strconv.FormatInt(id, 62)

【Redis 方案】
1. 用 Redis INCR 生成全局唯一 ID（单机 1 亿次/秒足够）
2. 将 id→short_code 缓存到 Redis Hash
3. 读取时用 short_code 反向解出 id，再查 DB 获取原 URL

【Go 代码关键片段】
// 自增 ID → 短码转换
func IDToShortCode(id int64) string {
    const chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_"
    var result []byte
    for id > 0 {
        result = append(result, chars[id%62])
        id /= 62
    }
    // reverse
    for i, j := 0, len(result)-1; i < j; i, j = i+1, j-1 {
        result[i], result[j] = result[j], result[i]
    }
    return string(result)
}

【扩展问题】
- Q：高并发下 INCR 会不会冲突？
  A：Redis INCR 是原子操作，不会冲突。如果需要更高吞吐量，可以用 Redis Slot 分段预分配。
- Q：如何防止短码被遍历猜测？
  A：使用随机盐值 + hash 混淆 ID，或在生成时加时间戳哈希。
```

### 【🔥🔥🔥🔥】分布式限流器设计

**需求**：设计一个支持多租户的分布式限流器。

```
✅ Go 工程师参考答案要点：

【方案选择】
- 令牌桶算法（Token Bucket）：允许突发流量，适合大多数场景
- 滑动窗口计数：更精确但实现复杂

【Redis + Lua 实现】
用 Lua 脚本保证原子性：

local key = KEYS[1]
local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = 1

-- 获取当前 token 数
local tokens = redis.call('get', key)
if not tokens then
    redis.call('set', key, capacity)
    redis.call('expire', key, 2)
    return 1
end

if tonumber(tokens) >= requested then
    redis.call('decrby', key, requested)
    return 1  -- 通过
else
    return 0  -- 拒绝
end

【Go 中的实践】
- 本地令牌桶作为一级限流（减少 Redis 调用）
- Redis 作为二级限流（保证全局一致性）
- Go 1.21+ 可用 golang.org/x/time/rate 做本地限流
```

### 【🔥🔥🔥】秒杀系统设计

**需求**：设计一个支持万人同时下单的秒杀系统。

```
✅ Go 工程师参考答案要点：

【核心思路：层层拦截 + 异步处理】

第一层：前端
- 按钮防抖 + 倒计时，减少无效请求
- JS 客户端限流

第二层：CDN / WAF
- 静态资源走 CDN
- IP 黑名单，同一 IP 只允许 N 次请求

第三层：Go 网关（本地缓存 + 限流）
- 基于 localcache 的本地令牌桶
- 恶意 IP 自动封禁

第四层：Redis 预扣库存
- 活动开始前将库存预热到 Redis
- 使用 Lua 脚本原子扣减
- Redis 扣减成功后发送 MQ 消息

第五层：MQ 异步创建订单
- RocketMQ/Kafka 削峰填谷
- 消费者按顺序处理，保证库存不超卖

【关键 Go 实现】
// 预扣库存 Lua 脚本
local stock_key = 'seckill:stock:' .. KEYS[1]
local user_key = 'seckill:user:' .. KEYS[1] .. ':' .. ARGV[1]

if redis.call('exists', user_key) == 1 then
    return -1  -- 重复购买
end

local stock = redis.call('get', stock_key)
if stock and tonumber(stock) > 0 then
    redis.call('decr', stock_key)
    redis.call('set', user_key, '1')
    return 1  -- 扣减成功
end
return 0  -- 库存不足
```

---

## 3. 面试实战技巧

### 正确姿势

| 阶段 | 动作 |
|------|------|
| 听到题目后 | 不要急着回答，问清楚需求和约束条件 |
| 开始设计时 | 先画框图，再说细节 |
| 讨论中 | 不断确认："我这样的理解对吗？" |
| 遇到盲区 | 诚实说"这块我不熟悉"，然后尝试推理而非瞎编 |
| 结束前 | 主动问还有没有其他角度可以考虑 |

### 常见错误

```
❌ 还没理解需求就开始设计 → 浪费时间和机会
❌ 只设计方案不讲原因 → 像背答案
❌ 回避 Tradeoff → 系统设计没有银弹，不谈取舍就是没想清楚
❌ 代码细节花太多时间 → 系统设计不是手撕代码
❌ 被带偏方向 → 面试官可能会故意设置陷阱，保持主线思维
```

---

## 延伸阅读

- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [Grokking the System Design Interview](https://www.educative.io/courses/grokking-the-system-design-interview)
