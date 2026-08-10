# 全链路压测实战：影子流量与数据隔离

> 考察频率：★★★★★  优先级：P0
> 关键词：影子流量、数据隔离、影子库、流量标记、全链路压测平台

---

## 面试官考察意图

这道题考的是**你有没有做过生产级别的全链路压测**。单接口压测人人都会，但能说出"影子流量"、"数据隔离"、"流量录制回放"这些词的，才是真正做过全链路压测的人。面试官想知道：

- 你能否设计一个**不污染生产数据的压测方案**
- 你是否理解**全链路压测与单接口压测的本质区别**
- 你有没有踩过全链路压测的坑（数据污染、压垮下游、监控告警风暴）

**高级工程师的回答** vs **初级工程师的回答**：

| 维度 | 初级 | 高级 |
|------|------|------|
| 数据隔离 | "压测数据用 test_ 前缀" | 影子库/影子表/Mirror流量完整隔离 |
| 流量构造 | "手动发请求" | 流量录制 + 回放，或流量模拟平台 |
| 下游保护 | 不考虑 | 压测流量打标记，下游自动降级 |
| 监控 | 看错误率就行 | 全链路 Trace + 关键节点耗时拆分 |
| 报告 | "压到5000QPS了" | SLA边界、扩容曲线、风险点清单 |

---

## 核心答案（30秒版）

全链路压测的核心是**流量隔离**：用镜像流量或录制回放的方式，把压测请求的读写全部打到影子环境，数据不污染生产。关键要处理好 MySQL 影子库、Redis 影子实例、Kafka 影子 Topic，以及下游服务的降级策略。压测时全链路打标记，监控里单独看"压测流量"的指标，避免告警风暴。

---

## 深度展开

### 一、为什么全链路压测和单接口压测完全不同

单接口压测只能验证**单个服务的容量**，但生产问题是**链路上的木桶效应**：

```
用户请求 → Gateway → Auth → Order → Payment → DB
                ↓        ↓      ↓        ↓
             限流熔断  超时   锁冲突   慢查询
```

单接口压测 Order 服务 5000 QPS 没问题，但加上 Auth 的 Token 校验 + Payment 的 DB 写入，整条链路可能只能跑到 800 QPS。

**全链路压测要解决三个问题**：
1. 如何构造真实流量（不是手动发请求）
2. 如何隔离压测数据（不污染生产 DB/Redis）
3. 如何观察整条链路的瓶颈（不是只看单个服务）

### 二、流量构造方案

#### 方案一：流量录制与回放（最真实）

```bash
# 用 TCPCopy 录制生产流量
tcpcopy -i <原始连接> -t <压测目标> -x <端口映射>

# 或者用 GoReplay
gor.exe -input-raw ":8080" -output-http "http://staging.example.com"
```

**优点**：流量最真实，覆盖所有边界 case  
**缺点**：需要额外机器做录制，流量回放是倍增录制量，可能不准

#### 方案二：流量模拟平台（最可控）

