[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [🛠️ 工程工具链](../README.md)

---

# 原生 Fuzzing（testing.F）：覆盖率引导的模糊测试实战

> 考察频率：★★★☆☆  优先级：P1（工程化加分项）
> 关键词：go fuzz、testing.F、种子语料、crasher、覆盖率引导、oss-fuzz

---

## 面试官考察意图

考察候选人**是否用过 Go 内置模糊测试**，以及能否把它接进 CI。
初级只知道 `go test`；高级要能讲清楚 **`FuzzXxx` 的写法、种子语料与生成语料的区别、crash 如何固化为回归用例、以及 fuzz 的适用范围与成本控制**。

---

## 核心答案（30 秒版）

Go 1.18 起，模糊测试进入标准工具链：

```go
func FuzzParseQuery(f *testing.F) {
    // 1) 种子语料（seed corpus）：可复现的初始输入
    f.Add("name=go&age=1")
    f.Add("")

    // 2) fuzz target：参数只能是基本类型 / string / []byte
    f.Fuzz(func(t *testing.T, raw string) {
        q, err := ParseQuery(raw)
        if err != nil {
            return
        }
        if got := q.Encode(); got != raw { // 不变量：编解码应可逆
            t.Fatalf("round-trip failed: %q -> %q", raw, got)
        }
    })
}
```

```bash
go test -run=FuzzParseQuery            # 只跑种子语料（CI 回归）
go test -fuzz=FuzzParseQuery -fuzztime=30s   # 覆盖率引导的模糊测试
```

发现崩溃时，Go 会把**最小化后的输入写入 `testdata/fuzz/FuzzXxx/`**，
之后普通 `go test` 就会复现它——这就是"crash → 永久回归用例"的闭环。

---

## 深度展开

### 1. 编写规范

```go
func FuzzDecode(f *testing.F) {
    f.Add([]byte(`{"a":1}`), int64(1))   // 多种类型混合的种子
    f.Fuzz(func(t *testing.T, data []byte, n int64) {
        _ = decode(data, n)              // 只要不 panic 就算通过
    })
}
```

规则：

| 项 | 约束 |
|----|------|
| 函数名 | 必须以 `Fuzz` 开头，参数是 `*testing.F` |
| `f.Fuzz` 回调参数类型 | `string`、`[]byte`、`bool`、各种 `int`/`uint`/`float` 及其类型 |
| 一个文件/包内 | 可以有多个 `FuzzXxx`，但**一次 `-fuzz` 只能跑一个** |
| 与单测共存 | 同一文件可以同时有 `TestXxx` / `BenchmarkXxx` / `FuzzXxx` |
| 断言 | 只能用 `t.Fatal`/`t.Fatalf` 判定"发现问题"，正常路径必须 return |

### 2. 覆盖率引导的工作方式

```
种子语料 ──→ 执行 ──→ 记录覆盖率 ──→ 变异（mutate）──→ 更有趣的输入入队
   ↑                                                        │
   └────────────────── 循环（-fuzztime 控制）────────────────┘
```

- 变异策略：位翻转、字节替换、插入/删除、跨输入拼接；
- **"有趣"的定义**：触发新代码路径（新覆盖率）的输入会被保留进语料；
- 语料缓存目录：`$GOCACHE/fuzz/<module>/<FuzzName>/`；
- **首次发现新 crasher 会写入 `testdata/fuzz/FuzzXxx/xxx`**，通常还会做一遍最小化（minimize）。

### 3. 与其它测试手段的分工

| 手段 | 输入来源 | 擅长发现 |
|------|---------|---------|
| `TestXxx` | 手写用例 | 已知行为、回归 |
| `BenchmarkXxx` | 固定输入 | 性能回退 |
| **Fuzzing** | 变异生成 | panic、越界、死循环、解析器边界、不变量破坏 |
| `-race` | 任意测试 | 数据竞争 |
| `go vet` / staticcheck | 静态分析 | 代码模式问题 |

**组合拳**：`go test -race -fuzz=FuzzXxx` 可以在模糊测试同时检测竞争。

### 4. 接进 CI 的正确姿势

```yaml
# CI：短时间 fuzz + 跑全部已归档 crasher（种子语料）
- run: go test ./...                          # 会跑 testdata/fuzz 下的所有 crasher
- run: go test -run=^$ -fuzz=FuzzParse -fuzztime=60s ./internal/parse
```

- **PR 里跑 30~60 秒**，夜间任务跑更久（或接 oss-fuzz 持续跑）；
- **`testdata/fuzz/` 必须提交进仓库**：它是安全网，保证历史 crash 不复发；
- **失败即阻断**：一旦 fuzz 出 panic，CI 必须红，不能只记日志。

### 5. 生产坑点

```go
// ❌ 坑 1：fuzz target 里做真实网络/文件 IO → 慢且不确定
f.Fuzz(func(t *testing.T, url string) {
    http.Get(url) // 绝对不要
})

// ❌ 坑 2：把"业务语义"当断言，导致大量误报
f.Fuzz(func(t *testing.T, s string) {
    if s != strings.ToLower(s) { t.Fatal("want lowercase") } // 输入本来就可能是大写
})

// ✅ 正确：断言"不变量"和"不该 panic"
f.Fuzz(func(t *testing.T, data []byte) {
    out, err := Parse(data)
    if err == nil && !validate(out) {
        t.Fatalf("invalid output for %q", data)
    }
})
```

- **不变量（invariant）才是好断言**：如"解析→序列化→再解析结果一致"、"输出必须满足 schema"；
- **避免非确定性**：不要依赖当前时间、map 遍历顺序、随机数种子未固定的 `math/rand`；
- **控制资源**：解析器类 fuzz 目标要能容忍超长输入（加长度上限，否则可能 OOM 或超时）；
- **不要 fuzz "会调用外部服务"的函数**：先抽出纯函数层再 fuzz。

---

## 高频追问

**Q：fuzz 发现的 crasher 为什么不直接修，还要提交进 `testdata/fuzz/`？**

因为**输入本身是最小复现用例**。提交后，任何人跑 `go test` 都会重新验证它，
避免"修好又改回去"的回归。这也是 fuzzing 与一次性脚本的最大区别。

**Q：`testing.F` 和 `testing.T` 的关系？**

`FuzzXxx(f *testing.F)` 里的 `f.Add` 用于添加种子，`f.Fuzz` 的回调里拿到的是 `*testing.T`，
断言写法与普通单测一致。`-run` 会执行种子语料（相当于跑单测）。

**Q：fuzz 目标参数支持 struct 吗？**

不支持。参数只能是基本类型、`string`、`[]byte`（以及它们的切片）。
复杂结构要靠 `[]byte` 反序列化或在参数里拆成多个基本类型。

**Q：和 `testing/quick` 有什么区别？**

`testing/quick` 是**基于属性的随机测试**，不看覆盖率、不自动最小化、不生成持久语料；
`testing.F` 是覆盖率引导的模糊测试，能持续探索深层分支。新代码应优先用 `testing.F`。

---

## 面试话术

> "Go 1.18 之后内置了 fuzzing：写 `FuzzXxx(f *testing.F)`，`f.Add` 放种子，
> `f.Fuzz` 里对基本类型做断言，跑 `go test -fuzz=... -fuzztime=30s` 就会做覆盖率引导的变异。
> 关键工程价值是它会把最小化后的 crasher 写进 `testdata/fuzz/`，
> 普通 `go test` 就能复现，等于给历史 bug 建了永久回归网。
> 断言要用不变量，别依赖时间和随机，PR 里跑几十秒就够。"

---

## 延伸阅读

- [Go Fuzzing（官方文档）](https://go.dev/doc/fuzz/)
- [Go 1.18 Release Notes — Fuzzing](https://go.dev/doc/go1.18#fuzzing)
- [testing.F 文档](https://pkg.go.dev/testing#F)

---

**[← 上一篇：GOTOOLCHAIN](./77-gotoolchain.md)** · **[返回模块首页 →](../README.md)**
