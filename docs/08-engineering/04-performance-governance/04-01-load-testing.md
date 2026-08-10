# 压测方法论：从单接口到全链路

> 考察频率：★★★★★  优先级：P0
> 关键词：QPS、TP99、基准压测、负载压测、峰值压测、稳定压测

---

## 面试官考察意图

这道题考的是**你有没有亲自压测过生产系统**，而不是背过概念。面试官真正想知道的是：

- 你能否设计一个**科学的压测方案**（不是乱打一通）
- 你能否**读懂压测报告**，定位真实瓶颈
- 你是否踩过压测的坑（比如压测把缓存打穿、把DB打挂）

**高级工程师的回答** vs **初级工程师的回答**：

| 维度 | 初级 | 高级 |
|------|------|------|
| 压测目标 | "压一下看能抗多少QPS" | 先定目标：容量验证/瓶颈识别/回归对比 |
| 压测分类 | 只做过单接口 | 清楚基准/负载/压力/稳定性压测的区别 |
| 数据准备 | 随便造几条数据 | 知道要预热、缓存、数据量级要接近生产 |
| 结果分析 | 看错误率就行 | 看TP99/GC/DB连接池/Redis耗时综合分析 |
| 报告输出 | 截图丢群里 | 给出SLA边界、扩容建议、风险点 |

---

## 核心答案（30秒版）

压测分四类：**基准压测**（单接口最优性能）→ **负载压测**（摸到容量上限）→ **压力压测**（超过容量观察降级）→ **稳定性压测**（长时间验证可靠性）。关键指标是 **QPS + TP99 + 错误率**，以及 **CPU/GC/DB连接池/Redis耗时** 四条曲线的健康度。压测前必须预热 + 缓存预填充 + 监控告警关闭策略。

---

## 深度展开

### 一、压测分类与目标

```
压测类型        目标                          持续时间
─────────────────────────────────────────────────────────
基准压测        单接口最优性能基线              30s~2min
负载压测        逐步加压找容量上限              10~30min
压力压测        超过容量验证降级/熔断          5~10min
稳定性压测      长时间验证（内存泄漏/连接泄漏）  4h+
尖峰压测        模拟流量突增（秒杀/活动）        按业务场景
```

### 二、压测前置准备（很多人死在这一步）

**1. 测试数据要真实**

```sql
-- 压测用户数据要与生产量级接近
-- 避免缓存命中异常（Redis缓存预填充）
-- 避免冷数据问题（索引失效场景）

-- 生产数据影子库方案
CREATE TABLE orders_shadow LIKE orders;
-- 压测期间写入影子表，不污染生产数据
INSERT INTO orders_shadow SELECT * FROM orders WHERE created_at > '2024-01-01';
```

**2. 压测环境尽量隔离**

```yaml
# k8s 压测环境配置（独立namespace，不影响生产）
apiVersion: v1
kind: Namespace
metadata:
  name: perf-test
---
# 副本数、资源限制与生产一致
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

**3. 依赖服务隔离**

- 压测 DB 要独立实例（不要和生產 DB 共享）
- 压测 Redis 同样独立（或用 shadow key 策略）
- 第三方 API 用 Mock Server（避免压垮下游）

**4. 压测前检查清单**

```bash
# 连接池是否足够
curl -s localhost:6060/debug/pprof/heap | grep -i conn

# GC 是否正常（压测前跑一次 force GC）
curl -s "localhost:6060/debug/gc" && sleep 5

# 日志级别是否已调低（压测期间用 warn，避免 io 打满）
# 日志开关确认
grep -r "LogLevel" config/

# 监控告警临时静默（避免告警风暴）
```

### 三、压测工具选型

| 工具 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **wrk/wrk2** | 单接口基准压测 | 简单、输出详细 | 不支持复杂场景 |
| **vegeta** | 恒定 QPS 压测 | 结果稳定、可做 CI | 集群压测能力弱 |
| **k6** | 综合压测（脚本化） | 支持场景编排、结果格式好 | 学习曲线 |
| **locust** | 分布式压测 | Python 脚本、团队熟悉 | Python GIL 限制 |
| **hey** | 简单快速验证 | 轻量 | 不适合大数据量 |

**推荐组合**：本地单接口用 `wrk2`（恒定 QPS，结果稳定），全链路用 `k6`（场景编排能力强）。

### 四、压测执行 SOP

**Step 1：先做基准压测（建立基线）**

```bash
# 单接口基准：持续30s，目标QPS=1000
wrk -t4 -c100 -d30s -R1000 http://target/api/v1/orders

