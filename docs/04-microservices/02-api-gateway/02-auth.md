# 认证与授权：JWT、OAuth2、API Key

> 考察频率：★★★★☆  难度：★★★★☆

## 认证 vs 授权

| 概念 | 说明 | 问题 |
|------|------|------|
| **认证（Authentication）** | 你是谁 | 验证身份：登录 |
| **授权（Authorization）** | 你能做什么 | 权限检查：你能访问这个接口吗 |

---

## API Key

### 适用场景

- 服务间调用（M2M）
- 公开 API（开放平台）
- 不适合用户认证（无法精确到用户级别）

### 实现

```go
// 生成 API Key（存储在数据库）
func GenerateAPIKey(userID int64) (string, string, error) {
    keyID := randomString(16)    // keyId 用于标识
    secret := randomString(32)   // secret 用于签名
    hash := sha256.Sum256([]byte(secret))

    // 存储：keyId 明文存储，secret 哈希存储
    db.Exec(`INSERT INTO api_keys (user_id, key_id, secret_hash, created_at)
             VALUES (?, ?, ?, NOW())`,
        userID, keyID, hex.EncodeToString(hash[:]))

    return keyID, secret, nil  // secret 只在创建时返回一次
}

// 验证 API Key
func ValidateAPIKey(keyID, signature, timestamp string) (*int64, error) {
    row := db.QueryRow(`SELECT user_id, secret_hash FROM api_keys WHERE key_id = ?`, keyID)
    var userID int64
    var secretHash string
    row.Scan(&userID, &secretHash)

    // 验证时间戳（防止重放攻击，5分钟内有效）
    ts, _ := strconv.ParseInt(timestamp, 10, 64)
    if time.Now().Unix()-ts > 300 {
        return nil, ErrTimestampExpired
    }

    // HMAC 签名验证
    msg := keyID + timestamp
    mac := hmac.New(sha256.New, []byte(secretHash))
    expected := hex.EncodeToString(mac.Sum([]byte(msg)))
    if !hmac.Equal([]byte(signature), []byte(expected)) {
        return nil, ErrInvalidSignature
    }

    return &userID, nil
}
```

### 常见问题

- API Key 直接放在 URL 里？**不安全**，URL 会被日志记录，用 Header + HMAC 签名
- 时间戳过期？防止重放攻击，必须有

---

## JWT（JSON Web Token）

### 结构

```
Header.Payload.Signature
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxMjM0LCJleHAiOjE3MDAwMDAwMDAsImlzcyI6Im15LXNlcnZpY2UifQ.signature
```

- **Header**：算法 + 类型（HMAC SHA256 或 RSA）
- **Payload**：用户信息、过期时间、签发者
- **Signature**：Header.Payload 的签名，防止篡改

### Go 实现

```go
import (
    "github.com/golang-jwt/jwt/v5"
    "github.com/google/uuid"
)

type Claims struct {
    UserID    int64  `json:"user_id"`
    Username  string `json:"username"`
    Role      string `json:"role"`
    jwt.RegisteredClaims
}

// 生成 JWT
func GenerateJWT(userID int64, username, role string) (string, error) {
    now := time.Now()
    claims := Claims{
        UserID:   userID,
        Username: username,
        Role:     role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(now.Add(2 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(now),
            NotBefore: jwt.NewNumericDate(now),
            Issuer:    "my-service",
            Subject:   fmt.Sprintf("%d", userID),
            ID:        uuid.New().String(),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(jwtSecret))
}

// 验证 JWT
func ParseJWT(tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return []byte(jwtSecret), nil
    })

    if err != nil {
        return nil, err
    }

    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    return nil, ErrInvalidToken
}

// Gin 中间件
func JWTAuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        auth := c.GetHeader("Authorization")
        if !strings.HasPrefix(auth, "Bearer ") {
            c.AbortWithStatus(401)
            return
        }

        tokenStr := strings.TrimPrefix(auth, "Bearer ")
        claims, err := ParseJWT(tokenStr)
        if err != nil {
            c.AbortWithStatus(401)
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("role", claims.Role)
        c.Next()
    }
}
```

### JWT 的安全问题

| 问题 | 说明 | 解决 |
|------|------|------|
| **Token 泄露** | 任何人拿到都能用 | HTTPS + 短期 Token |
| **无法主动注销** | 服务端无法使 Token 失效 | 黑名单（Redis）或短 token |
| **Payload 明文** | 用户信息可见，只 Base64 编码 | 敏感信息不要放 Payload |
| **签名算法 NONE** | 攻击者伪造 alg: none | 验证时指定期望的算法 |

---

## OAuth2

### 四种授权模式

| 模式 | 适用场景 | 安全性 |
|------|---------|--------|
| Authorization Code | 有后端的应用 | ✅ 最高 |
| PKCE | SPA / 移动端 | ✅ 高 |
| Client Credentials | 服务间调用 | 中 |
| Password Grant | 极度信任的老应用 | ❌ 危险 |

### Authorization Code + PKCE

