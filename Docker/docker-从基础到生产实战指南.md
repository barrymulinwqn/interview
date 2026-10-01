# Docker 从基础到生产实战指南

本文面向希望从“会运行容器”进阶到“能为真实产品设计、交付和运维容器平台”的工程师。重点是理解原理、做出正确的架构边界判断，并能在故障发生时有效定位问题，而不只是记忆命令。

> 适用范围：Docker Engine、Docker Compose、OCI 镜像和 Kubernetes 等容器编排平台。命令以 Linux Docker Engine 为主；macOS 或 Windows 的 Docker Desktop 通过 Linux 虚拟机运行容器，文件挂载和网络行为会有少量差异。

## 1. 从基础到进阶的学习路线

| 阶段 | 要掌握的能力 | 验证结果 |
| --- | --- | --- |
| 基础 | 镜像、容器、端口、日志、卷、网络 | 能运行单服务并正确保留数据 |
| 构建 | Dockerfile、缓存、多阶段构建、`.dockerignore` | 能构建小、可复现、非 root 的镜像 |
| 编排 | Compose、服务发现、健康检查、配置分层 | 能本地启动 API、数据库、缓存和 Worker |
| 交付 | CI、Registry、漏洞扫描、SBOM、签名 | 能将同一镜像发布到测试和生产 |
| 运维 | 安全、资源限制、监控、日志、回滚、备份 | 能定位发布失败、OOM 和网络故障 |
| 平台化 | Kubernetes、Pod、自动扩缩、策略治理 | 能解释 Docker、container 与 Pod 的边界 |

贯穿所有阶段的原则是：**镜像不可变、容器无状态、配置外置、数据持久化、发布可回滚、运行可观测。**

---

## 2. Docker 基础和核心原理

### 2.1 Docker 解决的问题

传统部署常出现“开发环境可运行，测试或生产环境不可运行”。根因通常是语言版本、系统库、环境变量、启动命令、依赖服务或部署步骤不一致。Docker 将应用与运行时依赖封装成镜像，使经过验证的同一份产物可以在不同环境运行。

Docker **不是虚拟机**。容器本质上是宿主机内核中被隔离和限制的一组进程。它通常比 VM 启动更快、资源开销更小，但也共享宿主机内核，所以运行时权限、镜像来源和主机安全必须被认真对待。

| 对比项 | 容器 | 虚拟机 |
| --- | --- | --- |
| 隔离层 | Namespace、cgroup、文件系统隔离 | Hypervisor 和完整客户机 OS |
| 内核 | 通常共享宿主机内核 | 每个 VM 有自己的内核 |
| 启动和体积 | 通常秒级，镜像多为 MB 到数百 MB | 通常分钟级，镜像常为 GB 级 |
| 适用场景 | 应用服务、任务、CI、标准化交付 | 强隔离、不同内核、遗留 OS、基础设施虚拟化 |

### 2.2 核心对象

| 对象 | 定义 | 关键理解 |
| --- | --- | --- |
| Docker Client | `docker` CLI 或 API 客户端 | 将请求发送给 Docker Engine，不直接运行容器 |
| Docker Engine / `dockerd` | 管理镜像、容器、卷和网络的守护进程 | Linux 上拥有高权限；加入 `docker` 组接近拥有 root 权限 |
| containerd / runc | 低层容器生命周期管理和 OCI 运行时 | Docker 通过它们创建实际隔离的 Linux 进程 |
| Image（镜像） | 只读、分层、可分发的运行模板 | 应固定版本或 digest，生产避免 `latest` |
| Container（容器） | 镜像的一次可运行实例 | 可删除重建；写入层数据默认不持久 |
| Registry（仓库） | 存储和分发镜像的服务 | Docker Hub、Harbor、ECR、ACR 等 |
| Volume（卷） | Docker 管理的持久化存储 | 保存需要跨容器生命周期保留的数据 |
| Network（网络） | 容器通信和名称解析边界 | 应把数据库、缓存等放入私有网络 |

运行链路如下：

```text
docker CLI / CI
      |
      | Docker API
      v
dockerd --> containerd --> runc --> Linux namespaces + cgroups + filesystem
      |                 |
      |                 +--> running container process
      +--> image / volume / network management
```

### 2.3 容器隔离为何成立

| 内核能力 | 作用 | 生产含义 |
| --- | --- | --- |
| Namespace | 隔离 PID、网络、挂载点、IPC、主机名、用户等视图 | 容器内看到的进程、网卡不等于宿主机全貌 |
| cgroup | 限制并统计 CPU、内存、I/O 和进程数 | 不设限制时，一个容器可能挤占整台主机 |
| Union filesystem / CoW | 叠加镜像只读层与容器可写层 | 在运行中修改容器文件不是可复现发布方式 |
| Capability | 将 root 权限拆成更细的能力 | 应默认丢弃不需要的能力 |
| Seccomp / AppArmor / SELinux | 限制系统调用或文件访问策略 | 是运行时纵深防御的一部分 |

### 2.4 镜像层、标签和 digest

镜像由多个只读层构成；运行容器时 Docker 加上一层临时可写层。镜像可共享基础层，因此多个服务使用相同运行时不会重复占用全部空间。

