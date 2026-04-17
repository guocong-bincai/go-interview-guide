# Go 错误处理最佳实践

> 考察频率：★★★★☆  优先级：P1

## TODO（待填写）

- [ ] error 接口本质（interface{Error() string}）
- [ ] errors.Is / errors.As / errors.Unwrap 的区别和使用场景
- [ ] %w 包装错误 vs 直接 fmt.Errorf
- [ ] 自定义错误类型：什么时候用 struct，什么时候用 sentinel error
- [ ] panic/recover 的使用边界（什么时候用，什么时候不用）
- [ ] 生产最佳实践：在哪层处理错误，在哪层打日志
- [ ] 高频追问：如何在不改原始错误的情况下附加上下文信息？
