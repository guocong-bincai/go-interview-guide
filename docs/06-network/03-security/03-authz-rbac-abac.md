# RBAC、ABAC 与统一权限设计

> 考察频率：★★★★☆  优先级：P1
> 关键词：RBAC、ABAC、ACL、OIDC、OAuth2、权限中间件、多租户

## 三种权限模型对比

### ACL（Access Control List）

最简单粗暴的模型，直接维护"用户-资源-权限"列表：

```go
// ACL 示例：每个资源维护一个用户列表
type ACL struct {
    UserPermissions map[string]map[string]bool // resource → user → permission
}

// 判断用户能否读某个文档
func (a *ACL) CanRead(userID, docID string) bool {
    perms := a.UserPermissions[docID]
    return perms[userID] || false
}
```

**问题：**
- 用户和资源多了之后，条目数爆炸（O(用户数 × 资源数)）
- 新增资源要手动配置所有相关用户
- 适用于资源少、用户少的小型系统

### RBAC（Role-Based Access Control）

通过**角色**解耦用户和权限：

```
RBAC 模型：
User → Role → Permission
         ↓
      Role hierarchy（角色继承）

示例：
用户 A（Admin 角色）→ 可以读/写/删所有资源
用户 B（Editor 角色）→ 可以读/写部分资源
用户 C（Viewer 角色）→ 只能读
```

**Go 数据结构：**

```go
type User struct {
    ID        uint64
    Username  string
    RoleIDs   []uint64  // 一个用户可以有多个角色
}

type Role struct {
    ID        uint64
    Name      string   // "admin", "editor", "viewer"
    ParentID  *uint64  // 角色继承（可选）
}

type Permission struct {
    ID         uint64
    Resource   string  // "document", "order", "user"
    Action     string  // "read", "write", "delete"
    Scope      string  // "own", "department", "all"  // 数据范围
}

// 用户权限查询
type RBACService struct {
    db *gorm.DB
}

func (s *RBACService) GetUserPermissions(userID uint64) ([]Permission, error) {
    var perms []Permission
    err := s.db.Raw(`
        SELECT DISTINCT p.*
        FROM users u
        JOIN user_roles ur ON u.id = ur.user_id
        JOIN roles r ON r.id = ur.role_id
        JOIN role_permissions rp ON rp.role_id = r.id
        JOIN permissions p ON p.id = rp.permission_id
        WHERE u.id = ?
    `, userID).Scan(&perms).Error
    return perms, err
}
```

### ABAC（Attribute-Based Access Control）

基于**属性**（用户属性 + 资源属性 + 环境属性）做动态授权：

```go
// ABAC 策略评估
type ABACPolicy struct {
    Subject  SubjectAttrs  // 谁（用户属性：部门、职级、地区）
    Resource ResourceAttrs // 什么（资源属性：创建者、密级、项目）
    Action   string        // 做什么
    Environment EnvAttrs   // 什么环境（时间、IP、设备）
}

type SubjectAttrs struct {
    UserID     uint64
    Department string   // 所属部门
    Level      int      // 职级（1=初级，5=高管）
    Region     string   // 所属地区
}

type ResourceAttrs struct {
    Creator     uint64   // 创建者
    Department  string   // 所属部门
    SecretLevel int      // 密级（1=公开，5=机密）
}

// ABAC 评估引擎
func Evaluate(policy ABACPolicy) bool {
    // 规则1：自己创建的文档可以删除
    if policy.Action == "delete" && policy.Subject.UserID == policy.Resource.Creator {
        return true
    }
    // 规则2：职级 >= 3 且同部门的可以读
    if policy.Action == "read" &&
        policy.Subject.Level >= 3 &&
        policy.Subject.Department == policy.Resource.Department {
        return true
    }
    // 规则3：机密文档只有职级 >= 4 才能访问
    if policy.Resource.SecretLevel >= 4 && policy.Subject.Level < 4 {
        return false
    }
    return false
}
```

### 三者对比

| 维度 | ACL | RBAC | ABAC |
|------|-----|------|------|
| 复杂度 | O(n×m) | O(n+m) | 动态计算 |
| 适用规模 | 小型系统 | 中大型系统 | 超大型/云原生 |
| 灵活性 | 低 | 中 | 高 |
| 权限粒度 | 粗 | 中 | 细 |
| 数据范围控制 | ❌ 不支持 | ⚠️ 需扩展 | ✅ 原生支持 |
| 性能 | 高（查表）| 高（查角色）| 中（策略计算）|

---

## OAuth2 与 OIDC

### OAuth2 四种授权模式

```
OAuth2 目的：让第三方应用获取用户数据，但不需要提供用户名密码

授权码模式（最安全，适合前后端分离）：
1. 用户点击"用 Google 登录"
2. 跳转到 Google 授权页，用户授权
3. Google 回调 → 返回 AUTHORIZATION CODE（临时）
4. 后端用 CODE 换 ACCESS TOKEN（后端之间通信，安全）
5. 用 ACCESS TOKEN 访问 Google API

简化模式（不安全，不推荐）：直接返回 ACCESS TOKEN
密码凭证模式：用户直接给第三方用户名密码（内部服务才用）
客户端模式：client_id + client_secret（M2M，服务间调用）
```

