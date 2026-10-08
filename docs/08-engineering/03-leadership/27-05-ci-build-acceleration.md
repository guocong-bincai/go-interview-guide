# CI 构建提速工程化：缓存、并行分片与远程构建

> 考察频率：★★★★☆  优先级：P1
> 关键词：Go build cache、GOCACHE、测试分片、并行、远程缓存、Bazel/Please、增量构建

## 面试官考察意图

"你们 CI 一次跑多久？怎么优化的？"——这是**能直接体现工程产出**的题。它考察候选人是否理解构建/测试流水线的**瓶颈定位**与**提速手段**：

1. **Go 构建缓存**（`GOCACHE`/`GOMODCACHE`、content-addressed）；
2. **测试分片**（sharding）与 `t.Parallel`；
3. **CI 缓存命中率**与**缓存失效**的根因；
4. **远程缓存/远程构建**（Bazel、Please、构建农场）何时值得上。

**核心立场：CI 提速不是玄学——先量化（哪一步慢），再缓存（避免重复做同样的事），然后并行（能拆就拆），最后才考虑分布式构建。**

---

## 核心答案（30 秒版）

**提速三板斧（按性价比排序）**：

1. **缓存**：Go 的 build cache 是 content-addressed 的——**输入不变则复用**。
   ```bash
   cache:
     paths:
       - ~/.cache/go-build     # GOCACHE
       - ~/go/pkg/mod          # GOMODCACHE
   ```
   CI 里 **`actions/setup-go` 的 `cache: true`** 自动缓存这两者。

2. **并行/分片**：
   ```bash
   go test -p $(nproc) ./...          # 包级并行
   go test -shuffle=on ./...          # 打乱用例顺序暴露隐性依赖
   # 大仓库用 gotestsum / 手工分片：把包列表切成 N 份，N 个 job 并行
   ```

3. **减少重复工作**：
   - 只跑**受影响的包**（`go test` 依赖分析或 `bazel query`）；
   - `-count=1` 关闭测试结果缓存时要谨慎（CI 通常**希望**缓存，别乱加）；
   - 用 `-race` 只在需要的 job 跑，别全量。

**一句话**：**先看 `go build -x`/CI timing 找瓶颈，再把"不变的输入"缓存掉，把"可切的单元"并行掉。**

---

## 深度展开

### 一、Go 构建缓存原理（面试常考）

Go 的构建缓存是 **content-addressed**：每条缓存条目的 key 由**所有输入**的哈希决定（源码、编译器版本、构建标志、依赖版本、环境变量如 `GOOS/GOARCH/CGO_ENABLED`）。

- **命中则复用**，跳过编译；
- **任意输入变化 → key 变 → 缓存 miss**，重编译。

```bash
go env GOCACHE      # 默认 ~/Library/Caches/go-build (macOS) / ~/.cache/go-build (linux)
go env GOMODCACHE   # ~/go/pkg/mod
go build -x ./...   # 看实际执行了哪些命令，定位 miss
```

**缓存失效的常见根因**（高频追问）：
- CI 里 **`git checkout` 带来新的 mtime**？Go 不靠 mtime，靠内容哈希，**不受影响** ✅；
- **`-trimpath`、build tag、ldflags 变化** → key 变（注意别在每次构建注入随机值，如 `-X main.commit=$(date)`）；
- **`CGO_ENABLED` 不一致**；
- **缓存目录没被持久化**（CI 换 runner）。

> 关键：**注入构建时间戳/随机值到二进制会让缓存永远 miss**。要"可复现构建"就不要这么做。

### 二、测试分片（大仓库必备）

单 job 跑几万个测试用例很慢，按**包**切成 N 片并行：

```bash
# 把包列表切成 4 份（简单取模示例）
pkgs=$(go list ./... | awk 'NR % 4 == '"$SHARD"'' )
go test -p $(nproc) $pkgs
```

```yaml
# GitHub Actions：matrix 分片
jobs:
  test:
    strategy:
      matrix:
        shard: [0, 1, 2, 3]
    steps:
      - run: |
          export SHARD=${{ matrix.shard }}
          ./scripts/test-shard.sh
```

**要点**：
- **分片要均衡**：按包的**历史耗时**切（大包别都落一片），否则最慢的一片决定总时长；
- 用 `gotestsum` 汇总各分片结果、生成统一报告；
- **`t.Parallel()`** 让包内用例并行，但注意共享状态（全局变量、端口、DB）会引入 flaky。

