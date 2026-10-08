# 可观测性成本治理：Prometheus 指标基数爆炸（Cardinality Explosion）

> 考察频率：★★★★★  优先级：P1
> 关键词：Cardinality、高基数、Time Series、label 纪律、relabel、native histogram、ClickHouse 分流

## 面试官考察意图

"你们监控量大吗？有没有遇到过 Prometheus 内存爆掉 / 查询变慢 / 账单暴涨？"——这是 5~8 年工程师**最能拉开差距**的可观测性题。它考察候选人是否理解：

1. **指标（Metric）的本质是聚合趋势，不是逐请求流水账**；
2. **基数 = 指标名 + 所有 label 组合出的时间序列数**，且 label 之间是**乘法关系**；
3. 出现基数爆炸时，**排查路径**（topk 查 series、`/api/v1/status/tsdb`）和**治理手段**（label 纪律、relabel 丢弃、path 归一化、高基数分流到 OLAP）。

**核心立场：Prometheus 的性能问题，90% 都可以归结为一句话——label 基数。** 谁把 `user_id` / `request_id` / 原始 URL 塞进 label，谁就是在给监控系统埋雷。

---

## 核心答案（30 秒版）

**基数（Cardinality）**：一个指标下所有 label 取值**组合**形成的时间序列数量。基数 = ∏(每个 label 的取值数)。

**为什么爆炸**：label 是**乘法**的。`http_request_duration_seconds_bucket` 实例(100) × le 分桶(10) × url(400) × method(5) = **200 万条 series** 💀。若 url 是动态 path，取值近乎无穷，基数无法计算。

**危害**：Prometheus 内存/存储暴涨、查询慢、rule 评估超时（`PrometheusRuleFailures`）、自建是容量问题、SaaS 是账单问题。

**治理五招**：
1. **label 纪律**：只放**有限枚举值**（环境、服务名、状态码、接口类别、集群），禁止 `user_id`/`order_id`/`request_id`/session_id/原始 URL 参数；
2. **path 归一化**：`/user/123/order/456` → `/user/:id/order/:id`（用路由模板而非原始 URL）；
3. **采集端 relabel 兜底**：`metric_relabel_configs` 丢弃危险 label；
4. **持续监控基数**：`topk(10, count by (__name__, job) ({__name__=~".+"}))` 定期巡检；
5. **高基数分析分流到 OLAP**：需要按 user_id 分析就进 ClickHouse，而不是塞进 Prometheus。

**一句话**：**指标负责趋势与聚合，不负责保存每个请求的人生轨迹；要按高基数维度分析，请去日志 / Trace / OLAP。**

---

## 深度展开

### 一、基数到底怎么算（面试必答的乘法陷阱）

给一条典型的 HTTP 指标打标签：

```text
http_request_duration_seconds_bucket
  instance  : 100 个实例
  le        : 10 个分桶
  url       : 400 个接口
  method    : 5 个方法

series = 100 × 10 × 400 × 5 = 2,000,000 条
```

如果再把 `user_id` 加进去（假设 100 万用户）：

```text
100 × 10 × 400 × 5 × 1,000,000 = 天文数字
```

**关键认知**：多加一个 label **不是加法，是乘法**。这也是为什么"我就加一个 user_id 而已"会造成事故。

业内经验阈值：
- 低基数：约 `1:5` 的 label 值比率；
- 标准基数：约 `1:80`；
- **高基数：约 `1:10000`**。

### 二、哪些字段绝对不能做 label

判断标准一句话：**这个 label 的值会不会无限增长？**

| ❌ 禁止做 label | ✅ 适合做 label |
|---|---|
| `user_id`、`order_id`、`session_id` | `env`（prod/dev） |
| `request_id`、`trace_id` | `service`、`cluster` |
| 原始 URL / path 参数 | `http_method` |
| 经纬度、时间戳 | `status_code`（或归类的 2xx/4xx/5xx） |
| 动态错误消息全文 | `endpoint`（路由模板归一化后） |

> 记忆口诀：**"一次请求一个值 / 一个用户一个值 / 一个订单一个值"→ 不能做 label。**

### 三、Go 侧怎么写才不炸

```go
// ❌ 灾难：user_id + 原始 path 全塞进去
httpDuration.WithLabelValues(
    r.URL.Path,                                              // /user/123/order/456
    r.Header.Get("X-User-ID"),                               // 百万级
    strconv.Itoa(resp.StatusCode),                           // 精确 200/404... 还行
).Observe(seconds)

// ✅ 正确：用路由模板 + 归类状态码，关掉 user_id
pattern := routePattern(r)     // 由路由框架给出 "/user/:id/order/:id"
statusClass := resp.StatusCode / 100 * 100 // 200 / 400 / 500
httpDuration.WithLabelValues(pattern, strconv.Itoa(statusClass)).Observe(seconds)
```

