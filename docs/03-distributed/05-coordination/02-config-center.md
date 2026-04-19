# 配置中心选型与实践

> 考察频率：★★★☆☆  优先级：P2
> 关键词：Apollo、Nacos、etcd、热更新、长轮询、配置灰度、本地缓存

## 面试官考察意图

这道题考察候选人对**配置管理**的系统性认知，不只是会用 SDK。高级工程师会关注：
- 为什么配置和代码必须分离（热更新、运维效率）
- 推送 vs 拉取的优劣对比（etcd Watch 是推送，Nacos 长轮询是伪推送）
- 配置灰度发布如何实现（按百分比/标签/机房）
- 配置中心故障时的容灾策略（本地缓存兜底）

初级工程师往往只知道"配置中心就是存配置的"。

---

## 核心答案（30 秒版）

**配置中心解决的是"代码不动、配置动"的问题，让配置变更无需重新部署。**

三大核心机制：
- **热更新**：推送（etcd Watch）或伪推送（Nacos 长轮询），变更实时生效
- **灰度发布**：按标签/机房/百分比渐进式推送，先小流量验证
- **容灾**：本地文件缓存兜底，注册中心挂了也能跑

选型结论：
- 阿里技术栈 → **Nacos**（零成本集成阿里云）
- K8s 生态 + Go 服务 → **etcd**（k8s 原生，原生支持 Watch）
- 企业级功能需求（权限、审计） → **Apollo**（功能最全）

---

## 深度展开

### 1. 为什么需要配置中心

#### 配置 vs 代码 不分离的问题

```
❌ 配置写在代码/配置文件里的问题：

1. 修改配置 → 需要重新打包/部署
2. 多环境（dev/test/staging/prod）配置不同
3. 配置需要保密（数据库密码不能放代码仓库）
4. 同一个服务多个实例，需要批量改
5. 配置变更无法追溯，谁改了什么
```

**配置中心的价值：**
```
┌──────────────────────────────────────────────────────┐
│                    配置中心                           │
│                                                      │
│  Config Server ◀──长轮询/Watch── 配置变更推送         │
│       │                                             │
│       ├──▶ Service A (实例1)                         │
│       ├──▶ Service A (实例2)                        │
│       └──▶ Service A (实例3)                        │
│                                                      │
│  ✅ 改配置，实时生效，无需重启                         │
│  ✅ 权限管控，变更审计                                │
│  ✅ 多环境隔离                                        │
└──────────────────────────────────────────────────────┘
```

---

### 2. 配置热更新原理

#### 方案一：长轮询（Nacos / Apollo）

```
客户端                          服务端
  │                               │
  │─── GET /config?fd=xxxx ──────▶│  收到请求，hold 住连接（30秒）
  │                               │  如果超时或配置变更，返回响应
  ◀─── 30000ms 超时返回 ──────────│
  │                               │
  │─── 立即再次发起请求 ───────────▶│  循环不断
  │                               │
```

**长轮询的问题：**
- 不是真正的推送，有 30s 延迟（最佳情况）
- 服务端压力大，每个实例都 hold 一个 HTTP 连接
- 伪推送，本质还是"拉"

#### 方案二：gRPC Watch（etcd）

```go
// etcd Watch 是真正的推送
ch := client.Watch(ctx, "config/orderservice/", clientv3.WithPrefix())

go func() {
    for wresp := range ch {
        for _, ev := range wresp.Events {
            // 真正的推送，毫秒级延迟
            fmt.Printf("配置变更: %s = %s\n", ev.Kv.Key, ev.Kv.Value)
            // 重新加载配置到内存
            reloadConfig()
        }
    }
}()
```

**etcd Watch 的优势：**
- 毫秒级推送（不是 30s 延迟）
- 基于 MVCC，可重放历史
- gRPC 长连接，比 HTTP 长轮询资源效率更高

#### 方案三：广播 + 回调（配置中心 SDK 内部实现）

```
Nacos/Apollo SDK 内部：
┌──────────────────────────────────────────────┐
│  SDK                                          │
│  ├── 本地缓存文件（故障兜底）                   │
│  ├── 长轮询协程（定时拉取）                     │
│  └── 变更回调（通知业务代码）                   │
└──────────────────────────────────────────────┘

1. SDK 启动 → 拉取全量配置 → 写入本地文件
2. 后台长轮询协程定时询问配置中心"有变化吗"
3. 有变化 → 下载最新配置 → 写入本地文件 → 触发回调
4. 配置中心挂了 → 从本地文件读取（降级）
```

---

### 3. Go 项目接入配置中心的代码模式

#### 统一配置封装（基于 Nacos，Apollo 同理）