### OIDC = OAuth2 + ID Token

OIDC 在 OAuth2 基础上增加了一个 `id_token`，包含用户身份信息：

```go
// OIDC 认证流程（Go + OIDC 库）
import "github.com/coreos/go-oidc/v3/oidc"

func oidcAuth(provider *oidc.Provider, verifier *oidc.IDTokenVerifier) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. 从请求中取 id_token（Authorization: Bearer xxx）
        rawToken := c.GetHeader("Authorization")
        tokenStr := strings.TrimPrefix(rawToken, "Bearer ")

        // 2. 验证 token 签名和过期时间
        idToken, err := verifier.Verify(c.Request.Context(), tokenStr)
        if err != nil {
            c.AbortWithStatus(401)
            return
        }

        // 3. 提取用户信息（claims）
        var claims struct {
            Subject   string `json:"sub"`
            Email     string `json:"email"`
            Name      string `json:"name"`
            Exp       int64  `json:"exp"`
        }
        idToken.Claims(&claims)

        c.Set("user_id", claims.Subject)
        c.Set("email", claims.Email)
        c.Next()
    }
}
```

### OAuth2 vs OIDC 区别

| | OAuth2 | OIDC |
|--|--------|------|
| 核心 | 授权（你能访问什么资源）| 认证（你是谁）|
| Token | Access Token | Access Token + **ID Token** |
| Token 格式 | 不透明（OAuth2 服务商定义）| JWT（标准格式）|
| 用户信息 | 需额外调 `/userinfo` 接口 | ID Token 内含用户信息 |
| 适用场景 | 第三方登录、API 授权 | 用户登录、SSO |

---

## 企业后台权限系统设计

### 权限模型：RBAC + 数据范围

```
整体权限模型：
┌─────────────────────────────────────────────┐
│  用户 - 角色 - 权限                          │
│  + 数据范围（自己/本部门/全公司）              │
└─────────────────────────────────────────────┘

例如：
- 运营角色：可查看所有订单（数据范围 = 全公司）
- 销售角色：只能查看自己负责的客户订单（数据范围 = 自己）
- 财务角色：只能查看财务相关的文档（资源权限）
```

### 表结构设计

```sql
-- 用户表
CREATE TABLE users (
    id        BIGINT PRIMARY KEY,
    username  VARCHAR(64) NOT NULL,
    dept_id   BIGINT,  -- 所属部门
    UNIQUE (username)
);

-- 部门表（树形结构）
CREATE TABLE departments (
    id        BIGINT PRIMARY KEY,
    name      VARCHAR(128),
    parent_id BIGINT,
    path      VARCHAR(512)  -- '/1/3/5/' 用于快速查子部门
);

-- 角色表
CREATE TABLE roles (
    id   BIGINT PRIMARY KEY,
    name VARCHAR(64)   -- 'admin', 'editor', 'viewer'
);

-- 菜单/资源表
CREATE TABLE menus (
    id        BIGINT PRIMARY KEY,
    name      VARCHAR(128),
    path      VARCHAR(256),   -- '/orders/list'
    parent_id BIGINT,
    menu_type TINYINT  -- 1=菜单, 2=按钮, 3=接口
);

-- 角色-菜单权限表
CREATE TABLE role_menus (
    role_id BIGINT,
    menu_id  BIGINT,
    PRIMARY KEY (role_id, menu_id)
);

-- 用户-角色表
CREATE TABLE user_roles (
    user_id BIGINT,
    role_id BIGINT,
    PRIMARY KEY (user_id, role_id)
);
```

### 数据范围控制（行级权限）

```go
// 数据范围枚举
const (
    ScopeAll       = "all"       // 全公司数据
    ScopeDept      = "dept"      // 本部门数据
    ScopeOwn       = "own"       // 仅自己的数据
)

// 查询订单时，自动注入数据范围过滤
func (s *OrderService) ListOrders(userID uint64, deptID uint64, roleScope string) *gorm.DB {
    q := s.db.Model(&Order{})

    switch roleScope {
    case ScopeAll:
        // 不加过滤，能查所有
    case ScopeDept:
        // 同部门的订单
        q = q.Where("dept_id = ?", deptID)
    case ScopeOwn:
        // 只能查自己的
        q = q.Where("creator_id = ?", userID)
    }
    return q
}
```

---

## 多租户权限模型

### 方案 1：共享数据库 + tenant_id 字段隔离

```go
// 每个表加 tenant_id 字段
type Order struct {
    ID       uint64
    TenantID uint64  // 租户 ID
    Amount   float64
}

// 全局中间件：注入 tenant_id
func TenantMiddleware(tenantID uint64) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("tenant_id = ?", tenantID)
    }
}

// 使用
func ListOrders(db *gorm.DB, tenantID uint64) {
    db.Scopes(TenantMiddleware(tenantID)).Find(&orders)
}
```

### 方案 2：独立 Schema（PostgreSQL）