要点：
- **用路由模板而不是原始 URL**——这是最高频的爆炸源，尤其在 RESTful 接口里；
- 状态码只留**分类**（2xx/3xx/4xx/5xx），精确码基数也可控但常无必要；
- 业务标识（user_id/order_id）通过 **Trace / 日志** 关联，不进 metric label；
- 用 `promauto` / `prometheus.NewCounterVec` 注册时，预检查 label 维度的**取值上界**。

### 四、排查：怎么找到"谁把基数搞爆了"

```bash
# 1) 查全局基数 top：哪些指标 series 数最多
curl -s 'http://prom:9090/api/v1/status/tsdb' | jq '.data.seriesCountByMetricName[0:10]'

# 2) PromQL：按指标 + job 统计 series 数，找头部
topk(10, count by (__name__, job) ({__name__=~".+"}))

# 3) 某个可疑指标的 label 值数量排行
count by (url) (http_request_duration_seconds_count)

# 4) series 增长趋势（看是不是某个时间点突变）
prometheus_tsdb_head_series
```

**典型根因**：上线了一个带 `user_id` label 的新埋点 / 有人把原始 path 直接打标 / 上游 relabel 规则被误删。

### 五、治理：采集端 relabel 兜底 + 分流

```yaml
# prometheus.yml
scrape_configs:
  - job_name: my-service
    metric_relabel_configs:
      # 丢弃危险的动态 label
      - action: labeldrop
        regex: "user_id|order_id|request_id|trace_id"
      # 把动态 path 归一化（示例：把数字段替换为 :id）
      - source_labels: [url]
        target_label: url
        regex: "/user/([0-9]+)/.*"
        replacement: "/user/:id/..."
```

```text
# 分流原则
低基数聚合趋势  -> Prometheus / VictoriaMetrics
高基数逐条分析  -> ClickHouse / Loki / 自建 OLAP
Trace 全量明细  -> Jaeger / Tempo（按采样率）
```

### 六、进阶：Native Histogram 与 cardinality 的关系

传统 histogram 的每个 bucket 都是**独立 series**（`_bucket{le="..."}`），分桶越多基数越大——这正是上例里 ×10 的来源。**Native Histogram（Prometheus 2.40+ 实验特性 / 2.50 稳定）** 把直方图编码进**单个 series**，从根本上减少 bucket 带来的基数膨胀。

```text
经典 histogram：1 个指标 × 10 个 le = 10 条 series
native histogram：1 条 series（bucket 编码在内部）
```

面试加分点：**如果项目用的是高精度分桶（比如 20+ 个 le），迁到 native histogram 能显著降基数。** 但要注意客户端和 PromQL 兼容性。

---

## 面试话术

> "我们对指标 label 有明确的纪律：**只放有限枚举维度**，像 user_id、request_id、原始 URL 这类高基数或无限增长的字段一律不进 label，需要关联就通过日志和 Trace 的 trace_id 去串。落地时做了三件事：一是路径用路由模板归一化，避免 `/user/123` 这种；二是在 Prometheus 采集端用 `metric_relabel_configs` 兜底 `labeldrop` 危险标签；三是定期用 `topk(count by (__name__))` 巡检基数 TopN，配合 `/api/v1/status/tsdb` 定位异常。需要按高基数维度做分析时，我们会把数据分流到 ClickHouse，而不是塞进 Prometheus。"

---

## 高频追问

- **Q：怎么判断一个 label 是不是高基数？** A：看取值是否有限枚举；是否"一次请求/一个用户/一个订单一个值"；经验上超过 1:10000 比率即为高基数。
- **Q：已经爆炸了怎么救？** A：先 relabel `labeldrop` 止血 → 改埋点去掉高基数 label → 用 `promtool tsdb` 的 delete 或直接删数据重建；长期靠 label 纪律 + 巡检。
- **Q：为什么不用 Prometheus 存高基数数据？** A：Prometheus 为每个 series 建索引、单机内存模型，高基数直接吃内存和查询耗时；高基数分析更适合列存 OLAP（ClickHouse）。
- **Q：histogram 分了 20 个桶会不会基数爆炸？** A：会，每个 le 是独立 series；考虑 native histogram 或减少分桶精度。