```go
// 1. 生成 PKCE 挑战
func GeneratePKCE() (verifier, challenge string) {
    verifier = randomString(64)
    hash := sha256.Sum256([]byte(verifier))
    challenge = base64.RawURLEncoding.EncodeToString(hash[:])
    return
}

// 2. 重定向到授权页面
func AuthorizeURL(clientID, redirectURI, state, codeChallenge string) string {
    params := url.Values{
        "response_type":         {"code"},
        "client_id":              {clientID},
        "redirect_uri":           {redirectURI},
        "state":                  {state},
        "code_challenge":         {codeChallenge},
        "code_challenge_method":  {"S256"},
    }
    return "https://auth.example.com/oauth/authorize?" + params.Encode()
}

// 3. 交换 Token
func ExchangeCode(code, codeVerifier, clientID, clientSecret string) (*TokenResponse, error) {
    resp, err := http.PostForm("https://auth.example.com/oauth/token", url.Values{
        "grant_type":    {"authorization_code"},
        "code":          {code},
        "redirect_uri":   {redirectURI},
        "client_id":     {clientID},
        "client_secret": {clientSecret},
        "code_verifier": {codeVerifier},
    })
    defer resp.Body.Close()

    var tokenResp TokenResponse
    json.NewDecoder(resp.Body).Decode(&tokenResp)
    return &tokenResp, nil
}

type TokenResponse struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
    ExpiresIn   int    `json:"expires_in"`
    TokenType   string `json:"token_type"`
}
```

### Token 刷新策略

```go
// Access Token 短期（15分钟），Refresh Token 长期（7天）
// 过期时用 Refresh Token 换新 Access Token

func RefreshAccessToken(refreshToken string) (*TokenResponse, error) {
    resp, err := http.PostForm("https://auth.example.com/oauth/token", url.Values{
        "grant_type":    {"refresh_token"},
        "refresh_token": {refreshToken},
        "client_id":     {clientID},
        "client_secret": {clientSecret},
    })
    // 返回新的 Access Token 和新的 Refresh Token
}
```

---

## 权限模型

### RBAC（Role-Based Access Control）

```
User ──属于──▶ Role ──拥有──▶ Permission
                     │
                     ▼
              menu.view / order.create / user.delete
```

```go
// 权限表
type Permission struct {
    ID   int64  `json:"id"`
    Code string `json:"code"`  // "order:create", "user:delete"
    Name string `json:"name"`
}

// 角色权限表
type RolePermission struct {
    RoleID   int64
    PermID   int64
}

// 检查权限
func CheckPermission(userID int64, requiredPerm string) bool {
    row := db.QueryRow(`
        SELECT COUNT(*) FROM user_roles ur
        JOIN role_permissions rp ON ur.role_id = rp.role_id
        JOIN permissions p ON rp.perm_id = p.id
        WHERE ur.user_id = ? AND p.code = ?
    `, userID, requiredPerm)
    var count int
    row.Scan(&count)
    return count > 0
}

// Gin 中间件：按权限码拦截
func RequirePerm(permCode string) gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := c.GetInt64("user_id")
        if !CheckPermission(userID, permCode) {
            c.JSON(403, gin.H{"code": 40301, "message": "permission denied"})
            c.Abort()
            return
        }
        c.Next()
    }
}

// 使用
router.POST("/orders", JWTAuthMiddleware(), RequirePerm("order:create"), CreateOrder)
```

### 细粒度权限：Casbin

```go
import "github.com/casbin/casbin/v2"

// 策略文件：model.conf
// [request_definition]
// r = sub, obj, act
// [policy_definition]
// p = sub, obj, act
// [policy_effect]
// e = some(where (p.eft == allow))
// [matchers]
// m = r.sub == p.sub && r.obj == p.obj && r.act == p.act

enforcer, _ := casbin.NewEnforcer("model.conf", "policy.csv")

// 检查权限
ok, _ := enforcer.Enforce("alice", "/orders", "create")  // true/false

// 批量加载策略
enforcer.LoadPolicy()
```

---

## 面试话术

**Q：JWT 和 Session 哪个更好？**

> 各有优劣。Session 的优势是服务端可控，随时可以主动失效，适合安全性要求高的场景；缺点是需要共享存储（Redis），水平扩展麻烦。JWT 的优势是无状态、自包含，天然适合微服务和分布式；缺点是签发后无法主动失效（除非用黑名单），Payload 是明文的不能放敏感信息。互联网业务 Token 模式更常见，涉金融/权限敏感场景用 Session。

**Q：Refresh Token 为什么不能和 Access Token 一起返回有效期？**

> 因为 Access Token 短期（15 分钟），Refresh Token 长期（7 天）。如果前者快过期了直接刷新，不用让用户重新登录。Refresh Token 只在 Access Token 过期时才用，防止频繁调用认证服务。

**Q：接口幂等是怎么保证的？**

> GET 是天然幂等的，POST/PUT/DELETE 要额外处理。常用方案：1）数据库唯一键，重复写入报错；2）Redis 幂等键；3）接口防重令牌（Token + 去重表）。支付这类场景必须强幂等，用分布式锁 + 唯一流水号。
