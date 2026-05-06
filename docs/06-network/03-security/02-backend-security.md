# 后端安全体系

> 考察频率：★★★★☆  优先级：P1
> 关键词：JWT、认证、鉴权、最小权限、审计日志、敏感数据加密、密钥轮换

## 1. 服务间认证 vs 用户认证 vs 内部调用认证

### 三种认证场景对比

| 场景 | 谁调用谁 | Token 类型 | 有效期 | 典型方案 |
|------|---------|-----------|--------|---------|
| **用户认证** | 用户 APP → 后端 | JWT / Session | 短（15min~7d）| JWT + Refresh Token |
| **服务间认证** | Service A → Service B | API Key / MTLS | 长/永久 | API Key / MTLS |
| **内部调用认证** | 网关 → 后端服务 | Service Token | 短（1h）| HMAC / JWT（内部签发）|

### 用户认证（JWT）

```go
// JWT 生成
type Claims struct {
    UserID   uint64 `json:"user_id"`
    Username string `json:"username"`
    jwt.RegisteredClaims
}

func generateAccessToken(userID uint64, username string) (string, error) {
    now := time.Now()
    claims := Claims{
        UserID:   userID,
        Username: username,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(now.Add(15 * time.Minute)),  // 15 分钟过期
            IssuedAt:  jwt.NewNumericDate(now),
            Issuer:    "my-app",
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(jwtSecret))
}

// JWT 验证中间件
func JWTAuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        auth := c.GetHeader("Authorization")
        tokenStr := strings.TrimPrefix(auth, "Bearer ")

        token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (interface{}, error) {
            return jwtSecret, nil
        })
        if err != nil || !token.Valid {
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
            return
        }
        claims := token.Claims.(*Claims)
        c.Set("user_id", claims.UserID)
        c.Next()
    }
}
```

### 服务间认证（API Key / MTLS）

```go
// 方案 1：API Key（简单场景）
func apiKeyAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        apiKey := c.GetHeader("X-API-Key")
        expected := os.Getenv("INTERNAL_API_KEY")

        // 固定字符串比对（可能被时序攻击）
        if !hmac.Equal([]byte(apiKey), []byte(expected)) {
            c.AbortWithStatusJSON(403, gin.H{"error": "invalid api key"})
            return
        }
        c.Next()
    }
}

// 方案 2：HMAC 签名（防篡改）
// 客户端：sign = HMAC-SHA256(method + path + timestamp + body, secret_key)
// 服务端：校验 signature + timestamp（防止重放）
func hmacAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        sig := c.GetHeader("X-Signature")
        ts := c.GetHeader("X-Timestamp")
        method := c.Request.Method
        path := c.Request.URL.Path

        // 1. 检查时间戳（5 分钟内的请求才有效，防止重放）
        tsInt, _ := strconv.ParseInt(ts, 10, 64)
        if time.Now().Unix()-tsInt > 300 {
            c.AbortWithStatusJSON(403, gin.H{"error": "request expired"})
            return
        }

        // 2. 重新计算签名并比对
        body, _ := io.ReadAll(c.Request.Body)
        msg := method + path + ts + string(body)
        expectedSig := hmacSHA256(msg, internalSecret)

        if !hmac.Equal([]byte(sig), []byte(expectedSig)) {
            c.AbortWithStatusJSON(403, gin.H{"error": "invalid signature"})
            return
        }
        c.Next()
    }
}
```

---

## 2. 最小权限原则如何落地

### 原则：每个服务/用户只拥有完成任务所需的最小权限

```
❌ 错误做法：
- 所有服务共用同一个 API Key
- 服务 A 有权访问服务 B 的数据库
- 用户注册后默认拥有管理员权限

✅ 正确做法：
- 服务间按功能分配独立 API Key（A 服务只能写订单，不能读用户表）
- 每个数据库用户权限最小化（只授权需要的表/字段）
- RBAC 角色权限模型：新用户默认无权限，逐项申请
```

### 数据库最小权限示例

