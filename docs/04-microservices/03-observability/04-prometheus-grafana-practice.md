# Prometheus + Grafana 监控实战

> 考察频率：★★★★☆  优先级：P1

## TODO（待填写）

## 1. Prometheus 数据模型
- [ ] 时间序列：`metric_name{label1="v1"} value timestamp`
- [ ] 四种指标类型：Counter / Gauge / Histogram / Summary
- [ ] Histogram vs Summary：桶（客户端 vs 服务端计算分位数）
- [ ] 标签设计原则：高基数标签（如 user_id）为什么不能用

## 2. PromQL 高频查询
- [ ] `rate()` vs `irate()`：平均速率 vs 瞬时速率
- [ ] `histogram_quantile(0.99, rate(...))`：P99 延迟计算
- [ ] `sum by (instance)`：按维度聚合
- [ ] Recording Rules：预计算慢查询（减少查询压力）

## 3. Go 服务接入全流程
- [ ] `prometheus/client_golang` 注册自定义指标
- [ ] `/metrics` 端点暴露
- [ ] HTTP 请求耗时 Histogram 完整代码示例
- [ ] 业务指标：订单数 / 支付成功率 / 库存剩余

## 4. Grafana 看板设计
- [ ] RED 方法：Rate（QPS）/ Error（错误率）/ Duration（P99 延迟）
- [ ] USE 方法：Utilization / Saturation / Errors
- [ ] 服务级 SLO 看板：可用性 / 错误率 / 延迟分布
- [ ] Variables：动态切换环境/实例

## 5. AlertManager 告警配置
- [ ] 告警规则：`PrometheusRule` CRD（K8s 环境）
- [ ] 抑制（Inhibit）：防止告警风暴
- [ ] 路由（Route）：按 severity 分发到不同渠道（Slack/钉钉）
- [ ] 告警分级：P0（立即）/ P1（30 分钟）/ P2（工作时间）

## 6. 生产踩坑
- [ ] Prometheus 内存暴涨：高基数 label 导致时间序列爆炸
- [ ] 长期存储：Thanos / VictoriaMetrics
- [ ] 跨机房联邦（Federation）

## 高频追问
- [ ] Histogram 的 bucket 应该怎么设置？
- [ ] P99 和 P999 的区别，分别适合什么场景？
- [ ] 为什么高基数 label 会导致 Prometheus OOM？
