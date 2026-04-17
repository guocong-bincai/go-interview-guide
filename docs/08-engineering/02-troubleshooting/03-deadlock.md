# 死锁排查

> 考察频率：★★★★☆  难度：★★★★☆

## 什么是死锁

死锁：两个或多个操作相互等待对方释放资源，导致程序永久阻塞。

**死锁的四个必要条件（Coffman 条件）：**
1. **互斥**：资源每次只能被一个 goroutine 持有
2. **持有并等待**：goroutine 持有资源同时等待其他资源
3. **不可抢占**：资源不能被强制释放，只能主动释放
4. **循环等待**：存在一个 goroutine 循环等待链

**破环任一条件即可避免死锁。**

---

## 1. Go 并发死锁

### 典型案例：无缓冲 Channel 双向等待

```go
// 错误示例
func deadlock() {
    ch := make(chan int)
    ch <- 1 // 阻塞：无人接收
    <-ch   // 永远不会执行
}
```

### 典型案例：sync.Mutex 重复加锁

```go
// 错误示例
func (s *Struct) Do() {
    s.mu.Lock()
    s.mu.Lock() // 死锁！同一个 goroutine 重复加锁
    defer s.mu.Unlock()
}
```

Go 1.18+ 会检测到 `sync: unlock of unlocked mutex` 并 panic。

### 典型案例：嵌套 Channel 阻塞

```go
func nestedDeadlock() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go func() {
        <-ch1     // 等 ch1
        ch2 <- 1  // 往 ch2 写
    }()

    go func() {
        <-ch2     // 等 ch2
        ch1 <- 1  // 往 ch1 写
    }()

    // 两个 goroutine 互相等待，死锁
    time.Sleep(time.Second)
}
```

---

## 2. 数据库死锁

### 常见原因

| 场景 | 说明 |
|------|------|
| 不同事务顺序不一致 | T1 锁 A 等 B，T2 锁 B 等 A |
| gap lock 范围重叠 | 范围查询锁区间重叠 |
| 唯一索引插入 | 多事务插入相同键值 |
| 外键未建索引 | 父表删除导致子表全表锁 |

### 案例：顺序不一致

```sql
-- T1
BEGIN;
SELECT * FROM accounts WHERE id=1 FOR UPDATE; -- 锁 id=1
SELECT * FROM accounts WHERE id=2 FOR UPDATE; -- 锁 id=2

-- T2（并发执行）
BEGIN;
SELECT * FROM accounts WHERE id=2 FOR UPDATE; -- 等 T1 释放
SELECT * FROM accounts WHERE id=1 FOR UPDATE; -- 等 T1 释放 id=1
-- 死锁！T1: [1,2] T2: [2,1]
```

**解决：所有事务按固定顺序加锁。**

### 案例：gap lock

```sql
-- T1
SELECT * FROM orders WHERE status='pending' FOR UPDATE;
-- 锁住所有 status='pending' 的行及其 gap

-- T2（并发执行）
INSERT INTO orders VALUES (100, 'pending');
-- 被 T1 的 gap lock 阻塞

-- T3（并发执行）同样被阻塞，可能形成死锁
```

### 排查方法

```sql
-- MySQL 查看死锁日志
SHOW ENGINE INNODB STATUS;

-- 查看当前锁等待
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 查看锁信息
SELECT * FROM information_schema.INNODB_LOCKS;
```

### 处理策略

```go
// 代码层：重试机制
func withRetry(fn func() error, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        if isDeadlock(err) {
            time.Sleep(time.Duration(i+1) * 100 * time.Millisecond)
            continue
        }
        return err
    }
    return ErrMaxRetries
}
```

---

## 3. 分布式死锁

### Redis 分布式锁死锁

```go
// 错误：锁过期但任务未完成，另一个 goroutine 获取锁后任务重复执行
func badLock(lockKey string, ttl time.Duration) bool {
    // 设置 10s 过期
    ok, _ := redis.SetNX(lockKey, "1", ttl).Result()
    if !ok {
        return false
    }
    // 任务执行 15s
    longRunningTask()
    redis.Del(lockKey)
    return true
}

// 正确：延长锁 TTL
func extendLock(lockKey string, ttl time.Duration) bool {
    for {
        ok, _ := redis.SetNX(lockKey, "1", ttl).Result()
        if !ok {
            return false
        }
        time.Sleep(ttl / 3)
    }
}
```