```sql
-- 应用服务只读订单表
CREATE USER 'order_service'@'%' IDENTIFIED BY 'xxx';
GRANT SELECT, INSERT, UPDATE ON orders.orders TO 'order_service'@'%';

-- 定时任务服务只能执行存储过程
CREATE USER 'cron_job'@'%' IDENTIFIED BY 'xxx';
GRANT EXECUTE ON PROCEDURE orders.calc_daily_stats TO 'cron_job'@'%';

-- 禁止：永远不要给应用 GRANT ALL
GRANT ALL ON orders.* TO 'app_user'@'%'; -- ❌ 太危险
```

---

## 3. 审计日志应记录什么

### 审计日志核心要素

```go
type AuditLog struct {
    Timestamp    time.Time
    UserID       string
    UserName     string
    Action       string        // "ORDER_CREATE", "USER_DELETE", "CONFIG_UPDATE"
    Resource     string        // "orders:12345", "users:99"
    ResourceType string        // "order", "user", "config"
    Result       string         // "success", "failed", "denied"
    Detail       string         // 额外信息（JSON 格式）
    ClientIP     string
    UserAgent    string
    RequestID    string        // 链路追踪 ID
}

// 需要记录的操作：
// ✅ 用户登录/登出（成功 + 失败）
// ✅ 敏感操作（删除订单、修改权限、导出数据）
// ✅ 资金相关（支付、退款、提现）
// ✅ 管理操作（添加管理员、修改配置）
// ✅ 批量操作（批量删除、批量导出）

// 不需要记录：
// ❌ 只读操作（查询订单、查看列表）
// ❌ 无关紧要的操作
```

### Go 审计日志实现

```go
type AuditLogger struct {
    db *sqlx.DB
}

func (l *AuditLogger) Log(ctx context.Context, entry AuditLog) error {
    detailJSON, _ := json.Marshal(entry.Detail)
    return l.db.ExecContext(ctx, `
        INSERT INTO audit_logs
            (timestamp, user_id, username, action, resource, resource_type,
             result, detail, client_ip, user_agent, request_id)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    `,
        entry.Timestamp, entry.UserID, entry.UserName, entry.Action,
        entry.Resource, entry.ResourceType, entry.Result,
        string(detailJSON), entry.ClientIP, entry.UserAgent, entry.RequestID,
    )
}

// 审计日志切面（拦截敏感操作）
func AuditLogMiddleware(action, resourceType string) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        // 业务逻辑执行完后记录审计日志
        entry := AuditLog{
            Timestamp:    time.Now(),
            UserID:       fmt.Sprintf("%v", c.Get("user_id")),
            Action:       action,
            Resource:     c.Param("id"),
            ResourceType: resourceType,
            Result:       getResultFromStatus(c.Writer.Status()),
            ClientIP:     c.ClientIP(),
            RequestID:    c.GetHeader("X-Request-ID"),
        }
        auditLogger.Log(c.Request.Context(), entry)
    }
}
```

---

## 4. 敏感数据处理

### 敏感数据分类

```
🔴 极高敏感：密码、支付密码、身份证号、银行卡号
🟠 高敏感：手机号、邮箱、地址、交易记录
🟡 中敏感：昵称、头像、订单号
🟢 低敏感：公开可见内容
```

### 加密存储

```go
import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/base64"
)

// AES-GCM 加密（可搜索加密，不暴露明文但支持等值查询）
func encryptAES(plaintext, key []byte) (string, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return "", err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }

    nonce := make([]byte, gcm.NonceSize())
    rand.Read(nonce)  // 每次随机 nonce

    ciphertext := gcm.Seal(nonce, nonce, plaintext, nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

// 密码存储：bcrypt（单向哈希，存储 hash 不存原文）
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(hash), err
}

func checkPassword(hash, password string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

### 脱敏展示

```go
// 手机号脱敏：138****5678
func maskPhone(phone string) string {
    if len(phone) < 11 {
        return phone
    }
    return phone[:3] + "****" + phone[7:]
}

// 身份证脱敏：110***********1234
func maskIDCard(id string) string {
    if len(id) < 8 {
        return id
    }
    return id[:3] + "***********" + id[len(id)-4:]
}

