# Go Module 与 Workspace 工程实践

> 考察频率：★★★☆☆  优先级：P1

## TODO（待填写）

## 1. go.mod 核心语义
- [ ] `module` / `go` / `require` / `replace` / `exclude` 指令详解
- [ ] `replace` 三种场景：本地调试、fork 替换、间接依赖 patch
- [ ] `exclude` 用途（排除有漏洞的版本）

## 2. 版本选择算法（MVS）
- [ ] Minimum Version Selection：选最小满足需求的版本（不是最新）
- [ ] 为什么 MVS 比 npm 语义化版本更可预测
- [ ] `go mod tidy` 做了什么（清理未使用依赖 + 补全间接依赖）

## 3. 私有模块与代理配置
- [ ] `GOPROXY`：代理链（`GOPROXY=https://goproxy.cn,direct`）
- [ ] `GONOSUMDB` / `GOPRIVATE`：内网 GitLab 依赖绕过校验
- [ ] 搭建私有 Go Module 代理（Athens / Nexus）

## 4. Go Workspace（go.work）
- [ ] 解决什么问题：多模块联调不需要每次 replace
- [ ] `go work init` / `go work use` 命令
- [ ] Workspace 模式 vs replace 指令对比
- [ ] 生产环境不应提交 go.work（仅本地开发用）

## 5. 常见工程问题
- [ ] `go mod vendor` 的使用场景（离线构建/CI 稳定性）
- [ ] 依赖版本冲突排查：`go mod graph | grep pkg`
- [ ] 升级全部依赖：`go get -u ./...` 的风险

## 高频追问
- [ ] `go.sum` 文件的作用是什么，能删掉吗？
- [ ] 同一个包两个版本依赖冲突怎么处理？
- [ ] `go mod tidy` 和 `go mod download` 的区别？