```go
package config

import (
    "fmt"
    "os"
    "sync"

    "github.com/nacos-group/nacos-sdk-go/v2/clients"
    "github.com/nacos-group/nacos-sdk-go/v2/clients/config_client"
    "github.com/nacos-group/nacos-sdk-go/v2/vo"
)

type Config struct {
    mu         sync.RWMutex
    dbHost     string
    dbPort     int
    dbPassword string  // 敏感配置
    featureX   bool
    qpsLimit   int
}

var globalConfig *Config

// 初始化 Nacos 配置客户端
func InitConfig() error {
    cli, err := clients.NewConfigClient(vo.NacosClientParam{
        ServerConfigs: []constant.ServerConfig{
            {IpAddr: "127.0.0.1", Port: 8848},
        },
        ClientConfig: &constant.ClientConfig{
            NamespaceId: "prod",
            TimeoutMs:   5 * 1000,
        },
    })
    if err != nil {
        return err
    }

    cfg := &Config{}
    globalConfig = cfg

    // 监听配置变更
    err = cli.ListenConfig(vo.ConfigParam{
        DataId:: "order-service",
        Group:   "DEFAULT_GROUP",
        OnChange: func(namespace, group, dataId, data string) {
            fmt.Println("配置变更，重新加载")
            cfg.reload(data)
        },
    })
    if err != nil {
        return err
    }

    // 首次拉取
    content, err := cli.GetConfig(vo.ConfigParam{
        DataId: "order-service",
        Group:  "DEFAULT_GROUP",
    })
    if err != nil {
        return err
    }
    cfg.reload(content)

    return nil
}

func (c *Config) reload(content string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    // 解析 YAML/JSON 格式配置
    // 这里用简单的模拟
    fmt.Sscanf(content, "db_host=%s db_port=%d qps=%d",
        &c.dbHost, &c.dbPort, &c.qpsLimit)
}

func (c *Config) DBPassword() string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.dbPassword  // 敏感配置走环境变量，不走配置中心
}
```

#### etcd 配置热更新

```go
package config

import (
    "context"
    "fmt"
    "sync"

    "go.etcd.io/etcd/client/v3"
)

type ConfigManager struct {
    cli     *clientv3.Client
    config  map[string]string
    mu      sync.RWMutex
    watchers []func(map[string]string)
}

func NewConfigManager(endpoints []string) (*ConfigManager, error) {
    cli, err := clientv3.New(clientv3.Config{
        Endpoints:   endpoints,
        DialTimeout:  5,
    })
    if err != nil {
        return nil, err
    }

    cm := &ConfigManager{cli: cli, config: make(map[string]string)}
    go cm.watchLoop()
    return cm, nil
}

func (cm *ConfigManager) watchLoop() {
    ch := cm.cli.Watch(context.Background(), "config/",
        clientv3.WithPrefix())

    for wresp := range ch {
        for _, ev := range wresp.Events {
            key := string(ev.Kv.Key)
            value := string(ev.Kv.Value)

            cm.mu.Lock()
            if ev.Type == clientv3.EventTypeDelete {
                delete(cm.config, key)
            } else {
                cm.config[key] = value
            }
            cm.mu.Unlock()

            // 通知所有监听器
            cm.mu.RLock()
            snapshot := cm.config
            cm.mu.RUnlock()
            for _, fn := range cm.watchers {
                fn(snapshot)
            }
        }
    }
}

func (cm *ConfigManager) RegisterWatcher(fn func(map[string]string)) {
    cm.watchers = append(cm.watchers, fn)
}

func (cm *ConfigManager) Get(key string) (string, bool) {
    cm.mu.RLock()
    defer cm.mu.RUnlock()
    v, ok := cm.config[key]
    return v, ok
}
```

---

### 4. 配置灰度发布

#### 灰度策略

| 策略 | 说明 | 实现方式 |
|------|------|---------|
| **百分比灰度** | 先给 1% 流量，验证后再扩到 10%/50%/100% | 配置中心按 ID 取模 |
| **标签灰度** | 按服务实例标签（region=sh, env=canary） | Nacos metadata 过滤 |
| **机房灰度** | 先推北京机房，再推上海 | 标签 + 配置分组 |
| **白名单** | 指定 IP/用户 ID | 配置中心按 IP 列表 |

#### 代码实现

```go
package config

import (
    "hash/fnv"
    "strconv"
)

// 百分比灰度：instanceId 取模
func shouldEnable(cfg *GrayConfig, instanceID string) bool {
    if cfg.Percent >= 100 {
        return true
    }
    if cfg.Percent <= 0 {
        return false
    }

    h := fnv.New32a()
    h.Write([]byte(instanceID))
    hash := h.Sum32()

    return (hash % 100) < cfg.Percent
}

type GrayConfig struct {
    Percent int      `json:"percent"`
    Tags    []string `json:"tags"`  // 标签白名单
    IPs     []string `json:"ips"`   // IP 白名单
}

// 按标签灰度
func matchTags(cfg *GrayConfig, myTags []string) bool {
    if len(cfg.Tags) == 0 {
        return true  // 没有标签限制，全量
    }
    for _, tag := range cfg.Tags {
        for _, my := range myTags {
            if tag == my {
                return true
            }
        }
    }
    return false
}
```