```sql
-- PostgreSQL 支持 schema 隔离
CREATE SCHEMA tenant_1001;   -- 租户 A 的 schema
CREATE SCHEMA tenant_1002;   -- 租户 B 的 schema
-- 每个租户的数据在独立 schema，查询时动态切换 search_path
SET search_path TO tenant_1001;
```

### 方案 3：独立数据库（完全隔离）

```
适用场景：金融、医疗等高安全要求，数据必须物理隔离
缺点：运维成本高（每个租户一套数据库实例）
```

---

## Go 统一鉴权中间件

### JWT + 权限验证

```go
func AuthMiddleware(jwtSecret []byte) gin.HandlerFunc {
    return func(c *gin.Context) {
        tokenStr := c.GetHeader("Authorization")
        if tokenStr == "" {
            c.AbortWithStatusJSON(401, gin.H{"error": "no token"})
            return
        }
        tokenStr = strings.TrimPrefix(tokenStr, "Bearer ")

        // 解析 JWT
        token, err := jwt.Parse(tokenStr, func(t *jwt.Token) (interface{}, error) {
            return jwtSecret, nil
        })
        if err != nil || !token.Valid {
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
            return
        }

        claims := token.Claims.(jwt.MapClaims)
        c.Set("user_id", claims["sub"])
        c.Set("tenant_id", claims["tenant_id"])
        c.Set("roles", claims["roles"])
        c.Next()
    }
}

// 权限检查中间件
func RequirePermission(requiredPerm string) gin.HandlerFunc {
    return func(c *gin.Context) {
        roles, exists := c.Get("roles")
        if !exists {
            c.AbortWithStatusJSON(403, gin.H{"error": "no role"})
            return
        }

        // 从数据库/缓存加载角色对应权限
        userRoles := roles.([]string)
        perms := loadUserPermissions(userRoles)

        if !contains(perms, requiredPerm) {
            c.AbortWithStatusJSON(403, gin.H{"error": "permission denied"})
            return
        }
        c.Next()
    }
}

// Gin 路由使用
router.GET("/orders",
    AuthMiddleware(jwtSecret),
    RequirePermission("order:read"),
    OrderListHandler,
)
```

### 菜单权限 + 接口权限 + 数据权限三层控制

```go
// 三层权限模型
type Permission struct {
    Menu  string  // 菜单级（控制左侧菜单显示）
    API   string  // 接口级（控制能否调用此 API）
    Data  string  // 数据级（控制能查哪些数据行）
}

// 认证流程：
// 1. 用户登录 → 获取 JWT（包含 user_id, tenant_id, roles）
// 2. 前端请求时，JWT 随请求发送
// 3. 中间件验证 JWT → 解析出 user_id
// 4. RequireMenu("order-list") → 检查用户是否有该菜单权限（前端控制）
// 5. RequireAPI("order:read") → 检查用户是否有该接口权限（网关控制）
// 6. Service 层 Scope(userID, deptID) → 自动注入数据范围过滤（后端控制）
```

---

## 高频追问

### Q1：菜单权限、接口权限、数据权限如何拆分？

**三层分离的好处：灵活组合，职责清晰：**

```go
// 菜单权限（前端控制）
// 前端根据用户的菜单权限渲染左侧菜单
// GET /api/v1/user/menus → 返回用户可见的菜单树

// 接口权限（网关/BFF 层控制）
// 请求到达服务前，先检查用户能否调用此 API
// 兜底防护，防止前端绕过菜单直接调用接口

// 数据权限（Service 层控制，最核心）
// 无论前端传什么过滤条件，Service 层都要加一层数据范围过滤
// 防止 SQL 注入绕过 where 条件

// 最佳实践：三权分立，互相兜底
func ListOrders(c *gin.Context) {
    userID := c.GetUint64("user_id")
    deptID := getUserDept(userID)
    roleScope := getRoleScope(userID)  // "all" / "dept" / "own"

    // 数据权限过滤（Service 层，必须）
    orders := orderService.List(c,
        WithDataScope(userID, deptID, roleScope),  // 自动加 WHERE
        WithStatus(c.Query("status")),
    )
}
```

### Q2：如何防止越权访问（用户 A 访问用户 B 的数据）？

**三个层次都要防护：**

1. **前端防护**（弱，用户可绕过）：隐藏不该看到的按钮/链接
2. **接口防护**（中等）：检查用户是否有权访问该资源
3. **数据层防护**（必须）：SQL 查询始终加 `WHERE creator_id = ?`

```go
// ❌ 不安全：直接用前端传的 id 查询
func GetOrder(orderID uint64) *Order {
    return db.First(&Order{}, orderID)
}

// ✅ 安全：加数据范围检查
func GetOrder(userID uint64, orderID uint64) (*Order, error) {
    var order Order
    err := db.Where("id = ? AND (creator_id = ? OR ? IN (SELECT role_id FROM user_roles WHERE user_id = ? AND role_id IN (SELECT id FROM roles WHERE scope = 'all')))",
        orderID, userID, userID, userID,
    ).First(&order).Error
    return &order, err
}
```