// 银行卡脱敏：**** **** **** 1234
func maskBankCard(card string) string {
    if len(card) < 4 {
        return card
    }
    return "**** **** **** " + card[len(card)-4:]
}
```

---

## 5. 密钥轮换与 Secret 管理

### 密钥轮换策略

```
最佳实践：
1. 定期轮换：每 90 天轮换一次 API Key / JWT Secret
2. 灰度切换：新旧密钥并存一段时间，平滑过渡
3. 紧急轮换：发现泄露后立即轮换，并审计日志

实现：密钥版本号
key_v1 = hash(password) + "v1"  → 验证时同时支持 v1 和 v2
key_v2 = hash(password) + "v2"
升级后：验证时优先用 v2，不支持 v1
```

```go
// 密钥版本管理
type SecretManager struct {
    currentVersion int
    secrets        map[int][]byte  // version → key
}

func (m *SecretManager) GetCurrent() []byte {
    return m.secrets[m.currentVersion]
}

func (m *SecretManager) Rotate(newKey []byte) {
    m.currentVersion++
    m.secrets[m.currentVersion] = newKey
    // 异步通知所有服务刷新密钥（配置中心推送）
    configCenter.Notify("secret_rotated", m.currentVersion)
}

// JWT 验证时支持多个版本
func verifyJWTMultiVersion(tokenStr string, manager *SecretManager) (*Claims, error) {
    for ver := manager.currentVersion; ver >= 1; ver-- {
        key := manager.secrets[ver]
        token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (interface{}, error) {
            return key, nil
        })
        if err == nil && token.Valid {
            return token.Claims.(*Claims), nil
        }
    }
    return nil, errors.New("invalid token for all versions")
}
```

### Secret 管理工具

| 工具 | 特点 | 适用场景 |
|------|------|---------|
| **Vault** | 支持动态 Secret、K/V 加密、审计 | 大型团队 |
| **AWS Secrets Manager** | 与 AWS IAM 集成，支持轮换 | AWS 云 |
| **K8s Secrets** | K8s 原生，简单易用 | K8s 环境 |
| **Nacos** | 配置中心+Secret 管理 | 国内公司常用 |
| **Apollo** | 配置中心，支持加密配置 | 国内公司常用 |

---

## 6. 如何回答"你们怎么做安全"

### 高级工程师的安全体系认知

```
分层安全体系：

┌────────────────────────────────────┐
│  身份认证层                         │
│  JWT + Refresh Token + API Key     │
├────────────────────────────────────┤
│  权限控制层                         │
│  RBAC + 数据范围过滤                │
├────────────────────────────────────┤
│  网关防护层                         │
│  WAF + 限流 + 黑白名单               │
├────────────────────────────────────┤
│  数据安全层                         │
│  敏感数据加密 + 脱敏 + 审计日志      │
├────────────────────────────────────┤
│  基础设施层                         │
│  网络隔离 + VPC + TLS + Secret 管理  │
└────────────────────────────────────┘
```

**面试回答框架（STAR 法则）：**
```
S（情境）：我们团队遇到过 XX 安全问题
T（任务）：我负责设计/改进安全体系
A（行动）：
  1. 引入 JWT + RBAC 鉴权，解决身份认证问题
  2. 关键操作加审计日志，满足合规要求
  3. 敏感数据用 AES 加密存储，展示时脱敏
  4. 引入 Vault 管理密钥，实现定期轮换
R（结果）：XX 安全事件减少 X%，通过等保三级认证
```

### OWASP Top 10 防护清单

| OWASP 风险 | 防护措施 |
|-----------|---------|
| **A01 访问控制失效** | RBAC + 每次验权 + 审计日志 |
| **A02 加密失败** | AES-256-GCM 加密存储，TLS 1.2+ |
| **A03 注入** | 参数化查询 + 输入校验 + ORM |
| **A04 不安全设计** | 威胁建模 + 安全设计评审 |
| **A05 安全配置错误** | 最小镜像 + 禁用调试接口 |
| **A06 敏感数据泄露** | 加密存储 + 脱敏展示 + HTTPS |
| **A07 认证失败** | JWT + 防暴力破解 + 登录锁定 |
| **A08 数据完整性失败** | 签名 + 版本控制 + 审计日志 |
| **A09 日志监控不足** | 结构化日志 + 异常告警 + SIEM |
| **A10 API 安全** | 限流 + 鉴权 + 输入校验 |
