# Go Module 与 Workspace 工程实践

> 考察频率：★★★☆☆  优先级：P1

## 面试官考察意图

这道题考察的是 Go 工程化能力的底线——5~8 年的工程师如果连 module 路径怎么写、replace 和 exclude 什么区别都说不清，基本可以判断项目经验偏业务层。面试官通常用这道题开场，试探候选人对 Go 工具链的掌握深度。

## 核心答案（30 秒版）

Go Module 是 Go 1.11 引入的依赖管理方案，核心文件是 `go.mod`，记录模块路径和依赖版本。**MVS（Minimum Version Selection）** 是版本选择算法，选取满足 semver 约束的最小版本，而非最新版本。私有模块通过 `GOPRIVATE`/`GONOSUMDB` 绕过 checksum 校验，通过 `GOPROXY` 设置代理。`go.work` 是多模块联调工具，解决本地多模块调试时反复用 replace 的痛点。

## 深度展开

### 1. go.mod 核心语义

```go
module github.com/my/project

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/redis/go-redis/v9 v9.3.0
)

require github.com/pkg/errors v0.9.1 // indirect

replace github.com/original/pkg => github.com/fork/pkg v1.2.3

exclude github.com/bad/pkg v0.1.0
```

- **`module`**：声明模块路径，同时也是 package 的 import 路径前缀。路径格式通常为 `github.com/org/repo` 或 `github.com/org/repo/v2`（v2+ 必须加 `/vN` 后缀）
- **`go`**：声明最低 Go 版本，影响 language feature 和依赖的最低版本要求
- **`require`**：直接依赖，格式 `packagepath version`，版本必须以 `v` 开头（semver）
- **`replace`**：替换来源，可用于：
  1. **本地调试**：`replace github.com/foo => ../foo`（本地 fork 修改）
  2. **fork 替换**：`replace github.com/original => github.com/my-fork v1.0.0`
  3. **间接依赖 patch**：生产中遇到 bug，在不升级大版本的情况下 patch 某个间接依赖
- **`exclude`**：排除特定版本，常用于修复 CVE 后临时规避（临时方案，最终还是要升级）
- **间接依赖（indirect）**：`go mod tidy` 自动推导，写明是因为被其他依赖引用而非直接导入

### 2. 版本选择算法（MVS）

Go 使用 **MVS（Minimum Version Selection）** 而非 npm 的"最高满足 semver"策略。这是 Go 模块系统最反直觉的地方。

**原理**：假设 A 依赖 B 和 C，B 需要 `pkg v1.3.0`，C 需要 `pkg v1.5.0`，则最终选择 `v1.5.0`（取最大满足版本）。

**为什么这样设计**：
- **可重复构建**：给定同一套 `go.mod`，任何人任何时间构建，结果一致（确定性）
- **单向版本升级**：升级只发生在你明确 `go get` 时，不会因为间接依赖的变动导致你的依赖悄然升级
- **对比 npm**：npm 使用最高满足版本，但 A 的 `^1.3.0` 和 B 的 `^1.5.0` 可能被解析成不同版本，导致"我的机器上能跑"问题

```bash
# 实际验证 MVS 行为
go mod graph | grep <package>  # 画出依赖图
go mod why <package>           # 解释为什么要这个包
```

**`go mod tidy`**：
1. 扫描所有 `.go` 文件的 import 语句
2. 移除未使用的依赖（`require` 中有但代码没引用）
3. 补全间接依赖（`go.sum` 中有记录但 `go.mod` 中无 `// indirect` 标记）
4. 更新 `go.sum` checksum

### 3. 私有模块与代理配置

**GOPROXY**：控制依赖下载来源和顺序
```bash
# 国内常用配置
export GOPROXY=https://goproxy.cn,direct
# 解释：优先从 goproxy.cn 拉取；拉不到则直连源站

# 七牛云
export GOPROXY=https://goproxy.cn,direct

# 官方（国外）
export GOPROXY=https://proxy.golang.org,direct
```

