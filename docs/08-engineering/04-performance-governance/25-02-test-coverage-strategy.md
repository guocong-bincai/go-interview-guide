# Go 测试覆盖率提升：从 40% 到 95% 的实战方法

> 考察频率：★★★★☆  优先级：P1
> 关键词：覆盖率工具、桩代码(mock)、Table-Driven Test、覆盖率盲区、白盒 vs 黑盒

## 面试官考察意图

很多团队盲目追求"覆盖率数字"，高级工程师需要回答的是：**什么样的覆盖率才算够？怎么有效提升覆盖率而不是刷数字？**

面试官想看到的是候选人对测试质量的理解深度——不只是会用 `go test -cover`，而是知道**覆盖率的局限性、怎么测边界条件、怎么 mock 外部依赖**。

---

## 核心答案（30 秒版）

覆盖率只是一个指标，**真正重要的是测试覆盖了关键路径和异常分支**。提升覆盖率的方法：优先用 Table-Driven Test 覆盖边界条件，用 gomock/mockery 隔离外部依赖，重点覆盖 nil/空值/超时/并发等异常路径。**覆盖率低于 70% 说明有严重盲区，超过 95% 但核心业务没测到等于无效——质量比数字重要。**

---

## 覆盖率现状诊断

### 常见误区

```go
// ❌ 只测正常路径，覆盖率 60%，但漏了 80% 的异常场景
func ProcessOrder(order Order) error {
    if order.Amount <= 0 {
        return ErrInvalidAmount  // 这行可能没被覆盖！
    }
    return saveToDB(order)  // 假设这行被覆盖
}

// ✅ Table-Driven Test 覆盖所有分支
func TestProcessOrder(t *testing.T) {
    tests := []struct{
        name    string
        amount  int64
        wantErr bool
    }{
        {"正常支付", 100, false},
        {"金额为零", 0, true},
        {"负金额", -50, true},
        {"超大金额", math.MaxInt64, false},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ProcessOrder(Order{Amount: tt.amount})
            if (err != nil) != tt.wantErr {
                t.Errorf("ProcessOrder() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### 覆盖率分析三步法

```bash
# 第 1 步：生成 HTML 覆盖率报告
go test -coverprofile=coverage.out ./pkg/...
go tool cover -html=coverage.out -o coverage.html

# 第 2 步：按包统计覆盖率
go tool cover -func=coverage.out

# 第 3 步：识别未覆盖的关键代码
go tool cover -func=coverage.out | grep -E "(0\.|0\.0)"
```

**关键问题：** 未覆盖的代码是故意没测（如 init()、错误处理兜底），还是遗漏？后者需要补。

---

## Table-Driven Test 最佳实践

Go 最经典的测试模式，一次定义多组用例：

```go
func TestParseHeader(t *testing.T) {
    tests := []struct {
        name     string
        input    string
        wantKey  string
        wantVal  string
        wantErr  bool
        errMatch string
    }{
        {
            "正常键值对",
            "Authorization: Bearer token123",
            "authorization", "Bearer token123", false, "",
        },
        {
            "缺少冒号",
            "NoColonValue",
            "", "", true, "invalid format",
        },
        {
            "空字符串",
            "",
            "", "", true, "empty header",
        },
        {
            "多个冒号",
            "X-Custom: value:with:colons",
            "x-custom", "value:with:colons", false, "",
        },
        {
            "纯空格键",
            "   : value",
            "", "", true, "empty key",
        },
        {
            "Unicode 键",
            "中文头部: 测试值",
            "中文头部", "测试值", false, "",
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            key, val, err := ParseHeader(tc.input)
            if tc.wantErr {
                require.Error(t, err)
                if tc.errMatch != "" {
                    assert.Contains(t, err.Error(), tc.errMatch)
                }
            } else {
                require.NoError(t, err)
                assert.Equal(t, tc.wantKey, strings.ToLower(key))
                assert.Equal(t, tc.wantVal, val)
            }
        })
    }
}
```

**Table-Driven Test 的价值：** 一行定义覆盖所有分支，新增 case 不改逻辑。

---

## Mock 外部依赖的正确姿势

### 方案对比

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| gomock + mockery | 大型项目、接口复杂 | 自动生成、IDE 友好 | 需维护 interface |
| 手写 stub | 简单函数 | 零依赖 | 样板代码多 |
| testcontainers | DB/Redis 集成测试 | 真实环境 | 慢，用于集成测试 |
| wiremock | HTTP 外部 API | JSON 配置灵活 | 额外服务 |

### gomock + mockery 实战

```bash
# 安装
go install github.com/vektra/mockery/v2@latest