```text
base OS layer             <- 多个镜像可共享
runtime layer             <- 例如 Python / Node.js
application dependencies
application source
---------------------
container writable layer <- 删除容器即丢失
```

- **标签（tag）**：易读但可变，例如 `orders-api:2026.09.30`。同一 tag 理论上可以被重新推送到不同镜像。
- **摘要（digest）**：内容哈希形式的不可变引用，例如 `registry.example.com/orders-api@sha256:...`。
- **生产实践**：CI 生成唯一的版本 tag，部署记录并最好使用 digest。tag 便于追踪，digest 保证实际运行的二进制完全一致。

---

## 3. 首次运行容器与生命周期

### 3.1 最小示例

```bash
# 前台运行，结束后自动删除，适合快速验证
docker run --rm hello-world

# 后台运行 nginx，将主机 8080 映射到容器 80
docker run -d --name web -p 8080:80 nginx:1.27-alpine

# 查看容器、日志和运行状态
docker ps
docker logs --tail 100 web
docker inspect web

# 停止并删除临时服务
docker stop web
docker rm web
```

`-p 8080:80` 的格式是 `主机端口:容器端口`。访问 `http://localhost:8080` 时，流量被转发到容器内监听 `80` 的进程。`EXPOSE 80` 只是镜像元数据，**不会**自动将端口发布到宿主机。

### 3.2 生命周期

```text
created -> running -> paused -> running -> exited -> removed
              |                     ^
              +------ restart ------+
```

容器内主进程（PID 1）退出，容器也会退出。常见错误是启动后台服务后 shell 立即结束，造成容器停止。容器应以前台方式运行其主服务，例如 `nginx -g 'daemon off;'` 或 `python -m uvicorn ...`。

---

## 4. Dockerfile：构建可复现的镜像

### 4.1 指令说明

| 指令 | 作用 | 实践建议 |
| --- | --- | --- |
| `FROM` | 指定基础镜像 | 固定明确版本，必要时固定 digest |
| `WORKDIR` | 设置工作目录 | 避免依赖镜像默认目录 |
| `COPY` | 复制构建上下文文件 | 优先于 `ADD`，并配合 `.dockerignore` |
| `RUN` | 在构建期执行命令 | 安装依赖后同层清理缓存 |
| `ARG` | 构建期变量 | 不能传入秘密，构建历史可能暴露值 |
| `ENV` | 运行时默认环境变量 | 仅放非敏感默认配置 |
| `USER` | 设置运行用户 | 生产镜像避免 root 运行 |
| `EXPOSE` | 标注文档化端口 | 不会发布端口 |
| `ENTRYPOINT` | 固定入口可执行文件 | 用 JSON 数组形式保证信号传递 |
| `CMD` | 默认命令或入口参数 | 可被 `docker run image args` 覆盖 |
| `HEALTHCHECK` | 定义容器级健康检查 | 检查应用可用性，而非仅检查进程 |

### 4.2 生产级 Python API 示例

以名为 `orders-api` 的服务为例。多阶段构建把编译工具和构建过程留在 builder 阶段，最终镜像只携带运行所需文件。

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim AS builder

WORKDIR /build
ENV PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1

COPY requirements.txt .
RUN python -m venv /opt/venv \
    && /opt/venv/bin/pip install -r requirements.txt

FROM python:3.13-slim AS runtime

RUN groupadd --system app \
    && useradd --system --gid app --create-home app

WORKDIR /app
ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

COPY --from=builder /opt/venv /opt/venv
COPY --chown=app:app ./app ./app

USER app
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD python -c "from urllib.request import urlopen; urlopen('http://127.0.0.1:8080/healthz', timeout=2)" || exit 1