**GOPRIVATE / GONOSUMDB**：跳过私有模块的 checksum 校验
```bash
# 等价写法
export GOPRIVATE=*.internal.company.com,github.com/company/*
export GONOSUMDB=github.com/company/*

# GONOSUMDB 配置哪些 path 不做 checksum 校验
# GONOSUMDB=* 跳过所有（不推荐）
```

**私有 Go Module 代理**：
- **Athens**：开源私有代理，支持 PostgreSQL 持久化
- **Nexus**：Sonatype Nexus 3.x 内置 Go 代理
- **goproxy.cn**：七牛维护的国内代理，`GOPROXY=https://goproxy.cn,direct`

**实际生产配置**：
```bash
# ~/.netrc 或 CI 环境变量
export GOPROXY=https://goproxy.cn,direct
export GOPRIVATE=git.internal.company.com
export GONOSUMDB=git.internal.company.com
export GIT_TERMINAL_PROMPT=0  # 禁止 git 交互式输入密码
```

### 4. Go Workspace（go.work）

**解决的问题**：多模块项目联调时，如果 A 模块依赖 B 模块，每次改 B 都要 `replace` → `go mod tidy` → revert，很痛苦。

```bash
# 初始化 workspace
go work init ./module-a ./module-b ./module-c

# 生成的 go.work
go 1.21

use (
    ./module-a
    ./module-b
    ./module-c
)
```

**`go work` 命令**：
- `go work init [dir...]`：创建 workspace
- `go work use ./new-module`：追加模块
- `go work sync`：将 workspace 依赖同步到各模块
- `go work edit`：修改 go.work（很少用）

**重要规则**：
- `go.work` **不应该提交到版本库**（`.gitignore` 中添加 `go.work`）
- 仅本地开发使用，生产构建走正常的 module proxy
- workspace 内模块的 `go.mod` 中的 `replace` 指令在 `go work` 模式下**被忽略**

### 5. 常见工程问题

**`go mod vendor`**：
```bash
go mod vendor  # 将所有依赖复制到 vendor/ 目录
go build -mod=vendor  # 使用 vendor 构建，CI 常用
```
使用场景：
- 离线构建（内网 CI 无法访问外网）
- 锁定依赖确保 CI 每次用完全相同的代码
- Docker 构建时避免网络抖动

**依赖版本冲突排查**：
```bash
go mod graph | grep redis  # 看 redis 相关依赖链
go mod why github.com/xxx  # 解释某个包为什么被引入
go list -m all             # 列出所有依赖及版本
go get github.com/foo@v1.2.3  # 升级到指定版本
go get github.com/foo@latest   # 升级到最新
```

**`go.sum` 能删吗**？
- **不能删**。`go.sum` 记录每个依赖的 crypto checksum，防止依赖被篡改
- 删了 `go.sum`，CI 第一次跑会重新计算并写入，本地可能会 checksum mismatch
- `GOSUMDB` 控制 checksum 数据库，设为 `off` 可跳过（不推荐生产环境）

### 6. Go 1.24 tool 指令：规范化工具依赖

> Go 1.24（2025 年 2 月发布）引入 `tool` 指令，解决「开发工具依赖如何管理」的长期痛点。

#### 6.1 痛点：以前怎么管工具依赖？

在 Go 1.24 之前，管理可执行工具（如 mockery、staticcheck、swag）有三种方式，都是 workaround：

| 方式 | 做法 | 问题 |
|------|------|------|
| **空导入** | 创建 `tools.go`，`import _ "github.com/xxx/mockery"` | 工具混入主依赖，版本不同步 |
| **手动安装** | `brew install mockery` 或下载二进制 | 团队版本不一致，CI 无法复现 |
| **Makefile** | 在 Makefile 里写 `go install xxx` | 工具依赖不在代码中，CI 隐性依赖 |

#### 6.2 Go 1.24 tool 指令

Go 1.24 允许在 `go.mod` 中直接声明工具依赖：

```go
module myproject

go 1.24

tool github.com/vektra/mockery/v2@v2.52.1
tool honnef.co/go/tools/cmd/staticcheck@latest
```