# 为 interface 生成 mock
mockery --name=UserRepo --output=./mocks --outpkg=mocks
```

```go
// user_repo.go — 生产代码中的接口
type UserRepo interface {
    GetByID(ctx context.Context, id int64) (*User, error)
    Save(ctx context.Context, u *User) error
}

// mocks/user_repo.go — 自动生成的 mock
type MockUserRepo struct {
    mock.Mock
}

func (m *MockUserRepo) GetByID(ctx context.Context, id int64) (*User, error) {
    args := m.Called(ctx, id)
    return args.Get(0).(*User), args.Error(1)
}

// 使用 mock 的测试
func TestUserService_GetProfile(t *testing.T) {
    ctrl :=gomock.NewController(t)
    defer ctrl.Finish()

    mockRepo := mocks.NewMockUserRepo(ctrl)
    
    // 期望：调用 GetByID 返回特定用户
    expectedUser := &User{ID: 1, Name: "Alice"}
    mockRepo.EXPECT().GetByID(gomock.Any(), int64(1)).Return(expectedUser, nil)

    svc := NewService(mockRepo)
    profile, err := svc.GetProfile(context.Background(), 1)
    
    assert.NoError(t, err)
    assert.Equal(t, "Alice", profile.Name)
}
```

---

## 覆盖率提升实战清单

### P0：必须覆盖的核心场景

```go
// 检查清单（打勾表示已覆盖）
☐ nil 输入处理
☐ 空切片/空 map 边界
☐ 零值参数处理
☐ 超时/cancel 场景
☐ 并发安全验证（race detector）
☐ 错误类型断言（errors.Is/As）
☐ 资源清理（defer/close）
```

### P1：高价值补充场景

```go
// 检查清单
☐ 大输入性能验证（内存不会 OOM）
☐ 极端数值（MaxInt64, floatNaN 等）
☐ Unicode / Emoji 编码边界
☐ 网络延迟模拟（wiremock / httptest）
☐ 数据库连接失败回退
☐ Redis 不可用降级
☐ 配置文件缺失/格式错误的 graceful failover
```

### P2：锦上添花

```go
☐ 不同 Go 版本兼容性
☐ 32-bit 平台验证
☐ 低内存环境行为
```

---

## 覆盖率指标解读

### 什么算健康？

| 覆盖率范围 | 评价 | 建议 |
|-----------|------|------|
| < 40% | 🚨 危险 | 存在大量未测试代码，必须有计划提升 |
| 40~60% | ⚠️ 一般 | 核心路径基本覆盖，但边缘场景不足 |
| 60~80% | ✅ 达标 | 大部分业务逻辑有保护 |
| 80~90% | 🔥 优秀 | 接近生产级质量水平 |
| > 95% | 📊 注意 | 可能过度测试边缘场景，关注 ROI |

**关键原则：** CI 中设置最低覆盖率门槛，比如要求 PR 的覆盖率不低于 75% 且不能相比主分支下降。

```yaml
# .gitlab-ci.yml — 覆盖率门禁示例
quality-gate:
  stage: test
  script:
    - go test -covermode=count -coverprofile=cov.out ./...
    - COVERAGE=$(go tool cover -func=cov.out | grep total | awk '{print substr($3, 1, length($3)-1)}')
    - echo "Coverage: $COVERAGE%"
    - if [ "$(echo "$COVERAGE < 75" | bc)" -eq 1 ]; then
        echo "Coverage below threshold!"; exit 1;
      fi
  allow_failure: false
```

---

## 面试话术

> "我通常把测试分成三层：单测（覆盖函数逻辑）、集成测试（覆盖模块交互）、E2E（覆盖核心流程）。单测我用 Table-Driven Test 为主，配合 gomock 隔离外部依赖。我们的生产项目单测覆盖率在 82% 左右，不是最高但也足够——因为我们额外做了全链路压测和混沌工程来兜底。我的经验是**与其追求 100% 覆盖率，不如确保每个核心业务流程都有端到端的自动化保障**。"

---

[🏠 首页](../../../README.md) · [📦 工程素养](../README.md) · [🛡️ 技术领导力](./README.md)
