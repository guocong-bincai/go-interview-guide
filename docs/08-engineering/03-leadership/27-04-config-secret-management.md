# 配置与密钥管理工程化：12-Factor Config + Secret 轮转

> 考察频率：★★★★☆  优先级：P1
> 关键词：12-Factor、配置外置、环境变量、动态配置、Secret 管理、KMS/Vault、轮转、热更新

## 面试官考察意图

"你们配置怎么管理？密钥放哪？改配置要不要重启？"——这道题区分**"能跑"和"能上线运营"**的工程师：

1. **配置外置**（12-Factor 的第三条）：代码与配置严格分离，同一份二进制能在不同环境跑；
2. **配置分层**（默认值 / 环境 / 动态）与**加载优先级**；
3. **动态配置热更新**（不重启生效）与**本地快照兜底**；
4. **Secret 管理**（不进 Git、加密存储、KMS/Vault、轮转、最小权限）；
5. **配置验证与故障隔离**（配置写错不能让服务起不来或静默降级）。

**核心立场：配置是"运行期的代码"，Secret 是"最高权限的配置"——两者都必须外置、可校验、可回滚、可轮转。**

---

## 核心答案（30 秒版）

**12-Factor Config**：配置（尤其密钥）**外置于代码**，通过**环境变量**注入，`go build` 出的同一份二进制在 dev/staging/prod 通用。

**配置三层**：
1. **默认值**（代码内，安全兜底）；
2. **静态配置**（环境变量 / 配置文件，启动时加载）；
3. **动态配置**（配置中心，运行时可改）。

**加载优先级**：环境变量 > 配置文件 > 代码默认值（越"临时/具体"的越优先）。

**Secret 五条军规**：
1. **永不进 Git**（`.env` 加 `.gitignore`，扫描工具防泄漏）；
2. **加密存储 + 运行时注入**（Vault / KMS / K8s Secret，而非明文文件）；
3. **最小权限**（一个服务只拿它需要的 secret）；
4. **可轮转**（定期换 + 双钥匙滚动切换，不中断服务）；
5. **日志/错误里脱敏**（secret 绝不打印）。

**一句话**：**配置外置解决"一份代码多环境"，Secret 管理解决"泄漏了也不慌"。**

---

## 深度展开

### 一、Go 里的配置加载：优先级与校验

```go
type Config struct {
    Addr        string        `env:"ADDR"          default:":8080"`
    DBDSN       string        `env:"DB_DSN"        required:"true"`
    RedisAddr   string        `env:"REDIS_ADDR"    default:"localhost:6379"`
    Timeout     time.Duration `env:"REQ_TIMEOUT"   default:"3s"`
}

func Load() (*Config, error) {
    cfg := &Config{}
    // 优先级：环境变量 > .env(开发) > 结构体 default tag
    if err := env.Parse(cfg); err != nil {
        return nil, fmt.Errorf("load config: %w", err)
    }
    if err := validate(cfg); err != nil { // 👈 启动即校验，fail fast
        return nil, fmt.Errorf("invalid config: %w", err)
    }
    return cfg, nil
}
```

要点：
- **启动时校验所有必填项**，缺了就**拒绝启动**（fail fast），别等运行到一半才炸；
- **默认值必须安全**（如超时默认 3s，而非 0=无超时）；
- 用 `github.com/caarlos0/env` 或 `kelseyhightower/envconfig` 减少手写样板。

### 二、配置分层与热更新

不是所有配置都能热更——要区分：

| 配置类型 | 例子 | 热更新 |
|---|---|---|
| 启动期配置 | 监听端口、DB 地址、连接池大小 | ❌ 改了要重启 |
| 运行期配置 | 限流阈值、开关、超时、采样率 | ✅ 可热更 |
| 敏感配置 | DB 密码、API Key | ✅ 但走轮转流程 |

**热更新实现（原子指针快照）**：

```go
type RuntimeConfig struct {
    RateLimit    int
    SampleRate   float64
    FeatureXOn   bool
}

type ConfigStore struct {
    cur atomic.Pointer[RuntimeConfig] // 不可变快照，读无锁
}

func (s *ConfigStore) Get() *RuntimeConfig { return s.cur.Load() }

// 配置中心推送 / 长轮询拉取后调用
func (s *ConfigStore) Update(c RuntimeConfig) {
    s.cur.Store(&c) // 整块替换，读侧永远看到一致快照
}
```

```go
// 业务侧：每次读最新值，天然线程安全
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if !h.cfg.Get().FeatureXOn {
        http.Error(w, "disabled", http.StatusServiceUnavailable)
        return
    }
    // ...
}
```