安装工具的方式：
```bash
# 通过 gotip install tool 安装（Go 1.24+）
gotip install tool github.com/vektra/mockery/v2@v2.52.1

# 运行工具
gotip tool github.com/vektra/mockery/v2 --all --output ./mocks

# 等价于
go install github.com/vektra/mockery/v2@v2.52.1@tool
```

**`go tool`** 子命令（Go 1.24 新增）：
```bash
go tool <package>@<version>  # 运行已声明的工具
go tool staticcheck@latest   # 运行 staticcheck
go tool --help               # 查看帮助
```

#### 6.3 可执行文件缓存优化

Go 1.24 之前，通过 `go run` 或 `go tool` 执行的可执行文件**不被缓存**，每次都重新编译。Go 1.24 开始：
- 可执行文件会被缓存到 Go 构建缓存（`~/.cache/go-build`）
- **包文件**缓存清理周期：**5 天**
- **可执行文件**缓存清理周期：**2 天**

这显著加速了 CI 中频繁调用 `go run` 的场景（如代码生成、协议编译）。

#### 6.4 对比：tool 指令 vs 空导入

| 维度 | 空导入（旧） | tool 指令（Go 1.24+） |
|------|------------|---------------------|
| 声明位置 | 代码文件 `tools.go` | `go.mod`（与业务依赖分离） |
| `go list -m all` 可见性 | 混入主依赖 | 显式标记为工具依赖 |
| 版本控制 | 与主模块版本耦合 | 独立版本，互不影响 |
| CI 复现性 | 依赖 Makefile/脚本 | go.mod 中声明，天然可复现 |
| 语义清晰度 | 模糊（间接依赖混入） | 清晰（工具就是工具） |

#### 6.5 实际应用场景

**场景 1：团队统一 mockery 版本**
```go
// go.mod
tool github.com/vektra/mockery/v2@v2.52.1
```
- 新成员 `gotip install tool github.com/vektra/mockery/v2` 即可安装正确版本
- CI 不再依赖手动 `go install`，构建结果可复现

**场景 2：CI 中使用代码生成工具**
```yaml
# .github/workflows/ci.yml
- name: Generate mocks
  run: gotip tool github.com/vektra/mockery/v2 --all --output ./mocks
```

**场景 3：统一管理 lint 工具链**
```go
// go.mod
tool honnef.co/go/tools/cmd/staticcheck@latest
tool github.com/golangci/golangci-lint/cmd/golangci-lint@latest
```

## 高频追问

### Q: `go.sum` 文件的作用是什么，能删掉吗？
> 不能删。`go.sum` 记录每个依赖的 **crypto checksum**（go binary hash），用于校验下载的包是否被篡改。没有 `go.sum`，Go 会回退到校验 `go.mod` 中的版本号，安全性降低。

### Q: 同一个包两个版本依赖冲突怎么处理？
> 先分析冲突来源：`go mod graph | grep <pkg>` 找出哪些包依赖了不同版本。MVS 通常会选较高的满足版本。如果产生冲突（如两个 semver 不兼容），需要升级/降级上层依赖来消除。

### Q: `go mod tidy` 和 `go mod download` 的区别？
> `go mod download`：仅下载依赖到本地缓存（`$GOPATH/pkg/mod`），不修改 `go.mod`/`go.sum`。`go mod tidy`：分析代码后修改 `go.mod`，补全间接依赖、移除未使用依赖。

### Q: `go get -u ./...` 升级全部依赖有什么风险？
> 风险极大：会把所有直接依赖升级到各自 semver 约束下的最新版本，可能引入 breaking change（即使满足 `^v1.x.x`），导致线上故障。正确做法是 `go get -u github.com/specific/pkg` 逐个升级。

## 延伸阅读

- [Go Modules Reference](https://go.dev/ref/mod)
- [Using Go Modules](https://go.dev/blog/using-go-modules)
- [Go Module Proxy 官方文档](https://go.dev/blog/module-mirror)
- [Keeping Your Modules Compatible](https://go.dev/blog/module-compatibility)
