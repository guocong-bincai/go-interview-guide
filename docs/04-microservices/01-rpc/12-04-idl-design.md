# API 设计规范与 Protobuf 版本兼容

> 考察频率：★★★☆☆  难度：★★★☆☆

## API 设计原则

### 自顶向下 vs 自底向上

- **自顶向下**：先设计 API proto，再生成代码（推荐，符合契约优先）
- **自底向上**：先写代码，再生成 proto（代码入侵，不推荐）

### Proto 设计规范

```protobuf
syntax = "proto3";

package usercenter;

option go_package = "github.com/company/usercenter/pb";

// 服务名：业务域 + 功能
// 方法名：动词 + 名词，符合 CRUD 语义
service UserService {
    // 批量接口：stream 或 Batch 前缀
    rpc BatchGetUser(BatchGetUserRequest) returns (BatchGetUserResponse);
    // 搜索接口
    rpc SearchUser(SearchUserRequest) returns (SearchUserResponse);
}

// 请求/响应：用 CamelCase，结尾不带 Request/Response 也可接受
message User {
    int64 id = 1;          // 字段从 1 开始，1-15 用 1 字节编码
    string name = 2;
    int32 age = 3;
    Gender gender = 4;
    map<string, string> extra = 5;  // 扩展字段用 map
    repeated string roles = 6;     // 数组用 repeated
}

// 枚举：必须从 0 开始，0 应该是 UNKNOWN 或 NOT_SET
enum Gender {
    GENDER_UNSPECIFIED = 0;
    GENDER_MALE = 1;
    GENDER_FEMALE = 2;
    GENDER_OTHER = 3;
}

// 时间类型：不要用 int64 表示时间，用 google.protobuf.Timestamp
import "google/protobuf/timestamp.proto";
message CreateOrderRequest {
    int64 user_id = 1;
    google.protobuf.Timestamp create_time = 2;
}
```

---

## 字段编号（Field Number）规则

### 编码效率

| 范围 | 字段类型 | 字节数 |
|------|---------|--------|
| 1-15 | 基本类型 | 1 字节 |
| 16-2047 | 基本类型 | 2 字节 |

### 编号原则

```
保留编号：
- 旧字段删除后，编号永远不用（客户端可能还有缓存）
- 可以在注释里标记 # deprecated，但编号不变

扩展编号（Proto3 不支持扩展，但可以用 reserved）：
message User {
    reserved 5, 10 to 15;  // 保留这些编号，不使用
    int64 id = 1;
    string name = 2;
}
```

---

## 向后兼容（Backward Compatibility）

Proto 的核心价值：**新增字段向后兼容，删除字段向前兼容**。

### ✅ 可以做（兼容变更）

```protobuf
// 原来
message User {
    int64 id = 1;
    string name = 2;
}

// 新增字段（客户端旧版本会忽略）
message User {
    int64 id = 1;
    string name = 2;
    string email = 3;        // ✅ 新增字段
    int32 age = 4;           // ✅ 新增字段
    repeated string tags = 5; // ✅ 新增数组
}

// 注意：新增字段要分配新的编号，不能复用旧编号
```

### ❌ 不能做（破坏性变更）

```protobuf
// ❌ 绝对禁止：修改已有字段的编号
message User {
    int64 id = 2;  // 原来是 1，改编号会乱码
    string name = 1;
}

// ❌ 绝对禁止：修改已有字段的类型
message User {
    int64 id = 1;   // 原来是 string
    string name = 2;
}

// ❌ 绝对禁止：直接删除字段（应该 deprecated）
// 正确做法：标记 reserved，但保留编号
message User {
    reserved 3;  // 保留编号，不使用
    int64 id = 1;
    string name = 2;
    // int32 age = 3;  // 删除了，保留编号
}
```

---

## 命名规范

### Package 命名

```
com.{公司}.{业务域}.{版本}
```

```protobuf
package usercenter.v1;

// 推荐
com.douyin.usercenter.v1
com.meituan.eats.order.v1
```

### Message 命名

- PascalCase：`CreateOrderRequest`、`UserResponse`
- 请求消息：动词 + Object + Request
- 响应消息：Object + Response 或直接用 Object（单条）
- 枚举：PascalCase，值用 SCREAMING_SNAKE_CASE

```protobuf
message GetUserRequest { ... }
message GetUserResponse { User user = 1; }
message ListUsersRequest { int32 page = 1; }
message ListUsersResponse { repeated User users = 1; }

enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_PAID = 2;
    ORDER_STATUS_COMPLETED = 3;
}
```

### RPC 方法命名

- 符合 HTTP 语义：GET = Query，POST = Create，PUT = Update，DELETE = Delete
- 不要暴露实现细节（不要叫 `GetUserFromDB`）