> ⚠️ **坑：局部字段修改会导致读写竞态**。正确做法是**整块原子替换**，而不是 `cfg.RateLimit = x`。

### 三、本地快照兜底（高可用关键）

配置中心的可用性**低于**业务服务时，不能让配置中心抖动导致全站挂掉：

```text
启动时：从配置中心拉取 → 成功后写入本地快照（文件/etcd）
拉取失败：降级读本地快照（最后一次成功值）
本地也没有：才用代码默认值
```

```go
func (m *ConfigManager) Watch(ctx context.Context) error {
    snapshot, err := m.remote.Get(ctx)
    if err != nil {
        if data, e := os.ReadFile(m.snapshotPath); e == nil {
            _ = json.Unmarshal(data, &snapshot) // 降级到本地快照
        } else {
            return fmt.Errorf("config center down and no snapshot: %w", err)
        }
    } else {
        _ = os.WriteFile(m.snapshotPath, mustJSON(snapshot), 0o600)
    }
    m.store.Update(snapshot)
    return nil
}
```

### 四、Secret 管理：绝不明文进仓库

**反面教材**：

```go
const dbPassword = "prod_p@ssw0rd_2024"  // ❌ 硬编码，永久泄漏
```

**正确姿势**：

```text
开发阶段：.env（.gitignore）+ 假值
CI/CD：  Secret 注入（GitHub Secrets / GitLab CI Variables）
生产：   Vault / KMS / AWS Secrets Manager / K8s Secret（加密存储）
运行时： 环境变量或挂载文件，进程内只在内存中持有
```

```go
func NewDB() (*sql.DB, error) {
    // 从环境变量读取（由 Vault Agent / K8s 注入）
    dsn := os.Getenv("DB_DSN")
    if dsn == "" {
        return nil, errors.New("DB_DSN not set")
    }
    // 绝不打日志：包裹类型实现 String() 返回 "***"
    return sql.Open("mysql", dsn)
}
```

**防泄漏扫描**：CI 里跑 `gitleaks` / `trufflehog`，禁止明文密钥进仓库。

### 五、Secret 轮转（高级考点）

**双钥匙滚动轮转**（不中断服务）：

```text
1. 生成新密钥 new，与旧密钥 old 并存（服务同时接受两者）
2. 在密钥管理系统中写入 new
3. 通知所有服务加载 new（开始用 new 签发/加密，仍接受 old 校验）
4. 观察 1 个周期，确认无 old 流量
5. 删除 old
```

数据库密码轮转同理：**先改权限同时支持新旧密码，再切换，最后回收旧密码**——绝不能"直接改密码然后重启所有服务"。

### 六、常见坑清单

- **配置写错导致服务起不来**：加校验 + 灰度发布新配置；
- **Secret 打进日志**：DSN/Key 包一层类型 + 脱敏；
- **环境变量里塞大量配置**：超过几千字节不适合，用文件挂载；
- **动态配置无版本/无审计**：配置变更要留痕、可回滚；
- **`os.Getenv` 到处散落**：集中在 `Config` 结构体，禁止业务层直接读环境变量。

---

## 面试话术

> "配置我们严格遵守 12-Factor，代码和配置分离，同一份二进制靠环境变量在不同环境跑，启动时集中加载并强校验必填项，缺了就 fail fast 拒绝启动。配置分三层——代码默认值、环境变量、配置中心的动态配置，优先级是环境变量最高。需要热更的只有运行期配置，比如限流阈值、开关，实现上用一个 `atomic.Pointer` 存不可变快照，整块替换保证无锁读且不会读到半更新状态。Secret 一律不进 Git，生产走 Vault/KMS 或 K8s Secret 注入，CI 里跑 gitleaks 防泄漏；轮转走双钥匙——新老并存、逐步切换、最后回收，不会因为改密码导致服务中断。配置中心挂了会降级到本地快照兜底。"

---

## 高频追问

- **Q：配置热更新怎么保证线程安全？** A：存不可变快照，用 `atomic.Pointer` 整块替换，读侧无锁；禁止局部字段修改。
- **Q：配置中心挂了怎么办？** A：本地快照兜底（最后一次成功值），配置中心可用性不应成为业务单点。
- **Q：Secret 怎么轮转不中断？** A：双钥匙模式，新旧并存、灰度切换、观察后回收。
- **Q：所有配置都能热更吗？** A：不能。端口、连接池这类启动期配置改了必须重启；只有幂等可替换的运行期配置才热更。