ENTRYPOINT ["python", "-m", "uvicorn"]
CMD ["app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

构建与运行验证：

```bash
docker build -t orders-api:dev .
docker run --rm -p 8080:8080 orders-api:dev
curl -f http://localhost:8080/healthz
```

### 4.3 缓存顺序为何重要

Docker 会复用命中的构建缓存。依赖清单通常比业务代码改动少，应先复制和安装依赖，再复制代码：

```dockerfile
# 推荐：requirements 不变时，依赖安装层能复用
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY ./app ./app
```

```dockerfile
# 不推荐：任意代码变动都会让依赖层缓存失效
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
```

构建缓存是效率工具而非正确性工具。基础镜像安全修复、依赖变更或构建异常时可以有选择地使用 `--no-cache`，但不应把它作为日常构建方式。

### 4.4 `.dockerignore` 是安全边界

```gitignore
.git
.env
.env.*
!.env.example
__pycache__/
*.py[cod]
.pytest_cache/
.venv/
node_modules/
dist/
coverage/
*.pem
*.key
```

没有 `.dockerignore` 时，构建上下文可能携带本地 `.git`、私钥、测试输出和依赖目录。即使后续没有 `COPY` 它们，也会拖慢构建，并扩大秘密泄露风险。

### 4.5 `ENTRYPOINT` 与 `CMD`

```dockerfile
ENTRYPOINT ["python", "-m", "uvicorn"]
CMD ["app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

`docker run orders-api:dev` 会拼接两者。`docker run orders-api:dev app.debug:app` 会保留 `ENTRYPOINT`，仅替换 `CMD`。JSON 数组形式不会经 shell 二次解析，Docker 能把 `SIGTERM` 直接交给实际服务进程，有利于优雅停止。

---

## 5. 存储、网络和配置

### 5.1 持久化存储

容器可写层适合临时文件，不适合数据库、用户上传文件和业务状态。容器一旦被替换或删除，可写层数据会丢失。

| 方式 | 数据位置 | 常见用途 | 注意事项 |
| --- | --- | --- | --- |
| Named volume | Docker 管理的宿主机目录 | 单机数据库、持久化开发数据 | 需要备份，不是跨主机高可用存储 |
| Bind mount | 显式宿主机路径 | 本地热更新、受控文件注入 | 与主机路径和权限强耦合 |
| tmpfs | 内存 | 短暂敏感文件、缓存 | 重启即消失，受内存限制影响 |
| 对象存储 / 托管数据库 | 外部系统 | 生产附件、业务数据库 | 是生产有状态服务的常见选择 |

```bash
# 创建和检查命名卷
docker volume create orders-postgres-data
docker volume inspect orders-postgres-data

# 以只读方式挂载公开配置
docker run --rm -v "$(pwd)/config.yaml:/app/config.yaml:ro" orders-api:dev

# 临时目录放在内存中
docker run --rm --tmpfs /tmp:rw,noexec,nosuid,size=64m orders-api:dev
```

### 5.2 网络和服务发现

Docker 的默认 bridge 网络适合试验。多服务应用应创建 user-defined bridge 网络，Docker 会为同一网络中的服务提供基于容器名的 DNS 解析。

```bash
docker network create orders-backend
docker run -d --name redis --network orders-backend redis:7.4-alpine
docker run --rm --network orders-backend redis:7.4-alpine \
  redis-cli -h redis ping
```

生产网络设计原则：

1. 只有 API 网关或反向代理映射公开端口；数据库、缓存、消息队列不应直接暴露到公网。
2. 服务间使用 DNS 名称而非固定容器 IP，容器重建后 IP 会变化。
3. 显式划分入口网络和内部网络；Compose 可用 `internal: true` 建立内部网络。
4. 网络可达不等于服务可用。客户端仍需设置连接超时、读取超时、重试、熔断和幂等性。

### 5.3 配置和秘密

配置会随环境变化，但镜像不应随环境重新构建。常见优先级为：运行参数 > 环境变量 > 挂载配置文件 > 镜像默认值。

| 类型 | 示例 | 正确处理方式 |
| --- | --- | --- |
| 非敏感配置 | 日志级别、功能开关、服务 URL | 环境变量、ConfigMap 或受控配置文件 |
| 秘密 | 数据库密码、OAuth secret、TLS 私钥 | Secret Manager / Vault、最小权限和轮换 |
| 构建期机密 | 私有依赖仓库 Token | BuildKit secret，绝不放入 `ARG` 或 `ENV` |

真实密码不应提交到 `.env`、Dockerfile、镜像层、标签、终端历史或应用日志。环境变量也可能被具备容器或主机诊断权限的用户读取；生产环境应优先使用短期凭据和工作负载身份。

---

## 6. Docker Compose：本地多服务产品

Compose 使用 YAML 声明多个容器的网络、卷、环境变量和启动关系。它非常适合本地开发、集成测试、单机演示和范围明确的单机产品；它不是多可用区高可用编排平台。

### 6.1 订单产品本地环境

以下例子包含 API、异步 Worker、PostgreSQL 和 Redis。API 与 Worker 使用**同一个镜像**，但使用不同启动命令，能保持构建与漏洞治理的一致性。

```yaml
name: orders

services:
  api:
    build:
      context: .
    image: orders-api:dev
    command: ["app.main:app", "--host", "0.0.0.0", "--port", "8080"]
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      APP_ENV: development
      DATABASE_URL: postgresql://orders:orders@postgres:5432/orders
      REDIS_URL: redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - edge
      - backend
    read_only: true
    tmpfs:
      - /tmp

  worker:
    image: orders-api:dev
    command: ["-m", "app.worker"]
    environment:
      APP_ENV: development
      DATABASE_URL: postgresql://orders:orders@postgres:5432/orders
      REDIS_URL: redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    read_only: true
    tmpfs:
      - /tmp

  postgres:
    image: postgres:17.2-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: orders
      POSTGRES_PASSWORD: orders
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orders -d orders"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s

  redis:
    image: redis:7.4-alpine
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  postgres-data:

networks:
  edge:
  backend:
    internal: true
```

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f api
curl -f http://127.0.0.1:8080/healthz
docker compose exec api sh
docker compose down             # 保留 volume 中的数据
docker compose down -v          # 同时删除 volume，数据不可恢复
```

这里将 API 绑定到 `127.0.0.1`，避免开发服务被局域网直接访问。`depends_on` 的健康条件只协调本次 Compose 启动顺序，不能替代应用自己的数据库连接重试、超时和故障恢复逻辑。

### 6.2 Compose 的边界

| 合适场景 | 不合适或需补足的能力 |
| --- | --- |
| 本地开发、CI 集成测试、故障复现 | 多主机调度、自动故障迁移、跨可用区高可用 |
| 单机内部工具、小型产品 | 复杂渐进发布、成熟 RBAC、全局网络策略 |
| 有明确备份恢复方案的单机服务 | 自动扩缩和多团队统一治理的大型平台 |

生产仍可使用 Compose，但必须明确主机补丁、反向代理、证书、备份、监控、容量、恢复演练和人工故障切换方案。若团队已有成熟 Kubernetes 平台，长期多服务产品通常更适合由 Kubernetes 管理。

---

## 7. 真实产品如何使用 Docker：订单与支付案例

### 7.1 需求和架构边界

假设产品为订单与支付 API：用户创建订单，系统写入数据库，异步 Worker 调用支付渠道并发送通知。它需要高可用、审计、可回滚、敏感数据保护和可观测性。

Docker 承担的是**一致的交付单元**：应用代码、语言运行时和非敏感依赖构建为 OCI 镜像。Docker 不应单独承担数据库高可用、秘密管理、全局流量治理或业务一致性；这些应由托管服务、编排平台和应用设计共同完成。

```text
Developer commit
      |
      v
CI: test -> build -> SBOM/scan -> sign -> push immutable image
      |
      v
Registry: orders-api:git-sha + sha256 digest
      |
      +-------------------> staging deployment and smoke test
      |
      v
Production Kubernetes / approved Compose host
      |
      +--> API replicas --> managed PostgreSQL
      +--> Worker replicas -> queue / Redis
      +--> logs, metrics, traces, alerts
```

### 7.2 责任划分

| 层次 | 应负责什么 | 不应负责什么 |
| --- | --- | --- |
| Dockerfile | 可复现运行环境、非 root 用户、基础健康命令 | 环境专属密码、生产域名、手工修复数据 |
| CI | 测试、构建、扫描、SBOM、签名、推送 | 在生产容器中手工修改文件 |
| Registry | 版本化镜像、保留策略、访问控制 | 用浮动标签当不可变发布记录 |
| 编排平台 | 副本、调度、服务发现、滚动发布、资源限制 | 替代业务幂等性和数据迁移策略 |
| 应用 | 超时、重试、幂等、健康端点、结构化日志 | 假定网络或下游永远可靠 |
| 数据平台 | 备份、加密、复制、恢复演练 | 依赖容器写入层保存业务数据 |

### 7.3 CI/CD 流水线

1. **静态检查和单元测试**：先验证代码，避免为明显失败的提交浪费构建和扫描资源。
2. **构建镜像**：使用 BuildKit；标记为 Git commit SHA 或发布版本，例如 `registry.example.com/orders-api:git-a1b2c3d`。
3. **供应链检查**：生成 SBOM，扫描 OS 与语言依赖漏洞，检查 Dockerfile 危险配置；对高危漏洞设置阻断策略和例外审批。
4. **签名并推送**：推送至私有 Registry，CI 身份只拥有所需最小权限；部署准入只接受可信来源镜像。
5. **测试环境部署**：使用与生产相同的镜像 digest，执行迁移演练、集成测试和 smoke test。
6. **生产渐进发布**：先小比例或小批节点发布，观测错误率、延迟、支付失败率和资源指标；异常时回滚。
7. **保留审计证据**：记录 commit、镜像 digest、扫描结果、配置版本、部署身份和时间。

```bash
export IMAGE="registry.example.com/payments/orders-api"
export VERSION="git-$(git rev-parse --short HEAD)"

docker buildx build \
  --platform linux/amd64 \
  --tag "$IMAGE:$VERSION" \
  --push \
  .

# 部署系统应记录并使用 push 后返回的 sha256 digest
```

多架构镜像只适合确实存在 ARM 与 x86 节点混合需求的场景。否则应针对实际运行节点架构构建，减少构建时间和排查复杂度。

### 7.4 生产运行时设计

#### 无状态 API

- API 容器不保存用户会话和业务文件；会话放入受控缓存或签名 token，附件放对象存储。
- 每个请求带 correlation ID，日志用 JSON 输出到 stdout/stderr，由平台统一采集。
- `GET /livez` 仅说明进程未卡死；`GET /readyz` 说明实例已加载配置并能承担流量。二者不能混为一谈。
- 外部调用必须设置连接、读取和总超时；仅对幂等操作做有上限、带退避和抖动的重试。

#### 异步 Worker

- Worker 与 API 独立部署、独立资源限制，可按队列长度扩缩容。
- 支付回调、通知发送和重试任务必须幂等；消息队列常提供“至少一次”投递，重复处理是正常情况。
- 长任务不能挤占 API 的 CPU 和内存配额，否则会直接影响用户请求延迟。

#### 数据和迁移

- 生产 PostgreSQL 优先选用托管数据库或专用高可用集群；数据库容器有 volume 不等于数据库高可用。
- schema 迁移由版本化 Job 或受控发布步骤执行，而不是由每个 API 副本启动时并发执行。
- 升级前验证备份，升级后验证恢复；只有“备份任务成功”日志不等于能恢复。

### 7.5 Kubernetes 部署示例

Docker 构建出的 OCI 镜像可被 Kubernetes 的 containerd、CRI-O 等运行时使用。以下清单展示生产 API 的关键控制点，镜像 digest 使用占位符：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: payments
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      serviceAccountName: orders-api
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: api
          image: registry.example.com/payments/orders-api@sha256:REPLACE_WITH_APPROVED_DIGEST
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - secretRef:
                name: orders-api-runtime
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          startupProbe:
            httpGet:
              path: /livez
              port: http
            failureThreshold: 30
            periodSeconds: 2
          livenessProbe:
            httpGet:
              path: /livez
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: orders-api
  namespace: payments
spec:
  selector:
    app: orders-api
  ports:
    - name: http
      port: 80
      targetPort: http
```

解释：

- `replicas: 3` 允许多个 API 实例分担流量；实际高可用还需要跨节点或跨可用区调度约束。
- `requests` 是调度器预留资源的依据；`limits` 是运行上限。内存超过 limit 往往会被 OOM kill，需要依赖监控和压测调优。
- `startupProbe` 给慢启动服务时间，避免 liveness 在初始化期间重启服务；readiness 失败时实例会从 Service 后端移除，不一定重启容器。
- `readOnlyRootFilesystem` 要求应用将临时文件写到挂载的 `/tmp` 或外部存储，这能尽早发现应用的隐性本地写入依赖。
- `secretRef` 仅展示注入机制。生产 Secret 应来自受控外部密钥系统，并限制服务账号的读取范围。

### 7.6 发布与回滚

容器回滚很快，但数据库迁移可能不可逆。因此发布必须把应用兼容性与 schema 迁移分开设计：

1. 先发布向后兼容的 schema 变更，例如新表或可空字段。
2. 发布能同时读写新旧结构的应用版本。
3. 数据回填和校验完成后，再移除旧字段或旧逻辑。
4. 保留前一个可工作的镜像 digest、部署清单和明确回滚条件。

不要把“回滚 Deployment”误认为“数据库自动回滚”。破坏性迁移前必须有经过演练的恢复方案。

---

## 8. Docker 常用命令与详细解释

执行清理类命令前，先使用 `ls`、`inspect`、`df` 或 `system df` 确认对象归属。尤其是 `prune` 和 `down -v`，可能删除仍有价值的缓存或数据卷。

### 8.1 环境和诊断

| 命令 | 作用 | 何时使用 |
| --- | --- | --- |
| `docker version` | 显示客户端和 Engine 版本 | 确认 CLI 能否与 daemon 通信 |
| `docker info` | 存储驱动、cgroup、镜像数等运行信息 | 排查宿主机运行时能力和资源 |
| `docker context ls` | 列出 Docker endpoint 上下文 | 使用远程 daemon 或 Desktop 时避免连错环境 |
| `docker system df` | 汇总镜像、容器、卷、构建缓存占用 | 磁盘异常增长时先定位 |
| `docker events` | 实时显示创建、重启、销毁、健康状态等事件 | 排查频繁重启或自动化行为 |

### 8.2 镜像命令

```bash
# 拉取确定版本；生产不要依赖 latest
docker pull nginx:1.27-alpine

# 从当前 Dockerfile 构建；-t 指定仓库名和标签
docker build -t orders-api:dev .

# 指定 Dockerfile 和构建参数；ARG 不可用于秘密
docker build -f docker/Dockerfile --build-arg APP_VERSION=1.4.0 -t orders-api:1.4.0 .

# 查看镜像、元数据和层历史
docker image ls
docker image inspect orders-api:dev
docker image history orders-api:dev

# 为同一镜像增加仓库标签并推送
docker tag orders-api:dev registry.example.com/payments/orders-api:1.4.0
docker push registry.example.com/payments/orders-api:1.4.0

# 删除无用镜像，先确认没有容器依赖它
docker image rm orders-api:dev
docker image prune
```

`docker buildx build` 是 BuildKit 构建前端，适合缓存导入导出、多平台镜像和更安全的构建期 secret。常规 `docker build` 足以开始，但生产 CI 通常应统一使用 BuildKit。

### 8.3 容器生命周期和交互

```bash
# 后台运行：名称、端口、环境、资源和重启策略
docker run -d \
  --name orders-api \
  --publish 8080:8080 \
  --env APP_ENV=staging \
  --memory 512m \
  --cpus 1 \
  --restart unless-stopped \
  orders-api:1.4.0

# 列出运行中或全部容器
docker container ls
docker container ls -a

# 有序停止会发送 SIGTERM，超时后才 SIGKILL
docker stop --time 30 orders-api
docker start orders-api
docker restart --time 30 orders-api

# 删除已停止容器；-f 强制停止后删除，排障时慎用
docker rm orders-api
docker rm -f orders-api

# 在运行容器中执行诊断；生产不要依赖手工改动
docker exec -it orders-api sh
docker exec orders-api printenv

# 复制文件仅用于诊断或取证，不是部署方式
docker cp orders-api:/app/report.json ./report.json
```

| 参数 | 含义 | 常见误区 |
| --- | --- | --- |
| `-d` | 后台运行 | 不代表服务健康，应配合日志和健康检查 |
| `--rm` | 容器退出后自动删除 | 不适合需要保留退出现场的故障排查 |
| `-p` / `--publish` | 发布主机端口到容器端口 | 默认可能监听所有接口，可用 `127.0.0.1:8080:8080` 限制本机 |
| `-e` / `--env` | 注入环境变量 | 不要借此传递长期秘密或暴露在终端历史中 |
| `-v` / `--volume` | 挂载 volume 或宿主机路径 | 复杂挂载优先使用 `--mount`，避免路径误解 |
| `--network` | 加入指定网络 | 服务调用应使用 DNS 名称，不要写死 IP |
| `--memory`、`--cpus` | 设置资源上限 | 单机没有 requests 概念，仍需基于负载调优 |
| `--restart` | Docker daemon 重启后的重启策略 | 不替代编排平台健康检查和发布能力 |

### 8.4 日志、进程和资源排查

```bash
# stdout/stderr 日志；-f 持续跟随，--since 限定时间范围
docker logs --follow --since 30m --tail 200 orders-api

# 配置、网络地址、挂载、退出码和 OOM 状态
docker inspect orders-api
docker inspect --format '{{.State.ExitCode}} {{.State.OOMKilled}}' orders-api

# 实时资源、容器进程和端口映射
docker stats orders-api
docker top orders-api
docker port orders-api

# 查看最终文件或验证 DNS
docker exec orders-api cat /etc/os-release
docker exec orders-api getent hosts postgres
```

`docker logs` 只读取容器写入 stdout/stderr 且由日志驱动保存的内容。生产中应用应输出结构化日志到 stdout，由 Fluent Bit、Vector、OpenTelemetry Collector 或平台日志代理统一采集；不要把检索、告警和保留完全寄托于单台主机的容器日志文件。

### 8.5 卷和网络命令

```bash
# Volume
docker volume create orders-data
docker volume ls
docker volume inspect orders-data
docker volume rm orders-data

# Network
docker network create --driver bridge orders-backend
docker network ls
docker network inspect orders-backend
docker network connect orders-backend orders-api
docker network disconnect orders-backend orders-api
```

`docker network inspect` 能确认容器是否真正接入预期网络。网络排查应按“进程是否监听 -> DNS 是否解析 -> TCP 是否可连 -> TLS/认证/应用协议是否正确”的顺序进行，避免直接归因于 Docker。

### 8.6 Compose 命令

```bash
# 启动、重建、查看状态和日志
docker compose up -d --build
docker compose ps
docker compose logs -f --tail 200 api

# 在指定服务中执行命令；exec 不创建新容器
docker compose exec api sh

# 输出变量替换和文件合并后的最终配置
docker compose config

# 停止并移除服务网络；-v 会删除命名卷中的数据
docker compose down
docker compose down -v
```

现代 Docker 使用 `docker compose`（Compose v2 插件）；旧 `docker-compose` 是遗留独立命令。团队应在文档与 CI 中统一版本，避免本地和流水线对 YAML 语义理解不同。

### 8.7 清理命令

```bash
# 先审计空间和对象
docker system df
docker container ls -a
docker image ls
docker volume ls

# 仅删除已停止容器或未被使用的镜像
docker container prune
docker image prune

# 删除所有未使用对象，可能影响其他项目的缓存或数据，需先评审
docker system prune
docker system prune --volumes
```

`docker system prune --volumes` 不应作为排障的第一反应。它会移除未被容器引用的卷，而“未被引用”不代表“业务上不再需要”。

---

## 9. Docker、Container、Pod 的详细区别

这三个词处于不同抽象层，并不是同义词。

| 维度 | Docker | Container（容器） | Pod |
| --- | --- | --- | --- |
| 本质 | 构建、分发、运行容器的一套产品与工具生态 | 隔离运行的一个或一组进程实例 | Kubernetes 最小的调度和部署单元 |
| 所属标准 | Docker 实现 OCI 等标准 | OCI 运行时概念，可由 Docker、containerd、CRI-O 等创建 | Kubernetes API 概念 |
| 是否必须依赖 Docker | 本身就是 Docker 工具 | 否，可由多种 runtime 创建 | 否，Kubernetes 1.24 后不再内置 dockershim |
| 包含关系 | Docker 可构建并运行容器 | 容器可被放进 Pod | 一个 Pod 包含一个或多个容器 |
| 调度单位 | Docker Engine 默认以容器为单位运行 | 单个容器实例 | Kubernetes 总是将整个 Pod 调度到同一节点 |
| 网络 | Docker network、端口发布、DNS | 有独立网络语义，取决于 runtime | 同 Pod 容器通常共享 IP、network namespace 和 `localhost` |
| 存储 | Volume、bind mount 等 | 有自己的文件系统视图 | Pod 可为多个容器挂载共享 volume |
| 生命周期 | Docker CLI、Compose、Swarm 或外部平台管理 | 独立启动和停止 | 被 Deployment、Job、StatefulSet 等控制器创建和替换 |

### 9.1 Docker：工具链与运行平台

Docker 通常指 Docker CLI、Docker Engine、BuildKit、Docker Compose、镜像格式和 Registry 工作流这套体验。它很适合本地开发和单机容器管理，也能构建 OCI 标准镜像。Kubernetes 运行这些镜像时，底层通常是 containerd 或 CRI-O，不要求节点安装 Docker Engine。

### 9.2 Container：实际运行的隔离进程

一个容器通常承担一个主要职责，例如 HTTP API、队列 Worker、数据库或一次性迁移任务。它拥有镜像、环境变量、进程、文件系统、网络和资源限制。容器不是轻量 VM：不要在同一个容器里用 supervisor 同时塞入 API、数据库、cron 和日志代理，除非它们确实无法独立管理且有清晰理由。

“一个容器一个进程”是职责单一的经验法则，不是内核限制。更准确地说，一个容器应承载一个可独立伸缩、发布、监控和故障恢复的职责。

### 9.3 Pod：紧耦合容器的共同边界

Pod 是 Kubernetes 将一个或多个紧密协作容器放在一起的单位。典型 Pod 包含一个业务容器，也可能包含：

- **Sidecar**：日志、代理、服务网格或凭据刷新辅助容器。
- **Init Container**：业务容器启动前执行一次的初始化，例如生成配置或准备数据。
- **Ephemeral Container**：临时调试容器，通常不参与正式服务生命周期。

同一 Pod 的容器通常共享：

1. 同一个网络 namespace，因此共享 Pod IP，可通过 `localhost` 通信。
2. 由 Pod 定义并共同挂载的 volume。
3. 调度位置、生命周期边界和部分资源上下文。

同一 Pod 的容器**不会默认共享完整文件系统或进程 namespace**。文件共享需要显式挂载同一个 volume；进程可见性需要启用 `shareProcessNamespace` 等特性。因为 Pod 是调度单位，API 和 Worker 应位于不同 Pod：二者扩缩容、健康检查和故障模式不同。只有必须共享网络和存储、需要同生共死的辅助组件才适合放进同一个 Pod。

### 9.4 具体判断

| 场景 | 推荐建模 | 原因 |
| --- | --- | --- |
| Web API 的三个副本 | 三个单容器 Pod，由 Deployment 管理 | 可独立滚动升级和扩缩容 |
| Envoy 代理与业务服务 | 同一 Pod 的两个容器 | 紧密配对，共享 `localhost` |
| API 与异步 Worker | 两个 Deployment、不同 Pod | CPU、队列指标、重试和发布节奏不同 |
| 数据库与 API | 独立数据库服务和 API Pod | 数据持久化、备份、故障转移边界不同 |
| 数据库 migration | 单独 Job | 可审计、可重试，避免多个 API 副本并发执行 |

---

## 10. 生产安全、可用性与可观测性

### 10.1 镜像与供应链安全

1. 使用可信、维护中的基础镜像，固定版本并定期更新。
2. 多阶段构建，只将运行时所需产物带入最终镜像。
3. 用 `.dockerignore` 排除 Git 历史、凭据和本地依赖。
4. 在 CI 生成 SBOM，扫描漏洞和许可证；定义漏洞修复 SLA，不只生成报告。
5. 对镜像签名，使用 Registry 访问控制和部署准入策略验证来源。
6. 不把真实秘密放入 Dockerfile、`ARG`、`ENV`、镜像层、Compose 文件或日志。

### 10.2 最小权限运行示例

```bash
docker run --rm \
  --user 10001:10001 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --pids-limit 200 \
  --memory 512m \
  orders-api:1.4.0
```

这些限制可能暴露应用隐含假设。例如 `--read-only` 可能发现应用将缓存、临时文件或日志写入镜像目录；正确修复通常是显式提供临时目录并输出日志到 stdout，而不是取消安全控制。

避免以下高风险做法：

- `--privileged`：几乎移除容器隔离，只能在明确、隔离且经过评审的场景使用。
- 挂载 `/var/run/docker.sock`：容器可能借此获得 Docker daemon 的高权限控制，风险接近主机高权限访问。
- root 用户运行应用：应用漏洞被利用后的影响范围更大。
- 为排障长期开放 SSH 或 shell：应使用受审计的临时调试机制。

### 10.3 可用性和资源

- 为每个服务设定 CPU、内存、进程数和临时存储边界，并以压测与运行指标调整。
- 容器停止时处理 `SIGTERM`：停止接收新流量、完成或安全中断在途请求、关闭连接后退出。
- 使用启动、存活、就绪检查，但不要让健康检查依赖不必要的外部系统，否则下游短暂故障会触发级联重启。
- 副本、跨故障域调度和负载均衡解决可用性；Docker `--restart` 不等于高可用。

### 10.4 最少可观测性要求

| 信号 | 需要回答的问题 | 常见实现 |
| --- | --- | --- |
| 日志 | 某请求为什么失败，影响了谁 | JSON stdout、集中日志、correlation ID |
| 指标 | 是否变慢、出错、资源耗尽或队列堆积 | Prometheus / OpenTelemetry metrics、仪表盘和告警 |
| 追踪 | 一次请求在哪个服务或外部调用耗时 | OpenTelemetry trace context |
| 事件 | 谁在何时部署、重启、改配置或扩容 | CI/CD 审计、Kubernetes event、变更记录 |

订单和支付系统不能只观察 CPU。还应观测请求量、错误率、P95/P99 延迟、支付渠道失败率、消息积压、数据库连接池、迁移状态、重试次数和业务对账差异。

---

## 11. 故障排查与事故响应

### 11.1 常见现象

| 现象 | 先检查什么 | 常见根因 | 处理方向 |
| --- | --- | --- | --- |
| 容器启动后立即退出 | `docker logs`、ExitCode | 主进程结束、命令错误、缺少配置 | 修复入口命令和配置，不用无限 sleep 掩盖 |
| 容器反复重启 | 日志、health、最近变更 | 依赖不可达、错误健康检查、应用崩溃 | 找到重启触发者，回滚或修复根因 |
| OOM kill | `OOMKilled`、`docker stats` | 内存泄漏、limit 偏低、负载突增 | 先保护服务，再做堆分析和限额调优 |
| 端口不能访问 | `docker port`、监听地址、防火墙 | 未发布端口、服务只监听 loopback | 从进程监听到网络路径逐层验证 |
| 服务名无法解析 | 网络 inspect、`getent hosts` | 未在同一网络、名称错误、固定 IP | 加入正确网络，使用服务 DNS 名 |
| 数据“丢失” | `Mounts`、volume inspect | 数据写入容器层、错误删卷 | 使用持久存储，按备份恢复 |
| 镜像拉取失败 | 引用、Registry 权限、节点网络 | 标签不存在、凭据过期、架构不匹配 | 修正引用，更新最小权限凭据，验证平台 |

### 11.2 排障顺序

1. 确认影响范围、开始时间和最近变更，避免多人同时做不可追踪的修改。
2. 查看容器或 Pod 是否运行、重启次数、退出码、事件、镜像版本和资源使用。
3. 查看应用日志和下游服务指标，用 request ID 或 trace ID 建立因果链。
4. 逐层验证：DNS -> TCP -> TLS -> 认证 -> 应用协议 -> 数据一致性。
5. 优先采取低风险缓解措施，例如回滚已知变更、扩展健康副本、限流或切换依赖。
6. 修复后验证业务指标和数据一致性，记录根因、时间线和预防措施。

生产中不要用 `docker exec` 直接修改代码、安装包或手工改配置来“修好”容器。这会制造未受版本控制的漂移，下一次重建仍会失败。正确做法是将修复提交到代码、配置仓库或受控部署声明中。

---

## 12. 高频架构判断题

### Docker 镜像为什么要小？

小镜像通常拉取更快、攻击面更小、扫描更容易、部署更稳定。但“最小”不能牺牲兼容性和可调试性。应先去除构建工具、缓存和无关文件，再选择与应用依赖匹配的基础镜像；不能机械地把所有服务改为 Alpine 或 distroless。

### 为什么不在容器中保存数据库数据？

容器可写层随实例删除而消失，也没有备份、复制、故障转移和一致性恢复能力。数据库数据至少应位于显式持久化存储中，生产关键数据最好由具备备份和高可用能力的数据库服务管理。

### `latest` 有什么问题？

`latest` 是可变标签，不是版本。今天部署与明天重新部署可能得到不同二进制，无法可靠复现、审计和回滚。应部署不可变 digest，并保留与源代码、配置和扫描结果的关联。

### Docker Compose 能用于生产吗？

可以用于范围明确的单机生产工作负载，但它不是默认的高可用方案。应依据可用性目标、故障域、数据策略、团队运维能力、发布频率和合规要求选择。若选 Compose，必须补齐主机管理、备份、证书、监控、日志、恢复演练和发布治理。

### 为什么区分 liveness 和 readiness？

liveness 判断进程是否需要重启；readiness 判断实例是否应接收流量。把数据库连通性等不稳定外部依赖直接作为 liveness 条件，可能在数据库短暂波动时同时重启大量 API，放大故障。健康检查必须服务于明确的恢复决策。

---

## 13. 生产发布前检查清单

- [ ] Dockerfile 使用固定、可信基础镜像，最终镜像不以 root 运行。
- [ ] `.dockerignore` 排除了 Git、私钥、环境文件和本地依赖。
- [ ] 镜像通过测试、漏洞扫描、SBOM 和签名或等效供应链校验。
- [ ] 部署引用明确版本，最好引用 digest，不使用浮动 `latest`。
- [ ] 秘密由受控密钥系统提供，未出现在代码、镜像、日志或命令历史中。
- [ ] API、Worker、迁移和数据库按独立职责部署，数据不写入容器层。
- [ ] 设置资源 requests/limits 或单机等效限制，已做压测或容量评估。
- [ ] 有合理的启动、存活和就绪检查，应用能优雅处理 `SIGTERM`。
- [ ] 日志、指标、追踪、审计事件和关键业务告警均已接入。
- [ ] 数据库与持久化数据有备份，且最近完成过恢复演练。
- [ ] 发布步骤、回滚条件、上一个稳定镜像 digest 和负责人已记录。

## 14. 总结

Docker 的价值不只是“把应用装进容器”，而是把应用交付变成可重复、可验证、可审计的过程。基础阶段要理解镜像、容器、网络和卷；进阶阶段要掌握多阶段构建、Compose、资源限制和安全；生产阶段则必须把 Docker 纳入 CI/CD、镜像供应链、编排、可观测性、数据保护和发布治理的完整体系。

最重要的架构判断是：**Docker 负责构建和运行一致的容器，容器负责单一运行职责，Pod 负责 Kubernetes 中紧耦合容器的共同调度；高可用、数据一致性和安全治理需要由整套产品架构共同保证。**