### 数据库行锁死锁

分布式事务中，不同服务按不同顺序更新数据导致死锁。

**解决：全局统一事务顺序，如按主键 ID 排序。**

---

## 4. 死锁检测工具

### 1. Go race detector

```bash
go test -race ./...
go run -race main.go
```

检测：数据竞争 + 死锁。

### 2. pprof goroutine 分析

```bash
# 查看 goroutine 数量
curl http://localhost:6060/debug/pprof/goroutine?debug=1

# 生成 profile
go tool pprof http://localhost:6060/debug/pprof goroutine
```

### 3. MySQL InnoDB Monitor

```sql
-- 开启标准 InnoDB 监控
CREATE TABLE innodb_monitor (a INT) ENGINE=INNODB;
SHOW ENGINE INNODB STATUS;
DROP TABLE innodb_monitor;
```

### 4. 链路追踪

```bash
# Jaeger 查看 span 等待关系
# 父 span 长时间未结束，子 span 在等待，说明可能有死锁
```

---

## 5. 预防死锁的代码规范

### 1. 按固定顺序获取锁

```go
// 错误
func (a *Account) Transfer(b *Account, amount int) {
    a.mu.Lock()
    b.mu.Lock()
    defer { b.mu.Unlock(); a.mu.Unlock() }
}

// 正确：按地址排序
func (a *Account) Transfer(b *Account, amount int) {
    first, second := a, b
    if a > b { // 指针地址排序
        first, second = b, a
    }
    first.mu.Lock()
    second.mu.Lock()
    defer { second.mu.Unlock(); first.mu.Unlock() }
}
```

### 2. 最小锁范围

```go
// 差：锁范围过大
func (s *Store) Process() {
    s.mu.Lock()
    data := s.loadFromDB() // 耗时操作在锁内
    s.mu.Unlock()
}

// 好：只锁必要的写操作
func (s *Store) Process() {
    data := s.loadFromDB() // 锁外
    s.mu.Lock()
    s.value = compute(data)
    s.mu.Unlock()
}
```

### 3. 使用 Context 超时

```go
func withTimeout() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    done := make(chan error, 1)
    go func() {
        done <- doTask(ctx)
    }()

    select {
    case err := <-done:
        return err
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

### 4. 锁超时（trylock 模式）

```go
type TryMutex struct {
    ch chan struct{}
}

func NewTryMutex() *TryMutex {
    return &TryMutex{ch: make(chan struct{}, 1)}
}

func (m *TryMutex) Lock(timeout time.Duration) bool {
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        select {
        case m.ch <- struct{}{}:
            return true
        default:
            time.Sleep(time.Millisecond)
        }
    }
    return false
}

func (m *TryMutex) Unlock() {
    <-m.ch
}
```

---

## 6. 典型面试题

### Q1: Go 中 select 与死锁的关系？

```go
// 所有 case 都被阻塞且没有 default，select 会永久阻塞
select {} // 永久阻塞

// 带 default 的 select 不会阻塞
select {
case <-ch:
default:
    // 非阻塞
}
```

### Q2: 如何避免 Channel 死锁？

- 发送端和接收端至少有一个在另一个 goroutine 中
- 使用带缓冲的 channel：`make(chan T, N)`
- 使用 `range` 遍历 channel 时，确保发送端会 `close`

### Q3: 生产环境中死锁如何快速恢复？

1. **监控告警**：goroutine 数量突增 / 接口超时
2. **抓 goroutine stack**：`kill -QUIT <pid>` 或 pprof
3. **分析调用链**：找到相互等待的 goroutine
4. **重启止损**：先恢复，再复盘根因

### Q4: 数据库死锁和业务死锁有何区别？

| 维度 | 数据库死锁 | 业务死锁 |
|------|----------|---------|
| 范围 | 单机/集群 | 跨服务 |
| 检测 | InnoDB 自动检测并回滚 | 需自己实现 |
| 处理 | 自动回滚一个事务 | 需代码处理 |
| 预防 | 索引 + 顺序一致 | 统一顺序 + 超时 |