# 关键输出：
# Requests/sec: 999.34       ← 基准 QPS
# Latency Distribution:
#      50%   12.34ms
#      90%   18.56ms
#      99%   35.21ms        ← TP99 基线
#      99.9% 89.45ms
```

**Step 2：逐步加压（负载压测）**

```bash
# 5分钟预热后，每30s加压200 QPS，观察TP99拐点
wrk -t8 -c200 -d30s -R200 http://target/api/v1/orders
wrk -t8 -c400 -d30s -R400 http://target/api/v1/orders
wrk -t8 -c600 -d30s -R600 http://target/api/v1/orders

# 拐点判断：
# TP99 突然从 35ms 跳到 200ms → 容量上限附近
# 错误率从 0% 跳到 5% → 已超载
```

**Step 3：找到瓶颈后定向排查**

```bash
# CPU 瓶颈
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# GC 瓶颈（频繁 STW）
curl -s "localhost:6060/debug/gc" && echo "" > /tmp/gc.log
# 观察 GC log 中 "GC pause" 频率

# DB 连接池瓶颈
# 检查 MaxOpenConns / MaxIdleConns 配置
# 监控 WaitCount 是否有增长

# goroutine 堆积（高延迟表现）
curl -s localhost:6060/debug/pprof/goroutine?debug=1 | head -50
```

### 五、压测报告模板

```markdown
## 压测报告：/api/v1/orders

### 基线数据
- 基准 QPS（单实例）：1200 req/s
- TP50 / TP90 / TP99：8ms / 12ms / 35ms
- 错误率：0%

### 容量拐点
- 800 QPS：TP99 稳定在 40ms ✅
- 1000 QPS：TP99 涨到 80ms，CPU 75% ⚠️
- 1200 QPS：TP99 涨到 200ms，错误率 3% ❌

### 瓶颈定位
- 主要瓶颈：DB 连接池（MaxOpenConns=20 打满）
- 次要瓶颈：GC 频繁（对象分配速率 800MB/s）

### 扩容建议
- 当前：4实例 × 2核 = 8核
- 目标 2倍容量：扩容至 8实例，或连接池调至 50
- SLA 承诺：1000 QPS 下 TP99 < 100ms

### 风险点
- 缓存穿透时 DB 会直接被打爆（需加熔断）
```

### 六、生产经验/踩坑

**坑1：压测机和被压服务网络不同区**
- 跨 Region 压测结果不准，要用同 VPC 或同机房机器
- 解决：用内网 VIP 压测，避免公网抖动

**坑2：压测数据不均匀打爆缓存**
- 如果压测 key 集中在少数缓存 key，缓存被打穿后结果完全不准
- 解决：用足够分散的测试数据集（覆盖足够多的缓存 key）

**坑3：压测时日志 IO 成为瓶颈**
- 压测 QPS 很高时，日志写入可能占 20%+ IO
- 解决：压测期间日志级别调至 warn，或直接关闭文件日志

**坑4：压测只打下游，没考虑上游入口限流**
- 如果 API Gateway 有限流，压测打不满也可能是入口限流
- 解决：确认压测路径不受 Gateway 限流影响

---

## 高频追问

**Q1：压测发现 TP99 突然升高怎么排查？**

按顺序查：
1. `pprof` 看 CPU 占用，是否 GC 在作妖（看 gc trace）
2. `pprof/goroutine` 看 goroutine 是否堆积（说明某处阻塞了）
3. 查 DB Slow Query Log，看是否有慢查询
4. 查 Redis 耗时（bigkey / hotkey 导致单命令慢）
5. 查连接池 `WaitCount` 是否在涨（连接不够用）

**Q2：如何用压测验证熔断器是否生效？**
- 逐步加压到容量上限，观察：
  - 错误率是否有明显跳升（熔断器打开）
  - 熔断后下游服务是否被保护（QPS 降但服务不崩溃）
  - 熔断恢复是否正常（半开→关闭）

**Q3：压测机和被压服务 CPU 不一样怎么办？**
- 按核心数比例换算（压测机 8核，被压机 4核，结果要除以2）
- 或者直接用 `GOMAXPROCS` 调整被压服务线程数来对齐

**Q4：没有压测环境，怎么评估系统容量？**
- 用信封估算法：单接口耗时 × 预期 QPS × 安全系数
- 参考线上监控：单实例峰值 QPS × 实例数 × 70%（留 30%buffer）

---

## 延伸阅读

- [wrk2 使用指南](https://github.com/wg/wrk/wiki/Installing-Wrk-on-Linux)
- [k6 官方文档](https://k6.io/docs/)
- [Go pprof 官方文档](https://pkg.go.dev/runtime/pprof)
- [Go GC Log 分析](https://go.dev/doc/gc-trace)
