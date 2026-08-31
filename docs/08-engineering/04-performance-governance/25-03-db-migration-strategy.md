# Go 数据库迁移：零停机变更策略实战

> 考察频率：★★★★☆  优先级：P1
> 关键词：golang-migrate、Goose、向后兼容、数据一致性、滚动部署、Schema migration

## 面试官考察意图

**"线上改数据库结构怎么保证不中断服务"** 是高级 Go 后端面试的经典问题。这道题考的不是工具使用，而是**架构演进的思维**——能否在不停机的情况下完成 schema 变更。

面试官想看的是候选人的系统性方案：从方案设计 → 版本兼容 → 分批发布 → 回滚预案，而不是只会说"加个字段而已"。

---

## 核心答案（30 秒版）

数据库迁移的核心原则是**渐进式变更 + 向后兼容**。分四步走：**第一步新增字段/索引（不影响旧代码）→ 第二步双写数据到新字段 → 第三步历史数据同步 → 第四步移除旧字段/切换逻辑**。每个迁移步骤通过 CI/CD 独立部署，配合金丝雀发布保证安全。**永远不要在单个部署中同时改 schema 和改代码。**

---

## 渐进式 Schema 变更的五个阶段

### 📌 场景：给订单表增加 `promotional_code` 字段并做业务改造

```
时间线：D=0 开始
```

### 第一阶段：只加字段（DDL only）

```sql
-- migration: V002__add_promo_field.sql
ALTER TABLE orders ADD COLUMN promo_code VARCHAR(64) DEFAULT '';

-- ✅ 旧代码不受影响 — 新字段有默认值 ""
-- ✅ 可以回滚 — DROP COLUMN 即可
-- ⚠️ 注意：大表 ALTER 要在线执行 (gh-ost / pt-online-schema-change)
```

**关键点：** 此阶段只改数据库不改代码。先部署这条 DDL，确认生效。

### 第二阶段：开启双写（新旧并存）

```go
// 应用层同时写入两个字段
func CreateOrder(ctx context.Context, req *CreateOrderReq) (*Order, error) {
    // 旧字段继续写
    order := &Order{
        UserID:       req.UserID,
        Amount:       req.Amount,
        Status:       "pending",
        PromoCode:    "",  // 空字符串
    }
    
    // 新字段也开始写（解析促销码）
    if code, ok := extractPromoCode(req); ok {
        order.PromoCode = code
    }
    
    return db.Save(order)
}

// ✅ 旧客户端不传 promoCode → 写入空串
// ✅ 新客户端传入 promoCode → 正确写入
// ✅ 没有数据不一致 — 只是多了一个有效字段
```

### 第三阶段：历史数据回填

```sql
-- 用批量脚本更新历史数据
-- 推荐：按 batch 分批更新，避免锁表
UPDATE orders SET promo_code = resolve_promo(user_id, created_at)
WHERE promo_code = '' AND created_at < '2026-08-01'
LIMIT 1000;
-- 循环执行直到影响行为 0
```

或使用 Go 程序分批处理：

```go
func migrateHistoricalData(ctx context.Context) error {
    const batchSize = 1000
    offset := 0
    
    for {
        result, err := db.ExecContext(ctx,
            `UPDATE orders SET promo_code = ? WHERE promo_code = '' LIMIT ?`,
            calcPromoCode(offset), batchSize,
        )
        if err != nil {
            return err
        }
        
        rowsAffected, _ := result.RowsAffected()
        if rowsAffected == 0 {
            break // 全部处理完毕
        }
        offset += int(rowsAffected)
        
        time.Sleep(50 * time.Millisecond) // 控制对 DB 的影响
    }
    
    slog.Info("historical data migration complete", 
        slog.Int("rows", offset))
    return nil
}
```

### 第四阶段：切换读写逻辑

```go
// 现在安全地依赖 promo_code 字段了
func ProcessPayment(ctx context.Context, orderID string) error {
    order, err := GetOrder(ctx, orderID)
    if err != nil {
        return err
    }
    
    // 现在可以放心地使用 promo_code
    if order.PromoCode != "" {
        discount, err := applyPromotion(order.PromoCode)
        if err != nil {
            return fmt.Errorf("promo failed: %w", err)
        }
        order.Amount -= discount
    }
    
    return processPayment(order)
}
```

