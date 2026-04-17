# 结构化日志与 ELK 方案

> 考察频率：★★★☆☆  难度：★★★☆☆
> 重点：能讲清楚日志规范、ELK 架构、采样策略

## 为什么需要结构化日志

### 普通日志 vs 结构化日志

**普通日志（难以检索）：**
```go
log.Printf("用户 %s 购买了 %d 件商品，总价 %f，订单号 %s", name, count, price, orderID)
// 输出：用户 张三 购买了 3 件商品，总价 99.50，订单号 ORD20260415
// 问题：无法按字段搜索，难以聚合分析
```

**结构化日志（JSON 格式）：**
```json
{
  "level": "INFO",
  "ts": "2026-04-15T12:08:00Z",
  "msg": "order placed",
  "user_id": "u12345",
  "order_id": "ORD20260415",
  "item_count": 3,
  "total_price": 99.50,
  "duration_ms": 45
}
```

---

## Go 结构化日志库

### zerolog（高性能）

```go
import "github.com/rs/zerolog/log"

// 基础使用
log.Info().
    Str("user_id", "u12345").
    Int("order_id", 100).
    Float64("amount", 99.50).
    Dur("latency", 45*time.Millisecond).
    Msg("order created")

// 嵌套结构
log.Info().Interface("order", map[string]interface{}{
    "id":    "ORD001",
    "items": []string{"item1", "item2"},
}).Msg("order details")

// 错误日志
log.Error().Err(err).Str("trace_id", traceID).Msg("payment failed")

// 采样：避免刷屏
sampled := log.Sample(&zerolog.BasicSampler{N: 100}) // 1% 采样
sampled.Info().Msg("debug info")
```

### zap（低延迟，推荐生产使用）

```go
import "go.uber.org/zap"

logger, _ := zap.NewProduction()
defer logger.Sync()

logger.Info("order placed",
    zap.String("order_id", "ORD001"),
    zap.Int("user_id", 12345),
    zap.Float64("amount", 99.50),
    zap.Duration("latency", 45*time.Millisecond),
)

// 糖语法（更简洁）
logger.Sugar().Infow("order placed",
    "order_id", "ORD001",
    "amount", 99.50,
)
```

---

## 日志规范

### 必须记录的字段

| 字段 | 说明 | 示例 |
|------|------|------|
| `trace_id` | 全链路追踪 ID | `4bf5a2c8-xxxx-xxxx` |
| `span_id` | 当前 span ID | `a1b2c3d4` |
| `user_id` | 用户 ID（脱敏） | `u_xxxx` |
| `action` | 操作类型 | `order.create` |
| `duration_ms` | 执行耗时 | `45` |
| `error` | 错误信息 | `timeout` |

### 日志级别规范

| 级别 | 场景 | 示例 |
|------|------|------|
| **DEBUG** | 开发调试 | SQL 语句、缓存命中/未命中 |
| **INFO** | 正常业务流程 | 订单创建、支付成功 |
| **WARN** | 潜在问题 | 重试、超时、限流触发 |
| **ERROR** | 需要关注 | 数据库错误、接口调用失败 |
| **FATAL** | 进程退出 | 配置文件缺失、端口绑定失败 |

### 禁止的行为

```go
// ❌ 禁止：日志打印对象未序列化
log.Info().Msgf("user: %v", userObj)  // 输出地址而非内容

// ❌ 禁止：敏感信息
log.Info().Msgf("password: %s", password)

// ❌ 禁止：日志中打印大对象
log.Info().Msgf("request body: %s", hugeJSONString)

// ❌ 禁止：日志覆盖业务逻辑
if log.Debug().Enabled() {
    log.Debug().Msg("expensive operation")  // 浪费判空开销
}
```

---

## ELK 架构

### 整体架构

```
应用服务（Go 服务）
    │
    │  JSON 日志文件写入
    ▼
Filebeat（日志收集代理）
    │  解析 JSON，轻量处理
    ▼
Logstash（聚合、过滤、转换）
    │  Grok 解析、富化（加 IP 地理位置）
    ▼
Elasticsearch（存储 + 搜索）
    │  索引化管理
    ▼
Kibana（可视化查询）
```

