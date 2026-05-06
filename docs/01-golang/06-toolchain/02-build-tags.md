# Build Tags 与条件编译

> 考察频率：★★★☆☆  优先级：P1

## 面试官考察意图

这道题考察的是候选人对 Go 编译控制的能力。5~8 年工程师在生产中一定会遇到"只在某环境编译某段代码"的场景——集成测试隔离、功能开关、平台差异化实现。说不清楚 build tag 语法和规则的，通常实践经验偏少。

## 核心答案（30 秒版）

Build Tags 控制在**文件级别**是否编译某段代码。旧语法是 `// +build`，新语法是 `//go:build`（Go 1.17+ 推荐）。Tag 可以组合：`linux && amd64` 表示同时满足 Linux + AMD64。文件名也可以带 OS/arch 后缀自动生效（`_linux.go`）。常用于集成测试隔离、功能开关、平台差异化实现。

## 深度展开

### 1. Build Tags 语法

**旧语法（Go 1.17 之前）**：
```go
// +build linux
// +build 386
```

**新语法（Go 1.17+，推荐）**：
```go
//go:build linux
//go:build 386
```

**语法规则**：
- `//go:build` 必须在 `package` 声明**上方**（紧邻，中间不能有空行）
- `// +build` 必须用**空行**与代码分隔（旧语法要求）
- 新旧语法**可以混用**（Go 会自动转换），但新语法更简洁
- 布尔表达式：`&&`（与）、`||`（或）、`!`（非）

```go
// 满足 Linux + AMD64
//go:build linux && amd64

// 满足 Linux 或 Darwin
//go:build linux || darwin

// 满足 非 Windows（即除 Windows 外的所有平台）
//go:build !windows
```

**文件名约定（自动生效）**：
| 文件名 | 等效 Tag |
|--------|---------|
| `example_linux.go` | `//go:build linux` |
| `example_darwin.go` | `//go:build darwin` |
| `example_amd64.go` | `//go:build amd64` |
| `example_arm64.go` | `//go:build arm64` |
| `example_test.go` | 普通测试文件（无需 tag） |
| `example_linux_test.go` | `//go:build linux` + 测试文件 |

文件名和文件内 tag 可以叠加，**同时满足**才编译。

### 2. 常用内置 Tag

| Tag 类型 | 常见值 | 说明 |
|---------|--------|------|
| OS | `linux`, `darwin`, `windows`, `freebsd` | 操作系统 |
| Arch | `amd64`, `arm64`, `386`, `wasm` | CPU 架构 |
| Go 版本 | `go1.21`, `go1.22` | 编译器版本（>= 该版本才编译） |
| CGO | `cgo`, `!cgo` | 是否启用 CGO |
| 调试 | `debug`, `testing` | 自定义 tag |

**Go 版本 tag 示例**（某特性仅 Go 1.21+ 可用）：
```go
//go:build go1.21
package newer
```

**cgo 差异化实现**：
```go
// file: memcheck_cgo.go
//go:build cgo
package memcheck

func CheckMemory(ptr uintptr, size int) { /* C 版本 */ }
```
```go
// file: memcheck_nocgo.go
//go:build !cgo
package memcheck

func CheckMemory(ptr uintptr, size int) { /* pure Go fallback */ }
```

### 3. 自定义 Tag 实战

**集成测试隔离**：
```go
// file: integration_test.go
//go:build integration

package mypkg

import (
    "testing"
    "github.com/testcontainers/testcontainers-go"
)

func TestWithDatabase(t *testing.T) {
    // 需要 Docker 的集成测试，CI 中通过 -tags=integration 运行
}
```
```bash
# 单元测试（默认）
go test ./...

# 集成测试（需要 Docker）
go test -tags=integration ./...
```

**功能开关（Pro / Community 版）**：
```go
// file: features_pro.go
//go:build pro

package billing

func GetAdvancedReports() []Report { /* Pro 功能 */ }
```
```go
// file: features_community.go
//go:build !pro

package billing

func GetAdvancedReports() []Report {
    panic("advanced reports require Pro edition")
}
```
```bash
# 构建 Pro 版
go build -tags=pro -o billing-pro .

# 构建社区版
go build -tags='' -o billing-community .
```

**调试代码**：
```go
// file: debug.go
//go:build debug

package main

var DebugMode = true
```
在 `debug` tag 开启时，某些生产代码会打印更详细日志或启用额外的断言。

**完整 Go 代码示例**——多平台日志实现：
```go
// file: logger_default.go
//go:build !windows

package log

import "os"

func GetFileDescriptor() *os.File {
    return os.Stdout
}
```
```go
// file: logger_windows.go
//go:build windows

package log

import (
    "golang.org/x/sys/windows"
)

func GetFileDescriptor() *os.File {
    return os.Stderr  // Windows 下 stderr 非 TTY，直接写文件
}
```

### 4. 与 embed 联动

不同环境嵌入不同配置文件：
```go
// file: config_dev.go
//go:build dev

package config

import _ "embed"

//go:embed config.yaml
var ConfigDev []byte
```
```go
// file: config_prod.go
//go:build !dev

package config

import _ "embed"

//go:embed config-prod.yaml
var ConfigProd []byte
```
```bash
go build -tags=dev -o app-dev .
go build -tags='' -o app-prod .
```

## 高频追问

### Q: `// +build` 和 `//go:build` 能混用吗？推荐哪个？
> 可以混用（Go 兼容两种）。但**推荐只用 `//go:build`**：语义更清晰（用布尔表达式而非空格分隔），且 `go vet` 会强制检查格式。

### Q: 如何只在 Linux 下编译某段代码，其他平台用默认实现？
> 两种方式：① 文件名加 `_linux.go` 后缀（自动加 tag）；② 在文件内写 `//go:build linux`。如果 Linux 有特殊实现，其他平台用通用实现，只需确保**非 Linux 文件**没有该函数的重复定义即可。

### Q: 集成测试文件怎么隔离？
> 在集成测试文件名加 `_integration_test.go`，内容写 `//go:build integration`，运行 `go test -tags=integration ./...`。这样普通 `go test ./...` 不会编译这些文件，避免依赖 Docker 等外部资源。

### Q: build tag 和 `if build.CGO_ENABLED` 有什么区别？
> Build tag 是**编译期**决定（不满足 tag 的文件不参与编译），`build.CGO_ENABLED` 是**运行时**检查。功能开关用 build tag（不同版本编译出不同二进制）；平台差异化用文件名约定或 `GOOS`/`GOARCH` 环境变量。

## 延伸阅读

- [Build Constraints (go.dev)](https://pkg.go.dev/go/build#hdr-Build_Constraints)
- [Go 1.17 Release Notes - Build constraint changes](https://go.dev/doc/go1.17#build)
- [go build -tags](https://pkg.go.dev/cmd/go#hdr-Compile_packages_and_dependencies)
