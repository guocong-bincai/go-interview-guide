# Go net/http 深度解析

> 考察频率：★★★★☆  优先级：P1

## TODO（待填写）

- [ ] http.Server 底层结构（连接生命周期、goroutine-per-connection 模型）
- [ ] Transport 连接池：MaxIdleConns、KeepAlive、连接复用原理
- [ ] Handler 接口与 ServeMux 路由匹配规则
- [ ] 中间件链式调用实现（http.Handler 包装模式）
- [ ] 超时配置：ReadTimeout / WriteTimeout / IdleTimeout 分别控制什么
- [ ] 优雅关闭：Shutdown vs Close 的区别
- [ ] 高频追问：如何实现一个简单的 HTTP 框架中间件？
