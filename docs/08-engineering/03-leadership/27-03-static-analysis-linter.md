# 静态代码检查工程化：golangci-lint 落地与误报治理

> 考察频率：★★★★☆  优先级：P1
> 关键词：golangci-lint、staticcheck、go vet、误报治理、增量检查、CI 门禁、gofumpt

## 面试官考察意图

"你们代码质量怎么保证？CI 里跑 lint 吗？"——这是工程素养模块的**必答题**，考察候选人是否真的在团队里落地过静态检查，而不只是"听说过 golangci-lint"：

1. **`go vet` 与 golangci-lint 的分工**（vet 是官方、必跑；golangci-lint 是聚合器）；
2. **linter 的选择与启用策略**（全开会被误报淹没，全关等于没有）；
3. **误报治理**（`//nolint` 的规范、baseline、逐步收紧）；
4. **CI 集成**（增量检查、缓存、gofmt 强约束、新代码零新增问题）。

**核心立场：静态检查的价值不在于"跑了一堆 linter"，而在于"让团队可持续地维持一条质量基线"——要么全绿，要么有明确豁免理由。**

---

## 核心答案（30 秒版）

**两层体系**：
- **`go vet`**：Go 官方自带，检查**必然的错误**（格式化串参数不匹配、锁值拷贝、nil 比较等），**必须进 CI，零容忍**；
- **`golangci-lint`**：聚合几十个 linter 的**超级 linter**，统一配置、并行执行、支持缓存与自动修复。

**落地四原则**：
1. **从最小集合起步**：先开 `govet`/`staticcheck`/`errcheck`/`ineffassign`/`unused`/`gofmt`，跑通后再加；
2. **CI 用增量门禁**：`golangci-lint run --new-from-rev=origin/main`——**只管本次改动新增的问题**，老代码不阻塞；
3. **豁免要留痕**：`//nolint:errcheck // 这里忽略是因为 xxx`，禁止裸 `//nolint`；
4. **格式强约束**：`gofmt`/`gofumpt` + `goimports` 进 pre-commit，别让人在 review 里吵架。

**一句话**：**静态检查是"用机器守住底线"，底线要能全绿，例外要能追溯。**

---

## 深度展开

### 一、`go vet` vs `golangci-lint`

```bash
# 官方 vet，检查编译器不会报但几乎必然是 bug 的写法
go vet ./...

# golangci-lint，聚合器
golangci-lint run ./...
```

| | go vet | golangci-lint |
|---|---|---|
| 来源 | 官方工具链 | 社区聚合器 |
| 检查范围 | 少量高置信度规则 | 几十种 linter（含 staticcheck） |
| 修复能力 | 无 | `--fix` 自动修部分问题 |
| 误报 | 极低 | 视 linter 而定 |
| 定位 | 底线 | 团队质量基线 + 风格统一 |

**`staticcheck`（SA/S/ST/QF 系列）** 是 golangci-lint 里最有价值的一组——它能抓到 `go vet` 抓不到的逻辑错误（如恒真条件、错误的 `time.Duration` 运算、无用赋值）。

### 二、`.golangci.yml` 推荐起步配置

```yaml
# .golangci.yml（v2 风格）
version: "2"
run:
  timeout: 5m
  tests: true

linters:
  default: none          # 不放开箱即用全集，按需开
  enable:
    - govet              # 官方必开
    - staticcheck        # 逻辑错误
    - errcheck           # 忽略的 error
    - ineffassign        # 无效赋值
    - unused             # 未使用变量/函数
    - gocritic           # 常见坑
    - bodyclose          # HTTP body 未关
    - sqlclosecheck      # sql.Rows 未关
    - rowserrcheck
    - contextcheck       # context 传递
    - errorlint          # errors.Is/As 误用
    - gofmt
    - goimports
  settings:
    errcheck:
      check-type-assertions: true
    staticcheck:
      checks: ["all"]
  exclusions:
    rules:
      # 测试文件放宽 errcheck
      - path: _test\.go
        linters: [errcheck, gosec]

issues:
  max-issues-per-linter: 0   # 不截断，避免"眼不见心不烦"
  max-same-issues: 0
```

### 三、CI 集成：增量门禁（关键实践）

全量检查一个历史项目会瞬间刷出几千条问题，团队会直接放弃。**正确做法是"新代码零新增问题"**：