### 第五阶段：清理旧逻辑

```sql
-- 等待所有流量切到新版后，可以：
-- 1. DROP COLUMN（再次在线执行）
-- 2. 或者保留一段时间作为"软删除"验证期
ALTER TABLE orders DROP COLUMN promo_code;
```

---

## golang-migrate 工具集成

### 项目结构

```
├── migrations/
│   ├── V001__create_orders_table.sql
│   ├── V002__add_promo_field.sql
│   ├── V003__add_idx_order_status.sql
│   └── V004__add_user_email_index.sql
├── cmd/
│   └── migrator/
│       └── main.go    # 迁移执行入口
```

### Go 实现

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "os"

    _ "github.com/lib/pq"

    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func main() {
    dsn := os.Getenv("DATABASE_URL")
    migrationsURL := "file://migrations"

    m, err := migrate.New(migrationsURL, dsn)
    if err != nil {
        log.Fatal(err)
    }
    defer m.Close()

    if err := m.Up(); err != nil {
        if err == migrate.ErrNoChange {
            log.Println("No new migrations to apply")
            return
        }
        log.Fatalf("Migration failed: %v", err)
    }
    log.Println("All migrations applied successfully")
}
```

### CI/CD 中的自动化

```yaml
# .gitlab-ci.yml
migrate:
  stage: deploy-staging
  image: golang:1.23-alpine
  before_script:
    - go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest
  script:
    - mkdir -p /tmp/migrations
    - cp -r migrations/* /tmp/migrations/
    - migrate -path /tmp/migrations -database "$DATABASE_URL" up
  environment:
    name: staging
  when: manual
```

---

## 常见陷阱与反模式

### ❌ 致命错误：一步到位

```go
// ❌ 绝对不要这样做！
// 一次性改了 schema 又改了代码 = 部署时必然有一个时间点 schema 和代码不匹配

// Step A: 删除旧的 user_name 字段
ALTER TABLE users DROP COLUMN user_name;

// Step B: 代码里还在读 user_name
name := row.Scan(&user.Name)  // panic! column doesn't exist
```

### ❌ 致命错误：大表直接 ALTER

```sql
-- ❌ 对于千万级表，这个语句会锁表几十分钟
ALTER TABLE orders ADD INDEX idx_created_at (created_at);

-- ✅ 使用 gh-ost 在线执行
gh-ost \
  --host=db-host \
  --port=3306 \
  --user=migrate \
  --password=$PASSWORD \
  --database=mydb \
  --table=orders \
  --alter="ADD INDEX idx_created_at (created_at)" \
  --allow-on-master \
  --execute
```

### ❌ 致命错误：回滚方案不存在

```bash
# 每个 Migration 必须有对应的 down 脚本
# V003__add_idx.sql 对应 V003__remove_idx.down.sql

# 一键回滚
migrate -path migrations -database "$DB" force 3  # 回到第3步
migrate -path migrations -database "$DB" down      # 反向执行
```

---

## 迁移安全清单

```
[ ] DDL 是否在测试环境完整验证？
[ ] 是否有对应的 Down 迁移脚本？
[ ] 是否考虑了 backward compatibility？（旧代码能正常读新 schema）
[ ] 大表操作是否用了在线工具（gh-ost / pt-osc）？
[ ] 是否配置了连接池超时保护？（避免长时间持有连接）
[ ] 是否有监控告警？（迁移期间的错误率/QPS 波动）
[ ] 回滚方案是否经过演练？
```

---

## 面试话术

> "我之前的经验是分五步来做 Schema 变更：先是纯 DDL 加字段（兼容旧代码），然后开启双写让新旧客户端共存，接着用脚本批量回填历史数据，等全量部署后再切换读写逻辑，最后才移除旧字段。关键原则是**每次只做一个方向的改动**，确保任何时候都能回滚。大表 DDL 用 gh-ost 在线执行，不锁表。"

---

[🏠 首页](../../../README.md) · [📦 工程素养](../README.md) · [🛡️ 技术领导力](./README.md)
