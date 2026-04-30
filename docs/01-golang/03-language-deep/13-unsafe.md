# unsafe 包：原理、使用场景与风险

> 考察频率：★★★★☆  优先级：P0

## TODO（待填写）

## 1. unsafe 的本质
- [ ] `unsafe.Pointer` 是什么（绕过 Go 类型系统的逃生通道）
- [ ] `unsafe.Sizeof` / `unsafe.Alignof` / `unsafe.Offsetof` 的用途
- [ ] `uintptr` vs `unsafe.Pointer` 的核心区别（uintptr 是整数，GC 不追踪）

## 2. unsafe.Pointer 四条合法转换规则（Go spec 原文）
- [ ] 规则一：任意类型指针 → `unsafe.Pointer`
- [ ] 规则二：`unsafe.Pointer` → 任意类型指针
- [ ] 规则三：`unsafe.Pointer` → `uintptr`（仅用于算术，不能存储）
- [ ] 规则四：`uintptr` → `unsafe.Pointer`（必须在同一表达式内，防止 GC 移动对象）
- [ ] 经典陷阱：`uintptr` 存入变量后再转回指针为什么不安全

## 3. 高性能技巧（面试加分点）
- [ ] 字符串与 `[]byte` 零拷贝互转（`reflect.StringHeader` + `unsafe`）
- [ ] Go 代码示例：比 `[]byte(str)` 快 10 倍的零拷贝转换
- [ ] 直接读取结构体私有字段（reflect + unsafe 组合）
- [ ] 内存对齐优化：用 `unsafe.Offsetof` 手动计算字段偏移

## 4. 什么时候才能用 unsafe
- [ ] 必须用的场景：实现 `sync.Pool`、`atomic.Value`、`reflect` 包底层
- [ ] 不该用的场景：普通业务代码（维护成本极高，Go 版本升级可能 break）
- [ ] 最小化原则：隔离在独立函数/包中，加注释说明

## 5. 与 reflect 的关系
- [ ] reflect 内部大量使用 unsafe（`reflect.Value` 底层就是 `unsafe.Pointer`）
- [ ] 为什么 reflect 操作慢（类型检查开销 + 指针间接跳转）

## 高频追问
- [ ] `uintptr` 和 `unsafe.Pointer` 能混用吗？什么时候会出问题？
- [ ] 为什么 Go 字符串转 `[]byte` 会发生内存拷贝？用 unsafe 怎么避免？
- [ ] reflect 和 unsafe 哪个性能更差？为什么？
