# Docker 容器化

> 考察频率：★★★☆☆  难度：★★★☆☆

## 核心问题

Docker 考察重点：
1. **多阶段构建**：减小镜像体积
2. **镜像优化**：层缓存、构建速度
3. **资源限制**：CPU、内存、pidcgroup
4. **安全加固**：非 root、只读根文件系统

---

## 1. Go 项目 Dockerfile 最佳实践

### 多阶段构建

```dockerfile
# ====== 阶段1：构建 ======
FROM golang:1.22-alpine AS builder

# 安装编译依赖
RUN apk add --no-cache gcc musl-dev

WORKDIR /app

# 复制依赖文件（利用缓存）
COPY go.mod go.sum ./
RUN go mod download

# 复制源码
COPY . .

# 编译 - 静态链接
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags='-w -s' \      # 去除调试信息
    -o /app/server .

# ====== 阶段2：运行 ======
FROM alpine:3.19

# 安全：创建非root用户
RUN addgroup -g 1000 appgroup && \
    adduser -u 1000 -G appgroup -s /bin/sh -D appuser

WORKDIR /app

# 只复制二进制
COPY --from=builder /app/server .

# 安全加固
RUN chown -R appuser:appgroup /app

USER appuser

# 只读文件系统 + 无特权端口
EXPOSE 8080

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1

ENTRYPOINT ["./server"]
```

### 为什么要多阶段构建？

| 方式 | 镜像大小 | 问题 |
|------|---------|------|
| 单一阶段 | 800MB+ | 包含编译器、构建工具、源码 |
| 多阶段 | 15-30MB | 只包含运行时和二进制 |

### 静态链接 vs 动态链接

```dockerfile
# 动态链接（需要libc）
FROM golang:1.22
RUN go build -o server .

# 镜像：~800MB

# 静态链接（不需要libc）
FROM golang:1.22
ENV CGO_ENABLED=0
RUN go build -o server .

# 镜像：~15MB（alpine）
```

---

## 2. 镜像优化技巧

### 利用 BuildKit 缓存

```dockerfile
# syntax=dockerfile.com/docker/dockerfile:1.4

# 先复制依赖再复制代码，利用Docker层缓存
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o server .
```

### 最小化层数

```dockerfile
# ❌ 不好：多层复制
COPY go.mod .
RUN go mod download
COPY go.sum .
COPY . .

# ✅ 好：一次复制
COPY go.mod go.sum ./
RUN go mod download
COPY . .
```

### 使用 .dockerignore

```gitignore
# .dockerignore
.git
.gitignore
*.md
docs/
test/
*_test.go
Makefile
Dockerfile
.env
```

---

## 3. 资源限制

### docker run 资源限制

```bash
# 内存限制
docker run -m 512m --memory-reservation=256m \
    --oom-kill-disable \    # 防止OOM被杀
    myapp

# CPU限制
docker run --cpus=1.5 \    # 最多1.5核
    --cpu-period=100000 \
    --cpu-quota=150000 \
    myapp

# PID限制（防止fork炸弹）
docker run --pids-limit=100 myapp
```

### docker-compose 配置

```yaml
version: "3.8"
services:
  api:
    build: .
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
    restart: unless-stopped
    read_only: true          # 只读文件系统
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
```

---

## 4. 安全加固

### 安全 checklist

```dockerfile
# 1. 非root用户
RUN adduser -u 1000 -D appuser
USER appuser

# 2. 只读根文件系统
docker run --read-only

# 3. 无特权端口
# 应用程序用 >1024 端口，通过 docker run -p 映射到 80/443

# 4. 禁用特权模式
docker run --privileged=false

# 5. 限制 Capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE

# 6. Seccomp 限制系统调用
docker run --security-opt seccomp=profile.json
```

### Go 的安全编译

```dockerfile
# 去除符号表、禁用CGO
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags='
        -w -s
        -extldflags="-static"
    ' -o server .
```

---

## 5. 健康检查与探针

### Docker HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

### K8s 探针（延伸）

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 15
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## 6. 面试话术

**Q：Docker 镜像怎么优化？**

> 核心是多阶段构建 + 静态编译。Go 只需要编译好的二进制，不需要运行时环境。我用 alpine 基础镜像 + CGO_ENABLED=0 静态编译，最终镜像可以压到 15MB 左右。另外利用 BuildKit 的层缓存，先复制 go.mod 下载依赖，再复制源码。

**Q：容器里跑 Go 要注意什么？**

> 1）Go 运行时默认没有容器感知，需要设置 `GOMAXPROCS` 和内存限制；2）注意 CPU limit 对调度的影响；3）Go 的 GC 和容器内存限制配合要做好，用 `GOMEMLIMIT` 环境变量；4）健康检查接口要单独实现，不能用 pprof 端口。

**Q：如何防止容器内恶意行为？**

> 遵循最小权限原则：用非 root 用户运行、只读文件系统、禁用 privileged、drop 所有 Linux capabilities。另外配合 seccomp 限制系统调用，用 AppArmor/SELinux 做进程隔离。

---

## 总结

| 优化项 | 效果 |
|-------|------|
| 多阶段构建 | 800MB → 15MB |
| alpine 基础镜像 | 减少 200MB+ |
| 静态编译 CGO_ENABLED=0 | 减少 libc 依赖 |
| 层缓存优化 | 构建速度提升 |
| 非 root 用户 | 安全性提升 |
