# 容灾设计：单机房、双活与异地多活

> 考察频率：★★★★★  优先级：P0
> 关键词：RPO、RTO、单机房、双活、异地多活、数据一致性、流量切换

[🏠 首页](../../../README.md) · [📚 分布式模块](../README.md)

---

## 面试官考察意图

这道题考察候选人对**分布式系统可用性设计**的整体认知。
- 初级工程师只能说出"要多机房部署"，讲不清 RPO/RTO 指标
- 高级工程师要能说清楚**不同容灾等级的技术方案差异、成本对比、选型原则**，以及**数据一致性保障手段**和**流量切换的线上操作风险**
- 加分项：能结合自己经历讲出"做过哪个级别的容灾，踩过什么坑"

---

## 核心答案（30 秒版）

容灾设计的核心是两个指标：**RPO（恢复点目标）**决定数据丢多少，**RTO（恢复时间目标）**决定服务停多久。

| 容灾等级 | 方案 | RPO | RTO | 代价 |
|---------|------|-----|-----|------|
| 冷备 | 主备单机房 | 分钟级 | 小时级 | 低 |
| 热备 | 主备多机房 | 秒级 | 分钟级 | 中 |
| 双活 | 同城双活 | 秒级 | 秒级 | 高 |
| 多活 | 异地多活 | 秒级 | 秒级 | 极高 |

**选型原则：** 业务中断损失 < 建设投入 → 金融/交易系统选双活，普通业务热备足够。

---

## 深度展开

### 1. RPO / RTO 量化标准

```
RPO（Recovery Point Objective）：
- 金融支付：RPO = 0（不允许丢数据）→ 必须同步双写
- 社交 Feed：RPO = 1 小时可接受 → 可用异步复制
- 日志系统：RPO = 5 分钟可接受

RTO（Recovery Time Objective）：
- 核心交易：RTO < 5 分钟 → 必须自动切换
- 后台系统：RTO < 30 分钟 → 可人工介入
```

**生产案例：** 某电商大促期间主机房网络闪断，运维手动切换 DNS 花了 18 分钟，损失订单峰值 2000 单/分钟。复盘后接入 VIPkid 式的健康检查 + 自动 DNS 切换，RTO 压到 45 秒。

---

### 2. 单机房故障场景与应对

#### 2.1 机房断电 / 光缆中断

**问题：** 单机房所有实例同时不可用

**解法：**
```
① 至少 2 个可用区（AZ）部署，AZ 间网络延迟 < 2ms
② MySQL 半同步复制（semi-sync）：主库确认至少一个从库落盘才返回
③ 弱依赖降级：非核心链路（如推荐服务）可切本地缓存兜底
④ 监控：机房网络中断时自动触发告警，不等人工发现
```

#### 2.2 灰度发布引发可用性事故

**问题：** 新版本有 Bug，发布到 10% 流量后引发大量超时

**解法（金丝雀 + 快速回滚）：**
```go
// Go 实现的灰度流量管理器（简化版）
type GrayRouter struct {
    enabled  atomic.Bool
    percent  atomic.Int32
    subsets  map[string]struct{} // 新版本实例
}

func (g *GrayRouter) Route(key string) bool {
    if !g.enabled.Load() {
        return false // 全量走老版本
    }
    // 按用户 ID hash，确保同用户每次路由一致
    h := fnv32(key) % 100
    return h < g.percent.Load()
}

// 快速回滚：收到告警后，enabled 置 false，RTO < 10 秒
func (g *GrayRouter) Rollback() {
    g.enabled.Store(false)
}
```

---

### 3. 同城双活：同城市两个机房

#### 3.1 架构设计

```
          ┌──────────────┐
          │   负载均衡    │
          │  （VIP/DNS） │
          └──────┬───────┘
           ┌─────┴─────┐
      ┌────▼───┐   ┌──▼────┐
      │ 机房 A  │   │ 机房 B  │
      │  active │   │ standby│
      └────┬───┘   └──┬────┘
           └─────┬─────┘
            MySQL 半同步
```

#### 3.2 流量分配策略

| 策略 | 适用场景 | 实现方式 |
|------|---------|---------|
| 主动-被动 | 平时只用主机房，从机房不承流 | VIP 漂移 + DNS 切换 |
| 主动-主动 | 俩机房同时承流，需解决数据同步 | 双向同步 + 冲突处理 |
| 读分发写集中 | 写只在一机房，读跨机房 | 跨机房读 + 同城同步写 |

**推荐：主动-被动模式**，简单可靠，故障切换时流量漂移即可。

#### 3.3 数据同步方案

**MySQL 半同步复制：**
```sql
-- 安装半同步插件
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

-- 启用半同步
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

**生产问题：** 半同步从库复制延迟大时，主库会退化为异步复制（sync_relay_log 已关闭），数据有丢失风险。

**解法：** 监控 `rpl_semi_sync_master_no_times`（退化为异步次数），超过阈值告警，强制切走流量。

---

### 4. 异地多活：三个城市五个机房

#### 4.1 架构挑战

```
            ┌──────────┐
            │ 全局路由层│  ← 智能 DNS / Anycast
            └────┬─────┘
         ┌───────┼───────┐
    ┌────▼─┐ ┌──▼──┐ ┌──▼──┐
    │ 上海 │ │ 北京 │ │ 广州 │
    │Zone A│ │Zone B│ │Zone C│
    └──┬──┘ └──┬──┘ └──┬──┘
       └────────┴────────┘
         跨地域数据同步（异步）
