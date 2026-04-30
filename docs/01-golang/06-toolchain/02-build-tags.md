# Build Tags 与条件编译

> 考察频率：★★★☆☆  优先级：P1

## TODO（待填写）

## 1. Build Tags 语法
- [ ] 旧语法：`// +build linux` vs 新语法：`//go:build linux`（Go 1.17+）
- [ ] 布尔表达式：`//go:build linux && amd64`
- [ ] 文件名约定：`_linux.go` / `_amd64.go` 自动生效

## 2. 常用内置 Tag
- [ ] OS：`linux` / `darwin` / `windows`
- [ ] 架构：`amd64` / `arm64` / `386`
- [ ] Go 版本：`go1.21`（Go 1.21+ 支持）
- [ ] `cgo` / `!cgo`：控制 CGO 相关代码

## 3. 自定义 Tag 实战
- [ ] 集成测试隔离：`//go:build integration`，运行时 `go test -tags=integration`
- [ ] 功能开关：`//go:build pro` 区分社区版/专业版
- [ ] 调试代码：`//go:build debug` 只在开发环境编译
- [ ] 完整 Go 代码示例

## 4. 与 embed 联动
- [ ] 不同环境嵌入不同配置文件
- [ ] `//go:embed` 与 build tag 组合使用

## 高频追问
- [ ] `// +build` 和 `//go:build` 能混用吗？推荐哪个？
- [ ] 如何只在 Linux 下编译某段代码，其他平台用默认实现？
- [ ] CI 中如何跑集成测试但不跑单元测试？