```protobuf
// ✅ 推荐
rpc GetUser(GetUserRequest) returns (User);
rpc CreateUser(CreateUserRequest) returns (User);
rpc UpdateUser(UpdateUserRequest) returns (User);
rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);

// ✅ 搜索类用 Search
rpc SearchUser(SearchUserRequest) returns (SearchUserResponse);

// ✅ 批量操作
rpc BatchGetUser(BatchGetUserRequest) returns (BatchGetUserResponse);
```

---

## 错误处理

### 标准错误模型

```protobuf
import "google/rpc/status.proto";

// 统一的错误响应
message ErrorResponse {
    int32 code = 1;                    // 业务错误码
    string message = 2;                // 错误信息
    string request_id = 3;             // 请求追踪 ID
    repeated ErrorDetail details = 4;  // 详细错误信息
}

message ErrorDetail {
    string field = 1;      // 出错字段
    string reason = 2;     // 出错原因
    string message = 3;    // 详细描述
}

// 推荐：不用 google.rpc.Status，因为它包含 protobuf.Any（不利于追踪）
```

### 错误码规范

```go
// 统一的业务错误码
const (
    ErrCodeOK           = 0
    ErrCodeParam        = 40001  // 参数错误
    ErrCodeUnauthorized = 40101  // 未认证
    ErrCodeForbidden    = 40301  // 无权限
    ErrCodeNotFound     = 40401  // 资源不存在
    ErrCodeConflict     = 40901  // 资源冲突
    ErrCodeInternal     = 50001  // 内部错误
)
```

---

## 版本管理

### URL 路径版本 vs Header 版本

```protobuf
// ✅ 推荐：URL 路径版本
service UserServiceV1 {
    rpc GetUser(GetUserRequest) returns (User) {
        option (google.api.http) = "/v1/users/{id}";
    }
}

service UserServiceV2 {
    rpc GetUser(GetUserRequest) returns (User) {
        option (google.api.http) = "/v2/users/{id}";
    }
}

// ❌ 不推荐：Header 版本（不直观，调试困难）
// X-Api-Version: v2
```

### Breaking Change 判断

| 变更类型 | Breaking? |
|---------|----------|
| 新增字段 | ❌ 否 |
| 新增 RPC | ❌ 否 |
| 删除字段 | ✅ 是 |
| 修改字段类型 | ✅ 是 |
| 修改字段编号 | ✅ 是 |
| 修改 RPC 语义 | ✅ 是 |
| 新增 required 字段 | ✅ 是 |

---

## 实践建议

### Proto 目录结构

```
proto/
├── user/
│   ├── user.proto           # 主 proto
│   ├── user.pb.go           # 生成（gitignore）
│   ├── user_grpc.pb.go      # 生成（gitignore）
│   └── user.proto.gz        # 压缩后的 proto（用于 protofetch）
├── order/
│   └── order.proto
└── common/
    ├── error.proto          # 通用错误定义
    └── pagination.proto     # 分页模型（所有 list 接口复用）
```

### Oneof 的使用

```protobuf
// 适合互斥字段，减少网络传输
message UpdateUserRequest {
    oneof update_field {
        string new_name = 1;
        int32 new_age = 2;
        Gender new_gender = 3;
    }
}
```

### Map 的限制

- key 只能是基本类型（string、int 系列）
- 不支持 `repeated map`

```protobuf
// ✅ 合法
map<string, string> extra = 1;
map<int64, User> users = 2;

// ❌ 非法
map<string, User> user_map = 1;  // value 不能是 message
map<string, repeated string> tags = 2;  // 不能 repeated map
```

---

## 面试话术

**Q：Protobuf 和 JSON 的区别，为什么 gRPC 用 Protobuf？**

> 核心区别是编码方式和效率。Protobuf 是二进制编码，字段用编号而非字段名传输，1、2 这样的小数字只占 1 字节，所以体积小、解析快（直接内存映射，不需要字符串解析）。JSON 是文本编码，可读性好但体积大、解析慢。gRPC 选择 Protobuf 是因为它要承载高并发 RPC 通信，性能差距明显。另外 Protobuf 有强类型和代码生成，IDL 本身是契约，比 JSON + 手写类型更可靠。

**Q：如何平滑升级 Protobuf，不影响线上服务？**

> 核心原则是"新增可以，修改删除不行"。新增字段时分配新编号，客户端旧版本会忽略未知字段（Proto3 默认行为）。删除字段时保留编号（`reserved`），防止未来复用。如果必须删除，先让所有客户端升级，再删字段。版本切换时新老接口并行一段时间，用 feature flag 控制流量。
