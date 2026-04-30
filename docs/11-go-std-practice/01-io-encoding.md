# Go 标准库生产实践：io / encoding / time / testing

> 考察频率：★★★☆☆  优先级：P1

## TODO（待填写）

## 1. io 包：Reader/Writer 组合模式
- [ ] `io.Reader` / `io.Writer` 接口本质（单方法接口，高度可组合）
- [ ] 常用组合：`io.TeeReader`（读时写副本）、`io.MultiWriter`（一写多处）
- [ ] `bufio.NewReader` / `bufio.NewWriter`：何时用缓冲（网络/文件 IO）
- [ ] `io.Pipe`：goroutine 之间的流式传输（生产者-消费者）
- [ ] 零拷贝：`io.WriterTo` 接口（`*os.File` 实现 sendfile 系统调用）
- [ ] 完整 Go 代码示例：流式处理大文件（不全量加载内存）

## 2. encoding/json：性能陷阱与替代方案
- [ ] 为什么标准库 json 慢：反射 + 接口转换开销
- [ ] `json.Number`：避免大整数精度丢失
- [ ] 自定义 `MarshalJSON` / `UnmarshalJSON`
- [ ] 性能对比（benchmark）：标准库 vs json-iterator vs bytedance/sonic
- [ ] sonic 的原理：JIT 编译 + SIMD 字符串扫描
- [ ] 选型建议：高性能场景用 sonic，普通场景标准库足够

## 3. time：高频踩坑
- [ ] `time.NewTimer` 泄漏：只用 `After` 但不停止 Timer
- [ ] `time.AfterFunc` 与 goroutine 泄漏
- [ ] Go 1.23 Timer/Ticker channel 变更（GC 可回收未停止的 Timer）
- [ ] 时区处理：`time.LoadLocation` vs `time.FixedZone`，数据库时区陷阱
- [ ] `time.Since` 精度：单调时钟 vs 挂钟（wall clock）

## 4. testing：最佳实践
- [ ] Table-Driven Test：一个函数测所有边界条件
- [ ] `TestMain`：全局 setup/teardown（数据库连接初始化）
- [ ] Subtests（`t.Run`）：独立命名 + 并行测试（`t.Parallel()`）
- [ ] Mock 最佳实践：接口隔离 + testify/mock vs gomock 对比
- [ ] Fuzz Testing（`go test -fuzz`）：自动生成边界输入
- [ ] 覆盖率：`go test -cover` + coverprofile 可视化

## 高频追问
- [ ] `bufio` 缓冲和直接 `io.Reader` 读取，性能差多少？
- [ ] Go 的 `encoding/json` 怎么处理循环引用？
- [ ] 为什么 `time.After` 在高频调用的循环里会导致内存泄漏？