### 三、CI 缓存配置示例

```yaml
# GitHub Actions
- uses: actions/setup-go@v5
  with:
    go-version: 'stable'
    cache: true                    # 自动缓存 GOCACHE + GOMODCACHE

# 或手工（GitLab CI）
variables:
  GOCACHE: $CI_PROJECT_DIR/.cache/go-build
  GOMODCACHE: $CI_PROJECT_DIR/.cache/go-mod
cache:
  key: go-$CI_COMMIT_REF_SLUG
  paths:
    - .cache/go-build
    - .cache/go-mod
```

**缓存命中率排查**：观测 `go build -x` 是否大量重复编译；缓存 key 设计要包含 **go 版本 + go.sum 哈希**。

### 四、只跑受影响的代码（增量构建）

大型 monorepo 里，改一个包不该重测全仓库：

```bash
# 找出受影响的包（被改动的包 + 依赖它的包）
go list -deps ./changed/pkg/...         # 被依赖方向
# 或直接用工具：bazel/please 做依赖图增量
```

- 小仓库：`go test ./...` 也够快（有缓存）；
- 大仓库：上 **Bazel/Please** 这类支持**精确依赖图 + 远程缓存**的构建系统。

### 五、远程缓存 / 远程构建

当本地/CI 缓存无法跨机器共享，或机器数量多时：

```text
本地缓存 → CI 缓存(persist) → 远程缓存(共享, 如 Bazel remote cache / sccache)
                                 → 远程执行(把编译单位分发到构建农场)
```

- **Bazel + remote cache**：任意机器命中同一 key 的产物，CI 冷启动也能秒级；
- **`sccache`**：给 C/C++/Rust 做远程编译缓存（Go 有内建缓存，一般不需要）；
- **代价**：引入构建系统复杂度，小团队不划算——**先做缓存 + 分片，收益最大、成本最低**。

### 六、其他提速点

| 手段 | 收益 | 注意 |
|---|---|---|
| 缓存 GOCACHE/GOMODCACHE | ★★★★★ | key 含 go 版本 |
| 按包分片 + matrix | ★★★★☆ | 分片要均衡 |
| `-p $(nproc)` | ★★★ | 内存可能不够 |
| 只跑受影响包 | ★★★★ | 需要依赖图 |
| 减少 `-race` 范围 | ★★★ | race 只在关键 job |
| 容器镜像层缓存 | ★★★ | 见 Docker 优化篇 |
| 远程构建 | ★★ | 复杂度高 |
| 编译时 PGO | ★★ | 见 PGO 篇 |

### 七、可复现构建与缓存的张力

- 想要**可复现**（同一 commit → 同一二进制），就**不能**注入时间戳/随机 commit；
- 想要**缓存命中**，也需要输入稳定；
- **两者方向一致**：稳定输入 = 高缓存命中 + 可复现。详见《可复现构建》一篇。

---

## 面试话术

> "CI 提速我一般按三步走。第一是**缓存**——Go 的 build cache 是内容寻址的，输入不变就复用，我们缓存 `GOCACHE` 和 `GOMODCACHE`，key 里带上 Go 版本和 go.sum，命中后编译基本跳过。第二是**并行分片**——把包列表按历史耗时切成 N 片用 matrix 跑，包内再用 `t.Parallel`；分片一定要均衡，否则最慢那片决定总时长。第三是**只跑受影响的包**，monorepo 里改一个包不该重测全仓库。踩过的坑主要是缓存永远 miss——比如构建时往二进制注入时间戳或随机 commit，导致输入变化缓存失效，这跟可复现构建的原则其实是冲突的。"

---

## 高频追问

- **Q：为什么 Go 缓存不看 mtime？** A：它是 content-addressed，key 由输入内容哈希决定，所以 checkout 改变 mtime 不影响命中。
- **Q：CI 里要不要加 `-count=1`？** A：加了会禁用测试结果缓存。CI 通常希望复用缓存，除非要强制重跑，否则别加。
- **Q：什么时候该上 Bazel？** A：仓库足够大、构建成为瓶颈、需要跨机器共享缓存和精确增量时；小项目用 Go 自带缓存 + 分片足够。
- **Q：分片后总时长没降？** A：多半是分片不均衡（某片包太大）或存在串行依赖，应按历史耗时切分。
