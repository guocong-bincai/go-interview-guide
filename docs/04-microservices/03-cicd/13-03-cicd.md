# CI/CD 流水线

> 考察频率：★★★☆☆  难度：★★★☆☆

## 核心问题

CI/CD 考察重点：
1. **流水线设计**：构建 → 测试 → 部署
2. **部署策略**：蓝绿、金丝雀、滚动发布
3. **回滚机制**：如何快速回退
4. **质量门禁**：自动化测试、代码扫描

---

## 1. Go 项目 CI/CD 流程

### GitHub Actions 示例

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  GO_VERSION: "1.22"

jobs:
  # ===== Stage 1: 代码检查 =====
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
      - name: golangci-lint
        run: |
          curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin
          golangci-lint run ./...

  # ===== Stage 2: 单元测试 =====
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
      - name: Test
        run: |
          go mod download
          go test -race -coverprofile=coverage.out ./...
      - name: Upload Coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage.out

  # ===== Stage 3: 构建镜像 =====
  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ===== Stage 4: 部署 =====
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to K8s
        uses: azure/k8s-deploy@v4
        with:
          manifests: |
            k8s/manifests/
          images: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### GitLab CI 示例

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  GO_VERSION: "1.22"
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

go-test:
  stage: test
  image: golang:$GO_VERSION
  script:
    - go mod download
    - go test -race -coverprofile=coverage.out ./...
  coverage: '/total:.*\s(\d+\.\d+)%/'
  artifacts:
    reports:
      coverage_report: coverage.out

docker-build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE .
    - docker push $IMAGE
  only:
    - main

k8s-deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp app=$IMAGE
    - kubectl rollout status deployment/myapp
  environment:
    name: production
  only:
    - main
```

---

## 2. 部署策略

### 滚动发布（Rolling Update）

```yaml
# K8s 滚动发布
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # 最多超出25%的Pod
      maxUnavailable: 25%  # 最多25%不可用
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
```

**特点**：
- 逐步替换，旧 → 新
- 对用户体验影响小
- 回滚需要反向操作

### 蓝绿部署（Blue-Green）

```
                    ┌─────────────┐
    流量 ──────────▶│   绿色(新)   │  (当前生产)
                    └─────────────┘

    切换后:
    流量 ──────────▶│   蓝色(新)   │  (当前生产)
                    └─────────────┘
                    │   绿色(旧)   │  (待销毁)
                    └─────────────┘
```

```yaml
# 蓝绿部署通过 Service Selector 切换
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue    # 切换 version: blue/green 即可
  ports:
    - port: 80
      targetPort: 8080
```

**回滚**：修改 Service Selector 瞬间回切。

### 金丝雀发布（Canary）

```yaml
# Argo Rollouts 金丝雀策略
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5      # 先导 5% 流量到新版本
        - pause: {duration: 10m}
        - setWeight: 20     # 20%
        - pause: {duration: 10m}
        - setWeight: 50     # 50%
        - pause: {}
      analysis:
        templates:
          - templateName: success-rate
        args:
          - name: service-name
            value: myapp-canary
```

**核心思想**：先放小量流量验证，有问题快速回滚。

### 部署策略对比

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 滚动发布 | 节省资源 | 回滚慢、版本共存 | 常规迭代 |
| 蓝绿部署 | 回滚秒级 | 双倍资源 | 重大版本 |
| 金丝雀 | 风险可控 | 复杂度高 | 高风险功能 |

---

## 3. 回滚机制

### 数据库回滚

```sql
-- Flyway / Liquibase 版本化管理
-- V1__init_schema.sql
-- V2__add_column.sql
-- V3__modify_index.sql

-- 回滚：执行 down 脚本
-- V2__add_column__undo.sql
```

### 应用回滚

```bash
# K8s 回滚
kubectl rollout undo deployment/myapp              # 回滚到上一版本
kubectl rollout undo deployment/myapp --to-revision=3  # 回滚到指定版本

# 查看历史
kubectl rollout history deployment/myapp
```

### 配置回滚

```yaml
# 配置中心 + 版本快照
# Apollo / Nacos 支持配置回滚
```

---

## 4. 质量门禁

### 安全扫描

```yaml
# Trivy 镜像扫描
trivy image --severity HIGH,CRITICAL myapp:latest

# 集成到 CI
- name: Trivy Security Scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:latest
    format: sarif
    output: trivy-results.sarif
```

### 代码覆盖率门禁

```bash
# 覆盖率必须 > 70%
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | grep total
# 如果 total < 70%，CI 失败
```

---

## 5. 面试话术

**Q：蓝绿部署和滚动发布各什么时候用？**

> 蓝绿适合重大版本发布，能秒级回滚，但需要双倍资源。滚动发布资源利用率高，但回滚慢。金丝雀适合高风险功能，先放小流量验证。我一般常规迭代用滚动，核心链路重大改动用蓝绿+金丝雀结合。

**Q：数据库变更怎么做不停机发布？**

> 核心原则：先加后删。1）加新字段（新加 nullable 或有默认值）；2）上线新代码，同时兼容新旧字段；3）旧数据迁移（后台任务）；4）旧代码下线和旧字段删除。遵循"扩展-迁移-收缩"三阶段。

**Q：CI/CD 怎么保证构建速度？**

> 1）依赖缓存（go mod download 单独一层）；2）Docker layer 缓存利用 BuildKit；3）测试并行（-parallel）；4）跳过不必要的 stage（比如只有 markdown 改了就跳过编译）；5）用 pnpm/npm workspaces 加速前端依赖。

---

## 总结

| 阶段 | 工具 |
|------|------|
| 代码检查 | golangci-lint, Trivy |
| 单元测试 | go test |
| 镜像构建 | Docker, BuildKit |
| 镜像扫描 | Trivy, Grype |
| 部署 | ArgoCD, Flux, Jenkins X |
| 配置管理 | Helm, Kustomize |
