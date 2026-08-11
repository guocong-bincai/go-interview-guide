[🏠 首页](../../../README.md) · [📦 Go 语言深度](../../README.md) · [💬 语言机制](../README.md)

---

# Interface Composition vs Embedding：组合与嵌入的区别

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：接口组合、类型嵌入、method set、隐式实现、依赖注入、鸭子类型

## 🎯 面试官考察意图

Go 中没有继承，但有"组合优于继承"的设计哲学。面试官想确认：

1. 能否区分接口组合（interface composition）和结构体嵌入（type embedding）
2. 理解 method set 规则：什么情况下一个类型自动实现了某个接口
3. 是否知道值接收者 vs 指针接收者对接口实现的影响
4. 能否用接口组合设计可测试、可扩展的系统架构

---

## ⚡ 核心答案（30秒版）

**接口组合 = 多个接口的字段合并为一个新接口；结构体嵌入 = 内嵌类型的字段和方法直接提升为外层类型的方法。**

核心区别：**接口组合产生的是"更大"的接口签名，结构体嵌入产生的是"更扁平"的方法集。**

面试高频陷阱：使用值接收者的方法无法满足 pointer type 的接口约束，反之亦然。如果 A 用值接收者实现了 X 接口，那么 `*A` 也能满足 X 接口，但 A 不能直接满足需要指针方法的接口。

---

## 🔬 深度展开

### 1. 接口组合（Interface Composition）

```go
// 基础接口
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// ✅ 接口组合：将两个接口合并为一个
type ReadWriter interface {
    Reader  // 嵌入 Reader 接口，等价于展开 Write 和 Read
    Writer  // 嵌入 Writer 接口
}

// ReadWriter 等价于：
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```

实际应用场景——**依赖注入**：

```go
// 定义细粒度接口（最小化依赖）
type UserStore interface {
    GetUser(id int) (*User, error)
    CreateUser(u *User) error
}

type CacheService interface {
    Get(key string) (string, error)
    Set(key, value string, ttl time.Duration) error
}

// 组合成更大的服务接口
type UserService interface {
    UserStore
    CacheService
    
    Login(username, password string) (*Token, error)
}
```

这样设计的好处：
- 接口越小越容易 mock（测试友好）
- UserService 可以接受任意实现了 UserStore + CacheService 的对象
- 符合**面向接口编程**原则

### 2. 结构体嵌入（Type Embedding）

```go
// 基础日志器
type Logger interface {
    Info(msg string)
    Error(msg string)
}

type fileLogger struct {
    path string
}

func (l *fileLogger) Info(msg string)  { fmt.Printf("[INFO] %s\n", msg) }
func (l *fileLogger) Error(msg string) { fmt.Printf("[ERROR] %s\n", msg) }

// ✅ 结构体嵌入：让 HTTPServer 拥有 Logger 的能力
type HTTPServer struct {
    addr     string
    logger   Logger    // 嵌入接口 —— 这不是 "is-a"，而是 "has-a"
    handler  http.Handler
}

func (s *HTTPServer) Start() {
    s.logger.Info("Starting server")  // 直接使用嵌入类型的方法
    http.ListenAndServe(s.addr, s.handler)
}

// ❌ 常见错误：嵌入具体类型（耦合度太高）
type HTTPServerBad struct {
    logger *fileLogger  // 💥 硬编码了实现，无法替换
}
```

### 3. Method Set 规则（面试必考！）

```go
type MyStruct struct{}

func (m MyStruct) ValueMethod()    {} // 只有 MyStruct 能调用
func (m *MyStruct) PointerMethod() {} // 只有 *MyStruct 能调用

var v MyStruct
v.ValueMethod()     // ✅ 值 receiver 的方法
// v.PointerMethod() // ❌ 编译错误

var p *MyStruct
p.ValueMethod()     // ✅ Go 会自动解引用
p.PointerMethod()   // ✅ 指针 receiver 的方法

// 关键：接口满足关系
var i1 ValueOnly = v      // ✅ MyStruct 实现了 ValueOnly
var i2 PointerOnly = &v   // ✅ *MyStruct 实现了 PointerOnly
// var i3 Both = v         // ❌ MyStruct 没有 PointerOnly 方法
```

**黄金法则**：
> 如果方法用 **值接收者** 声明，则 **值类型** 和 **指针类型** 都能实现该接口。
> 如果方法用 **指针接收者** 声明，则只有 **指针类型** 能实现该接口。

### 4. 实战：接口嵌套设计的反模式

#### ❌ 反模式1：接口膨胀（God Interface）

```go
// ❌ 什么都能做的大接口
type Service interface {
    Process(ctx context.Context) error
    Validate(input []byte) error
    Serialize(data interface{}) ([]byte, error)
    Deserialize(data []byte) (interface{}, error)
    Log(msg string)
    Metric(name string, value float64)
    Shutdown() error
}
```

应该拆分为职责单一的接口并组合：

```go
// ✅ 拆分后组合
type Processor interface { Process(ctx context.Context) error }
type Validator interface { Validate(input []byte) error }
type Serializer interface { 
    Serialize(data interface{}) ([]byte, error) 
    Deserialize([]byte) (interface{}, error) 
}
type Logger interface { Log(msg string) }

type FullService interface {
    Processor
    Validator
    Serializer
    Logger
    Shutdown() error
}
```

#### ❌ 反模式2：在公共接口中暴露实现细节

```go
// ❌ 暴露了内部 buffer 细节
type DataSource interface {
    GetData() []byte           // 返回具体实现的缓冲区
    SetBufferSize(size int)    // 修改内部实现细节
}

// ✅ 抽象数据流
type DataSource interface {
    Read() (data io.Reader, err error)  // 通过标准库接口传递
}
```

### 5. 依赖注入的最佳实践

```go
// 业务层定义细粒度接口
type UserRepository interface {
    FindByID(id int) (*User, error)
    Save(user *User) error
}

type EmailService interface {
    Send(to string, subject, body string) error
}

type PaymentGateway interface {
    Charge(amount float64, currency string, card Token) error
}

// 应用层组合需要的能力
type OrderService struct {
    users    UserRepository
    emails   EmailService
    payments PaymentGateway
}

func NewOrderService(
    u UserRepository,
    e EmailService,
    p PaymentGateway,
) *OrderService {
    return &OrderService{users: u, emails: e, payments: p}
}

// 测试时轻松 mock
func TestOrderService(t *testing.T) {
    mockUsers := &MockUserRepository{}
    mockEmails := &MockEmailService{}
    mockPay := &MockPaymentGateway{}
    
    svc := NewOrderService(mockUsers, mockEmails, mockPay)
    // ... 正常测试
}
```

---

## 🗣️ 面试话术

- **初级**："接口组合就是把多个小接口合并成大接口，结构体嵌入就是让一个类型拥有另一个类型的字段和方法。"
- **中级**："关键是 method set 规则：值 receiver 的方法值类型和指针类型都有，指针 receiver 的方法只有指针类型有。接口越小越好 mock。"
- **高级**："Go 的接口是 implicit duck typing，不需要显式声明 implements。最佳实践是让接口定义在客户端（使用者），而不是提供者（实现者）。嵌入接口用于组合，嵌入结构体用于扩展功能。依赖注入要确保接口足够细粒度，每个接口只表达一个抽象。"

---

## 🔗 关联阅读

- [Interface 内存布局](./03-01-interface.md)
- [Reflect 原理](./47-02-reflect.md)
- [sync.Pool 最佳实践](../02-concurrency/80-pool-best-practices.md)