```

#### 4.2 数据一致性核心问题：数据如何分区

**原则：就近写入 → 就近读取**

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| 用户就近 | 按用户 ID 路由到固定机房 | 一致性强 | 负载不均时难处理 |
| 写统一读分散 | 写只在主站，读可跨机房 | 实现简单 | 主站故障时所有写失败 |
| CRDT 冲突解决 | 冲突时自动合并（如 Redis Sets） | 不需要同步 | 只适用特定数据类型 |

**生产案例：** 某社交 Feed 系统使用"写统一读分散"，用户写微博只写主站（上海），读取时按用户地理位置路由到最近机房读从库。主站故障时，临时降级为"允许所有机房写"，通过消息队列异步汇聚到主站。

#### 4.3 跨机房网络延迟对系统的影响

| 路径 | 延迟 | 影响 |
|------|------|------|
| 同机房 | 0.1~0.5ms | 无感 |
| 同城双活 | 1~3ms | 可接受，RPC 超时 +2ms |
| 异地多活 | 20~50ms | **显著**，需要异步化或读写分离 |

**Go 生产优化：**
```go
// 跨机房调用时，增加超时容错
const同城RPCTimeout = 50 * time.Millisecond
const跨机房RPCTimeout = 200 * time.Millisecond

// 对外暴露的统一调用接口
func CallRemote(ctx context.Context, target string, req *Request) (*Response, error) {
    latency := estimateLatency(target) // 路由前估算延迟
    var cancel context.CancelFunc
    ctx, cancel = context.WithTimeout(ctx, latency+100*time.Millisecond)
    defer cancel()
    return grpcClient.Invoke(ctx, target, req)
}
```

---

### 5. 流量切换 SOP（生产操作手册）

#### 5.1 故障切换流程

```
检测故障（p99 告警 / 健康检查失败）
    ↓
确认是否需要切换（误报过滤，避免抖动）
    ↓
执行 DNS 切换（修改 VIP 路由或修改 DNS 解析）
    ↓
观察新机房流量是否正常（监控 QPS / 错误率）
    ↓
原机房下线实例（防止数据不一致）
    ↓
通知相关方（PA/运营/法务，根据业务影响决定）
```

#### 5.2 DNS 切换 vs VIP 漂移

| 方式 | 切换速度 | 实现复杂度 | 适用场景 |
|------|---------|-----------|---------|
| DNS 修改 | 较慢（TTL 生效需等）| 低 | 非核心业务，可接受分钟级 |
| VIP 漂移 | 快（秒级）| 高（需 LB 支持）| 核心交易，要求秒级 |
| Anycast | 最快（路由收敛）| 极高 | 腾讯/阿里云场景 |

**DNS 切换注意事项：**
- 故障前把 TTL 改小（如从 3600s 改到 60s），避免切换后 DNS 缓存导致流量回不来
- 切换完成后记得改回去，否则正常时每次解析都要查 DNS，性能浪费

#### 5.3 切换后数据一致性检查

```go
// 切换完成后，比对两机房数据一致性（简化版）
func CheckDataConsistency(localDB *sql.DB, remoteDB *sql.DB) error {
    tables := []string{"orders", "accounts", "inventory"}
    
    for _, table := range tables {
        localCount, err := getRowCount(localDB, table)
        if err != nil {
            return fmt.Errorf("local %s count failed: %w", table, err)
        }
        
        remoteCount, err := getRowCount(remoteDB, table)
        if err != nil {
            return fmt.Errorf("remote %s count failed: %w", table, err)
        }
        
        if localCount != remoteCount {
            return fmt.Errorf("data inconsistency in %s: local=%d remote=%d", 
                table, localCount, remoteCount)
        }
    }
    return nil
}
```

---

### 6. 容灾演练

**重要性：** 容灾方案不做演练 = 没有方案。生产案例中大量故障是演练不足导致的。

| 演练类型 | 频率 | 内容 |
|---------|------|------|
| 桌面推演 | 每季度 | 假设 XXX 故障，走一遍操作手册 |
| 模拟切换 | 每半年 | 断掉某机房，验证 RTO 是否达标 |
| 真实演练 | 每年 | 在低峰期真实切换，验证数据完整性 |

**演练最小间隔：** 重大系统变更后（如 MySQL 版本升级、K8s 迁移）必须加一次演练。

---

## 高频追问

### Q1：双活机房数据出现冲突怎么解决？

**答案：** 优先在架构上避免冲突（按用户 ID 分片，写到固定机房）。如果无法避免：
- 使用时间戳 + 用户 ID 作为冲突解决依据（后写胜出）
- 关键数据用分布式事务（2PC/TCC）保证强一致
- 展示/评论等对一致性要求低的场景用消息队列异步合并

### Q2：故障切换时如何防止缓存雪崩？

**答案：**
1. 切换前后**不清空缓存**，只禁用写功能，读继续
2. 如果必须重建缓存，**分批预热**，避免大量 Key 同时过期
3. 缓存与数据库同时切换，确保旧缓存不会读到旧数据库

### Q3：如何保证 DNS 切换后流量不回源到原机房？

**答案：**
1. 故障前降低 DNS TTL（如 3600s → 60s）
2. 切换后，原机房实例同时下线（关闭端口或从 LB 摘除）
3. 如果用了 VIP 漂移，通过路由表直接隔离原机房网络

### Q4：异地多活对业务代码有什么侵入性要求？

**答案：**
- **不能有本地状态**（如内存缓存、单机锁）
- **Session 必须外置**（Redis/DB，不要存内存）
- **写请求路由到固定机房**（中间件层实现，对业务代码透明）
- **唯一 ID 生成** 不能用单机自增，要用 Snowflake 等分布式 ID

---

## 延伸阅读

- [阿里云容灾白皮书](https://help.aliyun.com/document_detail/188445.html)
- [美团双活架构设计](https://tech.meituan.com/2018/03/29/disaster-recovery.html)
- [AWS 多区域架构](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
- 《Designing Data-Intensive Applications》第 8 章：分布式系统挑战