```yaml
# .github/workflows/lint.yml
name: lint
on: [pull_request]
jobs:
  golangci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # 必须，才能 diff 到 base
      - uses: actions/setup-go@v5
        with: { go-version: 'stable' }
      - name: golangci-lint (增量)
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
          args: --new-from-rev=origin/${{ github.base_ref }} --timeout=5m
```

**核心参数 `--new-from-rev`**：只报告相对基线**新增**的问题。老代码的存量问题不阻塞本次 PR，团队可以在迭代中逐步清理，而不是一次性大重构。

### 四、误报治理：`//nolint` 的规范

```go
// ❌ 裸豁免：后人不知道为什么要忽略，可能掩盖真 bug
id, _ := strconv.Atoi(s) //nolint

// ✅ 带原因 + 精确 linter
//nolint:errcheck // 上游已保证此处为数字，保留仅为兼容旧数据
id, _ := strconv.Atoi(s)

// ✅ 行内范围控制
//nolint:gosec // G404: 此处非安全用途，允许 math/rand
n := rand.Intn(100)
```

治理策略：
1. **禁止裸 `//nolint`**：CI 加一个自定义 grep 或 `nolintlint` linter 强制必须带 linter 名和说明；
2. **豁免集中评审**：`//nolint` 视同技术债，review 时重点看；
3. **定期收敛**：每次迭代清一批存量问题，逐步把 `--new-from-rev` 收紧甚至改成全量。

```yaml
# nolintlint：强制豁免必须解释
linters:
  enable: [nolintlint]
  settings:
    nolintlint:
      require-explanation: true
      require-specific: true
      allow-unused: false
```

### 五、自动化修复与格式化

```bash
# 自动修复能修的问题（imports、格式、部分 gocritic）
golangci-lint run --fix ./...

# gofumpt 比 gofmt 更严格（多出的空行、分组等）
gofumpt -l -w .
```

**pre-commit 钩子**（防止脏代码进仓库）：

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/golangci/golangci-lint
    rev: v2.x
    hooks:
      - id: golangci-lint
        args: [--fix]
  - repo: https://github.com/segmentio/golines
    rev: v0.12.2
    hooks: [{ id: golines }]
```

### 六、常见高频 linter 及其抓到的真实问题

| linter | 抓什么 | 真实场景 |
|---|---|---|
| `errcheck` | 未处理的 error | `w.Write()` 返回值丢失导致写失败无感 |
| `bodyclose` | HTTP response body 未关闭 | 连接池泄漏、TIME_WAIT 暴涨 |
| `sqlclosecheck` | `sql.Rows`/`Stmt` 未关 | 数据库连接泄漏 |
| `contextcheck` | context 未按链传递 | 超时/取消传播断裂 |
| `errorlint` | `==` 比较 error、`%v` 而非 `%w` | 破坏 `errors.Is/As` |
| `gosec` | 安全漏洞 | 弱随机、SQL 拼接、硬编码密钥 |
| `staticcheck` | 逻辑错误 | 恒真、duration 计算错、无用赋值 |
| `ineffassign` | 赋值后从未使用 | 掩盖逻辑 bug |

---

## 面试话术

> "我们 CI 里是两层：`go vet` 是官方底线，零容忍必跑；`golangci-lint` 作为团队质量基线，用 `--new-from-rev` 做**增量门禁**——只卡本次改动新增的问题，不然一个历史项目一上来几千条告警团队会直接摆烂。linter 集合是从最小集开始逐步加的，重点是 `errcheck`、`bodyclose`、`sqlclosecheck`、`contextcheck` 这些能抓到真实线上事故的。豁免用 `//nolint:xxx // 原因`，禁止裸豁免，用 `nolintlint` 强制要解释，把豁免当技术债管理。格式化用 gofumpt 加 pre-commit，省得在 review 里纠结风格。"

---

## 高频追问

- **Q：历史项目一堆 lint 问题怎么办？** A：`--new-from-rev` 增量门禁先止血，存量分批清理，不允许新增；不要一次性大改，风险高且 review 困难。
- **Q：golangci-lint 太慢？** A：开缓存（默认开）、`--concurrency`、CI 只跑增量、拆分 job；必要时只对变更包跑。
- **Q：`//nolint` 用多了怎么办？** A：`nolintlint` 强制带原因和 linter 名，review 时把豁免当技术债追踪，定期收敛。
- **Q：lint 和测试怎么分工？** A：lint 抓静态可判定的问题（流程、风格、明显 bug），测试抓运行时行为；两者互补，都不能替代 code review。
