# 分布式配置中心（Config Center）

> 考察频率：★★★★☆  难度：★★★☆☆  关键词：推 vs 拉、长轮询、灰度发布、本地容灾快照

## 一、为什么不能只是"远程配置文件"

把配置丢到 Git/OSS 上只是静态存储，真正的配置中心要解决四件事：

1. **动态生效**：改配置不重启、不发版（否则等于每次都要走发布流程）。
2. **一致性**：成千上万实例在秒级内看到同一份配置，不能一半新一半旧。
3. **隔离与权限**：多环境（dev/test/prod）、多租户、按服务/集群粒度隔离，且变更要有审计。
4. **容灾**：配置中心挂了，服务必须能启动、能运行——**不能用配置中心做单点**。

---

## 二、推模式 vs 拉模式（核心考点）

| 模式 | 机制 | 优点 | 缺点 |
|------|------|------|------|
| **推（Push）** | 服务端变更后主动通知客户端 | 实时性最好 | 服务端要维护海量长连接；网络抖动易丢通知 |
| **拉（Pull）** | 客户端定时轮询 | 实现简单、天然容错 | 实时性差（分钟级），实例多了打爆服务端 |
| **长轮询（Long Polling）** | 客户端发起请求挂起 30s，有变更立即返回，超时再发起 | 兼具实时性与简单性 | 需要服务端支持挂起连接（Nacos/Apollo 的核心） |

**长轮询是工业界主流**（Nacos、Apollo 都是这个思路）：客户端带 `MD5/lastVersion` 请求，服务端比对；无变更就 hold 住不返回，最多 30s 超时；期间有变更立即响应。效果接近"推"，但客户端逻辑简单、服务端无状态压力可控。

```
客户端                                服务端
  │  ① GET /config?md5=abc&timeout=30s  │
  │ ───────────────────────────────────►│  比对 md5
  │                                     │  无变更 → 挂起（hold）
  │                                     │  变更到来 → 立即返回新值
  │  ② 200 {md5: def, data: ...}        │
  │ ◄───────────────────────────────────│
  │  ③ 应用新配置 + 本地快照落盘         │
  │  ④ 立即发起下一轮长轮询              │
```

---

## 三、Go 客户端：长轮询 + 本地快照

```go
type ConfigClient struct {
	baseURL  string
	current  atomic.Value // map[string]string，无锁热更新
	snapshot string       // 本地兜底文件
}

// Run 后台长轮询，配置变更时原子替换
func (c *ConfigClient) Run(ctx context.Context) {
	md5 := ""
	for {
		select {
		case <-ctx.Done():
			return
		default:
		}

		// 长轮询：服务端 hold 最多 30s，有变更立即返回
		pctx, cancel := context.WithTimeout(ctx, 35*time.Second)
		resp, err := c.longPoll(pctx, md5)
		cancel()
		if err != nil {
			select {
			case <-time.After(time.Second): // 失败退避，避免空转打爆服务端
			case <-ctx.Done():
				return
			}
			continue
		}

		if resp.MD5 != md5 {
			md5 = resp.MD5
			c.current.Store(resp.Data)             // 原子替换，读方无锁
			_ = os.WriteFile(c.snapshot, resp.Raw, 0644) // 本地快照兜底
		}
	}
}

// Get 读取当前配置（内存读，纳秒级）
func (c *ConfigClient) Get(key string) string {
	m, _ := c.current.Load().(map[string]string)
	return m[key]
}

// 启动时：先读本地快照保证能起来，再异步拉最新
func (c *ConfigClient) Bootstrap(ctx context.Context) error {
	if raw, err := os.ReadFile(c.snapshot); err == nil {
		_ = c.load(raw) // 快照兜底：配置中心挂了我们也能启动
	}
	go c.Run(ctx)
	return nil
}
```

**三个必须设计到的点**：

- **本地快照（fail-static）**：启动先读本地磁盘快照 → 配置中心全挂也能启动；拉不到新配置就继续用旧的。
- **原子替换 + 只读**：配置用 `atomic.Value` 整体替换，**不要边读边改 map**（`fatal error: concurrent map read and map write`）。
- **变更回调要幂等**：如日志级别、限流阈值，通常直接读内存；但**连接池大小、线程数**这类不可热更的，要明确约定"需重启"。

---

## 四、配置灰度与回滚

| 手段 | 说明 |
|------|------|
| **按实例灰度** | 先推 1~5% 实例，观察指标再全量（Nacos 的 beta/gray 发布） |
| **灰度标签** | 按集群/IP/region 选择目标实例 |
| **版本对比 + 一键回滚** | 服务端保留历史版本与 diff，出问题 1 秒回滚 |
| **变更审计** | 谁在什么时候改了什么、影响了多少实例 |

---

## 五、常见坑

| 坑 | 说明 |
|----|------|
| 配置中心成单点 | 必须有本地快照 + 多副本 + 跨机房部署 |
| 变更风暴 | 一次推全量 → 所有实例同时重连/重建连接池 → 雪崩，要灰度 |
| 配置覆盖代码默认值 | 缺省值语义要明确：空字符串 ≠ 未配置（用指针或 `map` 存在性判断） |
| 把 secret 明文放配置中心 | 敏感项要加密存储 + KMS 托管，或走专门的 Secret 体系 |
| 用配置中心存"业务数据" | 配置是低频、少量、全局的；业务数据走 DB/缓存 |

---

## 面试话术

> **"配置中心不是远程 Git，核心是'动态生效 + 一致性 + 容灾'三件事。工业界主流是长轮询：客户端带 md5 请求，服务端无变更就 hold 30 秒，有变更立即返回——效果接近推，但客户端简单、服务端可控。客户端我会用原子替换做无锁热更新，并且一定落本地快照，保证配置中心挂了服务还能启动和运行。发布上要做灰度 + 一键回滚，绝不能一次推全量，否则所有实例同时重建连接池会雪崩。"**