### Filebeat 配置

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/myapp/*.log
    json.keys_under_root: true      # 把 JSON 字段提到根级别
    json.add_error_key: true
    json.message_key: msg

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~           # 自动加云服务商信息
  - add_docker_metadata: ~          # 自动加容器信息

output.logstash:
  hosts: ["logstash:5044"]
```

### Logstash 管道配置

```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  # JSON 解析
  json {
    source => "message"
    target => "parsed"
  }

  # 时间处理
  date {
    match => ["ts", "ISO8601"]
    target => "@timestamp"
  }

  # Grok 解析非结构化日志
  grok {
    match => { "message" => "%{WORD:level} %{TIMESTAMP_ISO8601:ts} %{DATA:logger} - %{GREEDYDATA:msg}" }
  }

  # 富化：添加 IP 地理位置
  geoip {
    source => "client_ip"
    target => "geo"
  }

  # 过滤：丢弃 DEBUG 日志
  if [level] == "DEBUG" {
    drop { }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "myapp-%{+YYYY.MM.dd}"
  }
}
```

---

## 日志采样策略

### 为什么要采样

高并发系统中，每秒可能产生数万条日志：
- 存储成本高
- 写入影响性能
- 查询效率低

### 采样方法

**1. 固定比例采样（简单，但丢失尾部问题）**
```go
func SampleLog(msg string, rate float64) {
    if rand.Float64() < rate {
        log.Info().Msg(msg)
    }
}
// 问题：ERROR 日志也可能被丢弃！
```

**2. 尾部采样（优先保留错误日志）**
```go
type SamplingLogger struct {
    errorLogger *zap.SugaredLogger
    sampleRate  float64
}

func (l *SamplingLogger) Info(msg string, fields ...interface{}) {
    if rand.Float64() < l.sampleRate {
        l.errorLogger.Info(msg, fields...)
    }
}

func (l *SamplingLogger) Error(msg string, fields ...interface{}) {
    // ERROR 日志 100% 保留
    l.errorLogger.Error(msg, fields...)
}
```

**3. 基于 TraceID 的采样（保证同一请求日志完整性）**
```go
func shouldSample(traceID string, rate float64) bool {
    // hash(traceID) 前缀决定是否采样
    // 同一 traceID 永远被同样处理
    h := crc32.ChecksumIEEE([]byte(traceID))
    return float64(h&0xFFFF)/65536.0 < rate
}
```

**4. 自适应采样（高峰期自动降采样）**
```go
func adaptiveSample(level string, traceID string) bool {
    qps := currentQPS()
    
    switch {
    case level == "ERROR":          // ERROR 永远保留
        return true
    case level == "WARN" && qps < 1000:
        return true
    case level == "INFO" && qps < 100:
        return true
    case qps > 5000:
        return rand.Float64() < 0.01  // 高峰期 1% 采样
    default:
        return rand.Float64() < 0.1
    }
}
```

---

## 日志与链路追踪结合

### 通过 TraceID 串联日志

```go
// 中间件自动注入 TraceID
func TracingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        traceID := r.Header.Get("X-Trace-ID")
        if traceID == "" {
            traceID = uuid.New().String()
        }
        
        // 注入到 context，所有日志自动带 traceID
        ctx := context.WithValue(r.Context(), "trace_id", traceID)
        w.Header().Set("X-Trace-ID", traceID)
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// 日志自动带 traceID
func logFromContext(ctx context.Context) *zerolog.Logger {
    traceID := ctx.Value("trace_id").(string)
    return log.With().Str("trace_id", traceID).Logger()
}
```

---

## 常见问题

### Q：日志写入文件还是直接写 Kafka/ES？

> **推荐：写文件 + Filebeat 收集**。原因：不影响主流程（异步刷盘）；Filebeat 批量发送，提高吞吐；服务重启不丢日志；解耦，日志系统可独立扩展。

### Q：如何控制日志量不影响性能？

> 使用 zerolog/zap 的异步模式（`zap.NewAsync`）；批量写入（每 100ms 或 100 条刷新一次）；采样 + 采样自适应策略。

### Q：Kibana 查询慢怎么优化？

> 按业务维度拆分索引（`app-orders`, `app-payments`）；使用 Rollover 定期创建新索引，避免单索引过大；开启 Elasticsearch Index Sorting。

### Q：日志中如何处理敏感信息？

> 脱敏四件套：手机号、身份证、银行卡、密码。示例：`138****1234`，`310***********1234`。中间件统一处理，不在业务代码中硬编码。