推荐自建或用开源平台（如 [Locust](https://locust.io/) 分布式部署）：

```python
# locustfile.py - 用户行为建模
from locust import HttpUser, task, between

class OrderUser(HttpUser):
    wait_time = between(1, 3)  # 模拟真实用户操作间隔
    token = None

    def on_start(self):
        # 登录获取 token
        resp = self.client.post("/api/v1/login", json={"user": "load_test", "pwd": "xxx"})
        self.token = resp.json()["token"]
        self.client.headers.update({"Authorization": f"Bearer {self.token}"})

    @task(3)
    def create_order(self):
        # 80% 读 + 20% 写，符合真实比例
        self.client.post("/api/v1/orders", json={"items": [...]})

    @task(1)
    def query_order(self):
        self.client.get("/api/v1/orders/123")
```

```bash
# 分布式部署压测集群
locust -f locustfile.py \
  --master \
  --expect-workers=10 \
  --headless \
  -u 5000 -r 100 --run-time 30m \
  --csv=/tmp/orders_report
```

#### 方案三：历史流量放大（生产峰值回放）

```bash
# 从监控系统导出历史流量曲线
# 按峰值时段的流量比例放大 1.5x、2x、3x

# 用 jq 处理 JSON 格式的流量日志
cat traffic_log.json | jq '.[] | select(.ts > 1700000000)' \
  | riot -p 2 -r http://staging.example.com
```

### 三、数据隔离方案（核心）

#### 1. MySQL 影子库

```sql
-- 生产库：orders
-- 影子库：orders_shadow（结构完全一样，数据是压测前复制的子集）

-- 压测请求打到影子库（应用层按 header 路由）
-- Spring: @Transactional(rollbackFor = Exception.class)
-- Go: 用 GORM 的 DSN 动态切换

func getDSN(ctx context.Context) string {
    if isShadowTraffic(ctx) {
        return os.Getenv("DB_DSN_SHADOW")  // orders_shadow
    }
    return os.Getenv("DB_DSN")             // orders
}
```

#### 2. Redis 影子实例

```go
// Redis 客户端按请求标记路由到不同实例
func GetRedisClient(ctx context.Context) *redis.Client {
    if isShadowTraffic(ctx) {
        return redis.NewClient(&redis.Options{
            Addr: "redis-shadow:6379",  // 独立实例
            Password: "",
            DB: 0,
        })
    }
    return redis.NewClient(&redis.Options{
        Addr: "redis-prod:6379",
    })
}

// 判断是否是压测流量
func isShadowTraffic(ctx context.Context) bool {
    // 从 context 或 header 中读取压测标记
    if trace, ok := ctx.Value("trace").(Trace); ok {
        return trace.IsShadow
    }
    return false
}
```

#### 3. Kafka 影子 Topic

```yaml
# 压测时：
# 生产 Topic：order-events
# 影子 Topic：order-events-shadow

# 消费者订阅规则
kafka:
  topics:
    - name: order-events
      group: payment-consumer
    - name: order-events-shadow  # 压测流量也消费，但不影响生产
      group: payment-consumer-shadow

# 下游消费者对影子消息的处理
func (c *Consumer) Consume(msg *kafka.Message) error {
    if strings.HasSuffix(msg.Topic, "-shadow") {
        // 影子消息：只记录，不实际处理
        return nil  // 不 ack，消息会重复消费但不生效
    }
    // 生产消息：正常处理
    return c.processPayment(msg)
}
```

### 四、全链路压测执行 SOP

**Step 1：压测前准备**

```bash
# 1. 数据准备：复制生产数据子集到影子库
mysqldump orders orders_user orders_product \
  --where="created_at > '2024-01-01'" \
  | mysql orders_shadow

# 2. Redis 影子实例数据填充
redis-cli -h redis-shadow CONFIG SET maxmemory-policy allkeys-lru
redis-cli -h redis-shadow --pipe < /tmp/redis_keys_dump.txt

# 3. 下游服务降级开关打开
curl -X POST "http://config-center/api/switch/shadow-mode" \
  -d '{"enabled": true, "downstream_degrade": true}'

# 4. 监控告警静默（避免压测触发告警）
curl -X POST "http://alertmanager/api/v1/silences" \
  -d '{"matchers": [{"name":"env","value":"prod"}], "endsAt": "2099-01-01T00:00:00Z"}'

# 5. 全链路 Trace 标记开启（区分压测流量和正常流量）
# 在压测流量入口添加 header
-X 'X-Shadow-Traffic: true' \
-X 'X-Shadow-Version: v1'
```

**Step 2：逐步加压**

```bash
# 使用 k6 分布式压测，发起全链路请求
# 压测脚本中每个请求带上压测标记
export default function() {
    const res = http.get('http://api-gateway/api/v1/orders', {
        headers: {
            'X-Shadow-Traffic': 'true',  // 全链路标记
            'X-Request-Id': `shadow-${Date.now()}`,
        },
    });
}
```

**Step 3：监控全链路指标**

```bash
# 各服务关键指标
# Gateway: QPS、Latency、Upstream Errors
# Auth: Token 校验耗时、Cache 命中率
# Order: DB 写入 QPS、Redis 操作耗时
# Payment: 外部支付 API 超时率、DB 事务时长

# 重点关注：
# 1. TP99 拐点（容量上限信号）
# 2. Error Rate 跳升（系统开始拒绝请求）
# 3. 下游服务是否被压垮（Prometheus + Grafana 全链路看板）
```

**Step 4：瓶颈定位**

```bash
# 全链路 Trace 分析（用 Jaeger 或 SkyWalking）
# 找到耗时最长的节点
jaeger query --service=order-service \
  --lookback=10m --max-duration=5s

# DB 慢查询分析
SHOW FULL PROCESSLIST;
-- 找出执行时间 > 100ms 的查询

# Redis bigkey 检测
redis-cli -h redis-shadow --bigkeys
```

### 五、全链路压测风险与对策

| 风险 | 后果 | 对策 |
|------|------|------|
| 压测流量污染生产数据 | 数据混乱 | 严格影子库隔离 + 写操作加 `-shadow` 后缀 |
| 压垮下游依赖服务 | 线上故障 | 下游降级开关 + QPS 限流保护 |
| 压测触发大量告警 | 告警疲劳 | 压测前向 SRE 报备 + AlertManager 静默 |
| Kafka 消息被错误消费 | 业务逻辑被执行 | 影子 Topic 独立 ConsumerGroup |
| DNS/负载均衡被打爆 | 压测结果不准 | 用内网 VIP + 独立压测入口 |

### 六、生产经验/踩坑

**坑1：压测数据量级不对导致结果失真**
- 如果影子库只有 1/10 的数据，索引命中率不同，结果完全不准
- 解决：影子库数据量级要与生产接近（至少 30%）

**坑2：压测时缓存命中导致"假性能"**
- 冷缓存 vs 热缓存性能差距可达 10x
- 解决：压测前先跑 10 分钟预热，把缓存填满再正式测

**坑3：压测流量绕过了入口限流**
- 如果 Gateway 限流只针对真实流量，压测流量没进限流统计
- 解决：压测脚本要走真实入口，限流策略要对压测流量同样生效

**坑4：压测时 TSDB（Prometheus）被打爆**
- 全链路压测会产生大量 Trace 数据，TSDB 可能扛不住
- 解决：压测期间降低采样率（1% 而非 100%），压测完再恢复

---

## 高频追问

**Q1：没有条件做全链路压测，如何评估系统容量？**

三个替代方案：
1. **链路拆分压测**：分别压测各下游服务，用排队论估算整链容量
2. **流量放大法**：在非高峰期录制备用流量，白天放大 2-3x 回放
3. **监控推算**：看生产峰值 QPS 时各节点的利用率，推算瓶颈在哪

**Q2：如何验证全链路压测的数据隔离是完整的？**

- 压测结束后检查：影子库有没有数据进入生产库
- 监控：生产 DB 的写入 QPS 是否为 0（压测期间）
- 消息队列：检查生产 Topic 有没有影子消息被消费

**Q3：压测发现某服务是瓶颈，怎么决定扩容还是优化？**

- 看扩容成本：加 1 倍实例 vs 优化 1 处代码，哪个更划算
- 看优化上限：优化后能提升多少 QPS，是否满足需求
- 经验：优化代码能提升 30-50%，扩容可以快速线性提升

---

## 延伸阅读

- [TCPCopy 流量复制工具](https://github.com/session-replay-tools/tcpcopy)
- [GoReplay 官方文档](https://gor.dev/)
- [k6 官方文档 - 分布式压测](https://k6.io/docs/results-export/)
- [全链路压测在阿里双11的实践](https://developer.aliyun.com/article/742310)