---

### 5. 配置中心选型对比

| 维度 | Apollo | Nacos | etcd |
|------|--------|-------|------|
| **语言生态** | Java 为主 | Java/Go/Python | Go 原生 |
| **功能完善度** | ⭐⭐⭐⭐⭐ 权限/审计/灰度/多环境 | ⭐⭐⭐⭐ 基础完善 | ⭐⭐ 功能弱 |
| **K8s 集成** | 一般 | 一般 | ⭐⭐⭐⭐⭐ k8s 原生 |
| **热更新延迟** | ~3 秒（长轮询） | ~1 秒（长轮询+MD5） | 毫秒级（Watch） |
| **运维复杂度** | 高（Java + MySQL + Config）| 中（单节点可跑） | 低（单二进制）|
| **Go SDK** | 一般（社区维护）| 好 | ⭐⭐⭐⭐⭐ 官方 clientv3 |
| **推荐场景** | 企业内部，有合规要求 | 阿里技术栈，中小型系统 | K8s 生态，Go 微服务 |
| **上线成本** | 高（依赖多） | 低 | 极低 |
| **配置上限** | 百万级 | 十万级 | 万级 |

---

### 6. 高可用设计：配置中心挂了怎么办？

#### 三级降级策略

```
第一级：本地缓存（SDK 自动降级）
    └── Nacos SDK 启动时写入本地文件
    └── 配置中心挂了 → 读本地文件

第二级：环境变量 / Kubernetes Secret
    └── 数据库密码等敏感配置走环境变量
    └── 不依赖配置中心

第三级：配置默认值（代码硬编码兜底）
    └── 读取失败时用代码中的默认值
    └── 保证服务能启动
```

```go
// 启动时加载配置的优先级
func loadConfig() (*AppConfig, error) {
    cfg := &AppConfig{}

    // 1. 先尝试从配置中心读取
    if nacosClient != nil {
        content, err := nacosClient.GetConfig(...)
        if err == nil {
            parseConfig(cfg, content)
            return cfg, nil  // 配置中心 OK，直接用
        }
        log.Printf("配置中心读取失败: %v，降级到本地缓存", err)
    }

    // 2. 配置中心失败 → 读本地缓存文件
    if data, err := os.ReadFile("config/local-cache.json"); err == nil {
        parseConfig(cfg, string(data))
        return cfg, nil
    }

    // 3. 本地文件也失败 → 用默认值
    log.Printf("所有配置源均失败，使用默认值")
    cfg.QPSLimit = 1000
    cfg.Timeout = 5 * time.Second
    return cfg, nil
}
```

---

## 高频追问

### Q1: 配置变更怎么保证原子性？

**答：** 配置中心本身不保证原子性（配置变更是覆盖，不是事务）。真正的做法：
1. **两阶段更新**：先写入临时 key（如 `config/v2`），再 rename 到正式 key（Nacos 支持）
2. **版本回滚**：Apollo 支持配置回滚，Nacos 支持历史版本查看
3. **灰度 + 回滚**：先灰度 1%，观察没问题再全量

### Q2: 配置中心如何保证一致性？

**答：** Nacos/Apollo 用**最终一致性**（长轮询拉取），etcd 用**强一致性**（Raft 多数派）。对于配置场景，最终一致性足够，因为：
- 配置变更本身不要求毫秒级同步
- 业务设计上，同一 key 的多个值不会同时生效
- 强一致性对性能损耗大，不适合配置推送场景

### Q3: etcd 能当配置中心用吗？和专门配置中心的区别？

**答：** 可以，但有限制：
- ✅ 优势：Watch 毫秒级推送，Go 生态好，K8s 原生
- ❌ 劣势：无权限管理、无灰度发布、无 UI、无审计日志
- 建议：小型 Go 微服务（K8s 部署）用 etcd；中大型系统用 Apollo/Nacos

---

## 延伸阅读

| 资料 | 链接 |
|------|------|
| Apollo 官方文档 | https://www.apolloconfig.com/ |
| Nacos 官方文档 | https://nacos.io/ |
| etcd 配置热更新实战 | https://github.com/etcd-io/etcd/blob/main/client/v3/namespace/kv.go |
| 配置中心架构设计 | https://developer.aliyun.com/article/606 |
