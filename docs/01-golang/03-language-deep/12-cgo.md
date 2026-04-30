# CGO 原理与面试考点

> 考察频率：★★★★☆  优先级：P0

## TODO（待填写）

## 1. CGO 是什么
- [ ] CGO 让 Go 调用 C 代码的原理（`import "C"` 的魔法）
- [ ] CGO 调用链路：Go → runtime → OS thread → C 函数
- [ ] `CGO_ENABLED=0` 静态编译的意义（Docker 镜像瘦身、交叉编译）

## 2. CGO 调用开销（必考数据）
- [ ] Go 函数调用 ~1ns，CGO 调用 ~100ns，差距 100 倍的根因
- [ ] 为什么 CGO 调用必须切换到系统线程（`runtime.LockOSThread`）
- [ ] CGO 调用期间 GC 如何处理（C 栈不受 GC 扫描）
- [ ] 大量 CGO 调用时 M 线程数量为什么暴涨

## 3. CGO 内存管理规则
- [ ] Go GC 不管 C 内存（`C.malloc` / `C.free` 需手动管理）
- [ ] C 代码不能持有 Go 指针（Go 对象可能被 GC 移动）
- [ ] `C.CString` / `C.GoString` 的内存泄漏陷阱
- [ ] 完整 Go 代码示例：正确传递字符串给 C

## 4. 什么时候用 CGO，什么时候不用
- [ ] 适合：调用 C 库（OpenSSL/SQLite/ffmpeg）、性能极端敏感的数学运算
- [ ] 不适合：高并发场景（线程数暴涨）、需要跨平台编译
- [ ] 替代方案：纯 Go 重写、gRPC 进程间通信

## 5. 生产踩坑
- [ ] CGO 导致 goroutine 调度延迟（M 线程被 C 占用）
- [ ] C 库内存泄漏如何用 valgrind + pprof 联合排查

## 高频追问
- [ ] 为什么 Go 官方建议尽量避免 CGO？
- [ ] CGO 调用为什么不能直接在 goroutine 里执行，而要 LockOSThread？
- [ ] 如何测量 CGO 的性能开销（benchmark 写法）？
