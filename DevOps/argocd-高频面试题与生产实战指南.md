# Argo CD 高频面试题与生产实战指南

本文面向 DevOps、SRE、平台工程师及 Kubernetes 工程师面试。每一道题均给出回答要点、技术细节、常见误区和可落地的生产案例。示例使用“支付结算 API（`payment-api`）”说明从代码构建到生产发布的全过程。

---

## 目录

1. [核心概念与架构](#一核心概念与架构)
2. [应用交付与配置管理](#二应用交付与配置管理)
3. [安全、多租户与高可用](#三安全多租户与高可用)
4. [排障、发布与回滚](#四排障发布与回滚)
5. [端到端生产案例](#五端到端生产案例)
6. [面试速答清单](#六面试速答清单)

---

## 一、核心概念与架构

### 1. 什么是 GitOps？Argo CD 在其中解决了什么问题？

**参考回答：**

GitOps 是一种以 Git 仓库作为期望状态（desired state）唯一可信来源的运维模式。基础设施和应用配置均通过 Pull Request 评审、合并和版本历史进行管理；自动化控制器持续将运行环境的实际状态（live state）收敛到 Git 中声明的期望状态。

Argo CD 是 Kubernetes 的声明式持续交付（CD）控制器，负责：

- 从 Git、Helm 仓库、OCI Helm Registry 等来源读取 Kubernetes 清单；
- 将清单渲染为目标资源，并与集群实际资源比较；
- 显示 `Synced` / `OutOfSync`、`Healthy` / `Degraded` 等状态；
- 在人工批准或自动同步策略下执行部署、修剪和自愈；
- 通过 RBAC、`AppProject`、审计记录治理多团队发布。

传统 CI 直接持有生产集群的 `kubectl` 凭证，并以“推送”方式部署；Argo CD 则由运行在受控网络内的控制器“拉取”Git 变更并部署。这样 CI 不必拥有生产集群写权限，回滚也变为一次可审计的 Git 提交或 Argo CD 历史操作。

**实际例子：**

支付团队构建 `payment-api:2.8.1` 后，不直接执行 `kubectl apply`，而是创建 PR，把生产环境的镜像标签从 `2.8.0` 改为 `2.8.1`。变更经开发、测试和变更经理审批后合并。Argo CD 检测到 Git revision 变化，按策略同步到生产集群；提交历史就是完整的发布审计链。

**常见误区：**

- GitOps 不等于“把 YAML 放到 Git”。必须有持续 reconciliation（持续对比和收敛）机制。
- Argo CD 不是 CI 工具，通常不负责编译、单测或构建镜像；这些仍由 GitHub Actions、GitLab CI、Jenkins、Tekton 等完成。
- 自动同步不意味着所有环境都无需审批；生产环境通常结合受保护分支、PR 审批和变更窗口。

---

### 2. 请说明 Argo CD 的核心组件及一次同步的调用链。

**参考回答：**

| 组件 | 主要职责 |
|---|---|
| `argocd-server` | 提供 Web UI、CLI/gRPC/REST API、SSO 登录入口和 RBAC 鉴权。 |
| `argocd-application-controller` | 监控 `Application`，比较期望与实际状态，编排同步、健康评估和资源修剪。 |
| `argocd-repo-server` | 克隆/缓存 Git 仓库，执行 Helm、Kustomize、Jsonnet、插件等渲染，返回目标清单。 |
| `argocd-redis` | 缓存和协调数据；不是 Git 或 Kubernetes 的权威数据源。 |
| `argocd-applicationset-controller` | 根据 Git 目录、集群、Pull Request 等生成多个 `Application`。 |
| Dex 或外部 OIDC 身份提供商 | Dex 可作为身份代理；生产中更常接入 Entra ID、Okta、Keycloak 等企业 IdP。 |

一次同步的典型链路：

```text
Git commit / Webhook / 定时刷新
             |
             v
application-controller  ---> repo-server ---> Git/Helm/OCI source
             |                  |
             |                  v
             |             rendered manifests
             v
Kubernetes API Server ---> target cluster resources
             |
             v
actual state + health status ---> application-controller
```

控制器把 `Application.spec.source` 渲染后的结果作为期望状态，从 Kubernetes API 读取目标资源作为实际状态，再根据同步策略创建、更新、删除资源。它并不直接 SSH 到节点或容器中执行命令。

**面试加分点：**

- 多集群场景下，Argo CD 对目标集群使用注册的 Kubernetes API 凭据；它不要求在每个工作集群安装完整 Argo CD。
- 仓库凭据应由 `repo-server` 使用，集群写入权限由 `application-controller` 对 Kubernetes API 使用。两类权限应分离并最小化。

---

### 3. `Application`、`AppProject` 和 `ApplicationSet` 分别是什么？如何选择？

**参考回答：**

- **`Application`**：Argo CD 最核心的 CRD，定义“从哪个来源、哪个 revision、哪条路径，部署到哪个集群和 namespace”。
- **`AppProject`**：租户或业务边界。它限制 Application 可使用的源码仓库、目标集群/namespace、允许的资源种类，并附带项目角色（project roles）。
- **`ApplicationSet`**：Application 的模板和生成器。它根据目录、集群列表、Git 文件、Pull Request 或矩阵等输入，批量生成 Application。

选择原则：

| 场景 | 推荐方式 |
|---|---|
| 一个服务部署到一个环境 | 单个 `Application` |
| 团队边界、可访问仓库和部署范围隔离 | `AppProject` + `Application` |
| 同一服务要部署到 dev/test/prod 或多个集群 | `ApplicationSet` |
| 平台根应用统一管理很多基础组件 | App of Apps 或 `ApplicationSet`，按规模选择 |

**实际例子：**

平台团队定义 `payments` 项目，允许其应用只从 `https://github.com/acme/platform-config.git` 获取配置，只能部署到 `payments-dev`、`payments-prod` namespace。然后用 `ApplicationSet` 为 `dev`、`uat`、`prod` 三个环境生成三个 `payment-api` Application，避免复制三份几乎相同的 CR。

---

### 4. `Application` 清单中最重要的字段有哪些？

**参考回答：**

以下示例将生产支付 API 从 Git 中的 Kustomize overlay 部署到指定 namespace：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-api-prod
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: payments
spec:
  project: payments
  source:
    repoURL: https://github.com/acme/platform-config.git
    targetRevision: main
    path: apps/payment-api/overlays/prod
  destination:
    name: prod-apac
    namespace: payments-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
```

字段说明：

- `metadata.namespace`：Application CR 所在的 Argo CD 控制平面 namespace，常为 `argocd`；它**不是**业务资源目标 namespace。
- `spec.project`：必须匹配 `AppProject`，默认项目虽方便，但生产不宜长期依赖。
- `source.repoURL`、`targetRevision`、`path`：分别指定来源、版本和清单路径。`targetRevision` 可为分支、tag 或 Git SHA；生产建议使用受保护分支或不可变 tag/SHA。
- `destination`：目标集群和 namespace。`name` 需匹配 Argo CD 已注册的集群名；也可使用 `server` 指向 API Server 地址。
- `syncPolicy.automated`：开启自动同步；`prune` 允许删除 Git 中已移除的受管资源，`selfHeal` 用于纠正集群中的手工漂移。

**常见误区：**

- `CreateNamespace=true` 只负责创建 namespace，不会自动为该 namespace 配置 ResourceQuota、NetworkPolicy 或正确 RBAC。
- 不要让生产应用跟随 `HEAD` 或无治理的开发分支；它会破坏发布可追溯性。

---

### 5. Argo CD 如何判定 `OutOfSync`？什么是配置漂移（drift）？

**参考回答：**

Argo CD 对比渲染出的期望资源与 Kubernetes API 返回的实际资源。当两者在受管理字段上存在差异，应用为 `OutOfSync`。差异可能来自：

1. Git 中有新提交，但尚未同步；
2. 管理员在集群中手工修改了 Deployment、Service、HPA 等资源；
3. API Server 默认值、控制器或 webhook 注入字段导致的非业务差异；
4. 另一个控制器也在管理同一字段或同一资源，造成控制权冲突。

`selfHeal: true` 时，Argo CD 会尝试把手工修改恢复为 Git 的期望状态。它适合声明式、单一所有者的资源；不应盲目用于会被其他控制器合法修改的字段。

例如 HPA 会修改 Deployment 的 `spec.replicas`。若 Git 固定 `replicas: 3`，同时 HPA 负责弹性伸缩，就会反复发生漂移。正确做法通常是移除 Deployment 中的 `spec.replicas` 的管理，或针对该字段配置精确的差异忽略规则：

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

**关键原则：**只忽略已经确认由其他控制器拥有、且不会掩盖风险的最小字段。不能用宽泛的 `ignoreDifferences` 把真实配置偏差隐藏掉。

---

### 6. 自动同步、`prune`、`selfHeal` 各有什么风险和治理方式？

**参考回答：**

| 设置 | 行为 | 生产建议 |
|---|---|---|
| 无 `automated` | 发现差异后等待人工同步 | 适用于强审批生产环境或首次上线。 |
| `automated` | Git 变更自动部署 | 适用于开发、测试环境；生产须有受保护分支和合并审批。 |
| `prune: true` | 删除 Git 中移除的受管资源 | 可用，但须避免误删共享资源；采用资源所有权、PR 审批和预生产验证。 |
| `selfHeal: true` | 修复对受管字段的集群手工修改 | 用于禁止手工改生产的环境；须处理 HPA、operator 等字段所有权。 |
| `allowEmpty: true` | 允许目标清单为空且可 prune | 高风险，仅用于明确的退役或临时环境流程。 |

实践上，开发环境常使用 `automated + prune + selfHeal`；生产环境也可以自动化，但变更入口应是受保护的 Git 分支、CODEOWNERS 审批、策略检查及明确的变更窗口。这样“自动”指的是审批后快速一致地执行，而非绕过发布治理。

---

### 7. Argo CD 支持哪些清单工具？Helm 与 Kustomize 如何取舍？

**参考回答：**

Argo CD 能原生渲染普通 YAML、Kustomize、Helm、Jsonnet，并可通过 Config Management Plugin 扩展其他工具。它通常使用 Helm 的模板渲染能力，但不以 Helm release 的方式管理生命周期；部署追踪、同步和历史由 Argo CD 负责。

| 工具 | 擅长场景 | 注意事项 |
|---|---|---|
| Plain YAML | 少量稳定资源 | 环境差异多时复制会失控。 |
| Kustomize | 基础清单 + 环境 overlay、补丁、镜像替换 | 更适合把环境配置显式保存在 Git。 |
| Helm | 第三方组件、可复用产品包 | 严格控制 values 来源与 chart 版本；不要依赖漂浮版本。 |
| Jsonnet/插件 | 复杂生成逻辑或组织标准工具链 | 增加学习和维护成本，应控制插件的执行安全。 |

**实际例子：**

- 将 `ingress-nginx`、`cert-manager` 作为固定版本的 Helm chart 部署。
- 将自研 `payment-api` 使用 Kustomize：`base` 包含 Deployment、Service、ServiceAccount；`overlays/prod` 覆盖镜像 tag、副本、资源配额、域名和 PDB。

使用 Helm 的 Application 示例：

```yaml
spec:
  source:
    repoURL: https://charts.example.com
    chart: payment-observability
    targetRevision: 4.2.3
    helm:
      releaseName: payment-observability
      valueFiles:
        - $values/environments/prod/observability-values.yaml
  sources:
    - repoURL: https://charts.example.com
      chart: payment-observability
      targetRevision: 4.2.3
      helm:
        valueFiles:
          - $values/environments/prod/observability-values.yaml
    - repoURL: https://github.com/acme/platform-config.git
      targetRevision: main
      ref: values
```

多源（`sources`）写法可把 chart 与私有 values 分开管理。实际清单中应二选一地使用 `source` 或 `sources`，不要同时配置。

---

### 8. 如何实现资源依赖和有序发布？Sync Wave 与 Hook 的区别是什么？

**参考回答：**

Argo CD 默认按资源类型及名称排序，但业务依赖不能只依赖默认顺序。可用以下机制：

- **Sync Wave**：通过注解 `argocd.argoproj.io/sync-wave` 指定阶段，数值小的先应用；适合 CRD、Namespace、数据库迁移 Job、工作负载之间的确定性顺序。
- **Resource Hook**：在同步生命周期特定阶段运行 Job/Workload，常用 `PreSync`、`Sync`、`PostSync`、`SyncFail`。适合数据库迁移、烟囱测试、失败通知。

数据库 schema 迁移的安全示例：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: payment-schema-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
    argocd.argoproj.io/sync-wave: "-1"
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: payment-migrator
      containers:
        - name: migrate
          image: registry.example.com/payment/migrator:2.8.1
          args: ["migrate", "up"]
```

**面试重点：**

- 数据库迁移必须向后兼容。例如先增加可空列和新索引，再发布读取新字段的应用，最后在另一次变更中清理旧字段。
- Hook Job 要有明确超时、幂等设计和最小数据库权限。失败后不要自动无限重试或让人工绕过迁移强行发布。
- Wave 解决资源应用顺序，不代表应用真正就绪；就绪判断仍依赖健康检查、Job 完成和同步状态。

---

### 9. App of Apps 与 ApplicationSet 有何区别？什么时候会导致“控制器打架”？

**参考回答：**

**App of Apps** 指一个根 `Application` 管理一组子 Application CR。它适合表示静态、层次化的平台构成，例如根应用管理 `cert-manager`、监控、日志、网络策略和业务团队根应用。

**ApplicationSet** 由生成器动态创建 Application，适合“同一模板乘以环境/集群/目录”的场景，例如每个新区域集群自动生成全套观测应用。

两者不应同时管理同一个子 Application。否则根应用和 ApplicationSet 都会修改同一个 CR 的 spec，产生反复漂移甚至错误删除。还要避免多个 Application 管理同一个 Kubernetes 资源；每个资源必须有清晰唯一的声明所有者。

一个基于目录的 ApplicationSet 示例：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: payment-api
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/acme/platform-config.git
        revision: main
        directories:
          - path: apps/payment-api/overlays/*
  template:
    metadata:
      name: "payment-api-{{path.basename}}"
    spec:
      project: payments
      source:
        repoURL: https://github.com/acme/platform-config.git
        targetRevision: main
        path: "{{path}}"
      destination:
        name: in-cluster
        namespace: "payments-{{path.basename}}"
```

生产中还应通过 `AppProject` 限制这个模板可部署的 namespace，避免路径或模板变量意外扩大部署范围。

---

## 二、应用交付与配置管理

### 10. 如何设计多环境 GitOps 仓库？

**参考回答：**

一个易审计的结构是将应用基础清单与环境差异分离：

```text
platform-config/
├── apps/
│   └── payment-api/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── serviceaccount.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           ├── dev/
│           ├── uat/
│           └── prod/
├── projects/
│   └── payments.yaml
└── bootstrap/
    └── root-app.yaml
```

推荐实践：

1. **代码与部署配置分库或分目录管理。** CI 构建应用代码；受控配置仓库存放环境声明。
2. **环境提升通过 PR 完成。** 测试通过后，把已验证的不可变镜像 digest 或 tag 从 `dev` 显式提升至 `uat`、`prod`，而不是生产重新构建。
3. **生产使用不可变引用。** 镜像 digest 最可追溯，例如 `registry.example.com/payment/api@sha256:...`；避免使用 `latest`。
4. **每个环境使用独立 namespace、配额、网络策略、Secret 引用和目标集群。**
5. **保护生产目录和分支。** 对 `overlays/prod` 启用 CODEOWNERS，要求应用负责人和平台负责人审批。

**常见误区：**只用一个 Git 分支代表环境会让合并、回滚和历史关联变复杂。多数团队更容易治理的方式是单一主干 + 每个环境有明确的 overlay 路径；也可以采用环境分支，但必须约定清晰的提升与回滚流程。

---

### 11. 如何让 CI 与 Argo CD 协作完成镜像发布？

**参考回答：**

推荐的不可变发布链路：

```text
Commit application code
  -> CI: test, SAST, build image, SBOM/sign
  -> push immutable image digest
  -> CI creates PR in platform-config
  -> review/policy checks/merge
  -> Argo CD detects Git revision and syncs
  -> metrics and smoke checks validate release
```

CI 更新配置的方式有两种：

1. **CI 创建配置 PR**：最通用、最易审计。CI 使用专门的 Git 身份更新 `kustomization.yaml` 的 `images` 字段或 Helm values 中的镜像 digest。
2. **Argo CD Image Updater**：它可根据配置策略发现新镜像并写回 Git 或更新应用参数。适合开发/测试环境的自动更新；生产仍宜要求 Git 写回、PR 审批和不可变 digest。

Kustomize 生产 overlay 的片段：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: payments-prod
images:
  - name: registry.example.com/payment/api
    newName: registry.example.com/payment/api
    digest: sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
patches:
  - path: deployment-production.yaml
```

**面试加分点：**CI 不应把生产集群的管理员 kubeconfig 存在变量中。CI 只需要配置仓库的受限写入权限；Argo CD 才是受控的集群写入实体。

---

### 12. Argo CD 如何处理 Secret？为什么不能把明文 Secret 直接提交到 Git？

**参考回答：**

Git 的提交历史具有长期、广泛复制的特性。即使后续删除明文 Secret，旧提交、fork、缓存和克隆中仍可能存在，因此明文密码、私钥、Token 和 kubeconfig 不应提交。

常见方案：

| 方案 | 机制 | 适用场景 |
|---|---|---|
| External Secrets Operator | 从 Vault、AWS Secrets Manager、Azure Key Vault 等拉取，生成 Kubernetes Secret | 云或企业密钥系统为权威源，生产首选。 |
| Sealed Secrets | 公钥加密后的 Secret 可安全提交；集群内控制器解密 | 中小规模 GitOps，注意密钥轮换与集群边界。 |
| SOPS + KMS/age | Git 中保存加密文件，渲染时解密 | 需要 Git 审计加密内容且已有 KMS 治理。 |

以 External Secrets 为例，Git 中只保存对密钥路径的引用：

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-db-credentials
  namespace: payments-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: production-vault
    kind: ClusterSecretStore
  target:
    name: payment-db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: kv/payments/prod/database
        property: password
```

应用的 Deployment 只引用 `payment-db-credentials`。Vault 访问身份应使用 workload identity、IRSA 或受限 ServiceAccount，而不是在 Git 中保存长期云访问密钥。

---

### 13. 如何使用 `AppProject` 和 RBAC 实现多团队隔离？

**参考回答：**

`AppProject` 是“应用可以从哪里来、能部署到哪里、允许什么资源”的策略边界；Argo CD RBAC 是“谁可以查看、同步、修改或删除什么”的操作边界。两者必须一起使用。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments
  namespace: argocd
spec:
  description: Payment domain workloads
  sourceRepos:
    - https://github.com/acme/platform-config.git
  destinations:
    - namespace: payments-dev
      server: https://kubernetes.default.svc
    - namespace: payments-prod
      name: prod-apac
  namespaceResourceWhitelist:
    - group: ""
      kind: ConfigMap
    - group: ""
      kind: Service
    - group: apps
      kind: Deployment
    - group: autoscaling
      kind: HorizontalPodAutoscaler
  roles:
    - name: deployer
      policies:
        - p, proj:payments:deployer, applications, get, payments/*, allow
        - p, proj:payments:deployer, applications, sync, payments/*, allow
      groups:
        - oidc:payments-deployers
```

生产治理建议：

- 不使用长期共享的 `admin` Token；接入 OIDC，并将企业用户组映射到角色。
- 开发人员有 dev 同步权限；生产同步权限分给小范围发布组，或只允许通过受保护 Git 合并触发自动同步。
- 对集群级资源采用默认拒绝（`clusterResourceWhitelist`）策略。CRD、ClusterRole、Namespace、NetworkPolicy 等高影响资源应由平台项目单独管理。
- `AppProject` 不能替代 Kubernetes RBAC。`application-controller` 在目标集群中的 ServiceAccount 仍须最小授权。

---

### 14. 如何安全地将多个 Kubernetes 集群接入 Argo CD？

**参考回答：**

接入集群的本质是为 Argo CD 保存到目标 Kubernetes API 的访问配置与凭据。生产实践应遵循：

1. 使用专用 ServiceAccount，按 namespace 或项目授予最小权限，绝不直接复用 `cluster-admin`；
2. 只允许控制平面到目标 API Server 的必要网络路径，使用私有网络、VPN、PrivateLink 或防火墙规则；
3. 管理 kubeconfig、bearer token、客户端证书的轮换和失效流程；
4. 用 `AppProject.destinations` 限制每个团队可使用的集群和 namespace；
5. 对集群接入、凭据变更和高权限同步建立审计告警。

如果用声明式集群 Secret 管理注册信息，该 Secret 本身极其敏感，必须由外部密钥方案保护并严格限制谁能读取 `argocd` namespace。对于大规模集群舰队，还应评估 hub-and-spoke、每区域独立 Argo CD 实例或代理连接模型，降低单一控制平面对所有生产集群的爆炸半径。

---

## 三、安全、多租户与高可用

### 15. Argo CD 生产环境有哪些主要安全风险？如何缓解？

**参考回答：**

| 风险 | 后果 | 控制措施 |
|---|---|---|
| Git 仓库被篡改 | 恶意清单被部署到生产 | 受保护分支、强制 PR 审批、签名提交/供应链验证、最小化写权限。 |
| `application-controller` 权限过大 | 任意应用可获取集群管理员能力 | 用 `AppProject` 限制目的地和资源；目标集群采用最小 RBAC；分离平台与业务实例。 |
| 明文密钥或仓库凭据泄露 | 数据库、云账户或源码泄露 | External Secrets/SOPS，密钥轮换，限制读取 Argo CD Secret。 |
| 不安全 Config Management Plugin | 渲染阶段执行任意代码 | 插件隔离、固定镜像、最小文件/网络权限、审计输入来源。 |
| Web UI/API 暴露 | 未授权同步或 Token 泄露 | OIDC + MFA，禁用匿名访问，TLS，短期 Token，审计日志。 |
| Hook Job 过度授权 | 迁移 Job 成为横向移动入口 | 独立 ServiceAccount、最小 RBAC、固定镜像 digest、NetworkPolicy。 |

需要强调：Argo CD 是供应链的一部分，不是单纯的部署面板。Git 评审、镜像签名、准入控制（如 Kyverno 或 Gatekeeper）和运行时安全共同构成防线。

---

### 16. Argo CD 如何高可用部署和扩容？

**参考回答：**

Argo CD 的 HA 目标是避免控制平面单点，而不是让同一资源被重复部署。生产通常使用官方 HA 安装清单或 Helm chart 的 HA 配置，并至少关注：

- `argocd-server` 多副本，前置负载均衡/Ingress；
- `repo-server` 多副本，按仓库数量、渲染耗时和并发扩容；
- `application-controller` 采用支持的高可用/分片（sharding）配置，按 Application 与集群数量扩容；
- Redis 使用支持高可用的部署模式或托管服务，并按官方兼容配置实施；
- 为关键 Pod 配置 PDB、反亲和/拓扑分布、资源 requests/limits 和监控告警；
- 将 Git、OIDC、DNS、证书、网络路径也纳入可用性设计。

衡量容量不能只看 Pod CPU。应观察 Application 数量、目标集群数、Git 仓库数量、刷新队列长度、同步耗时、repo-server 渲染延迟和 Kubernetes API 限流错误。

**常见误区：**把所有组件简单加副本不一定能线性扩容。要遵循当前版本官方 HA 架构；控制器并发、分片策略、Redis 连接、API Server 限流和仓库缓存都会成为瓶颈。

---

### 17. 为什么 Argo CD 有时显示 `Unknown`、`Progressing` 或 `Degraded`？如何定义健康检查？

**参考回答：**

- **`Healthy`**：资源达到控制器认定的健康状态，例如 Deployment 的可用副本满足期望。
- **`Progressing`**：资源正在收敛，例如滚动升级尚未完成。
- **`Degraded`**：资源已失败或不可用，例如 Deployment 超过进度截止时间、Job 失败。
- **`Missing`**：期望资源在目标集群中不存在。
- **`Unknown`**：Argo CD 无法评估健康，常见于未内置健康规则的 CRD，或没有权限读取资源状态。

对于自定义资源（例如 `Rollout`、内部 `PaymentGateway` CR），应通过 Lua health check 或自定义健康配置，基于其 `status.conditions` 判断是否就绪。健康规则要使用控制器实际写入、语义稳定的条件，例如 `Ready=True`，而非短暂的消息文本。

先执行以下排查：

```bash
argocd app get payment-api-prod
kubectl get application payment-api-prod -n argocd -o yaml
kubectl get deploy,pods,events -n payments-prod
kubectl describe deployment payment-api -n payments-prod
```

如果资源健康正常但 Argo CD 显示 `Unknown`，检查该 CRD 是否存在合适的健康脚本、Argo CD 是否有读取该资源 `status` 的权限，以及 CR controller 是否正确更新 status。

---

### 18. 如何配合策略即代码（Policy as Code）阻止危险配置进入集群？

**参考回答：**

Argo CD 负责声明式部署，但不能代替策略治理。成熟流程至少有三层：

1. **PR 阶段**：运行 `kustomize build` / `helm template`、schema 校验、OPA Conftest 或 Checkov 等静态策略检查；
2. **准入阶段**：使用 Kyverno 或 OPA Gatekeeper 在 Kubernetes API 拒绝不合规资源；
3. **运行阶段**：监控漂移、审计日志、镜像漏洞与行为异常。

支付 API 的典型策略：

- 生产 Deployment 必须设置 `resources.requests/limits`、readinessProbe 和 `runAsNonRoot`；
- 禁止 `privileged: true`、`hostPath`、`hostNetwork`；
- 禁止 `:latest`，要求来自受信任 registry 的 digest；
- 必须存在 NetworkPolicy 和 PDB；
- namespace 必须具有 `team`、`environment`、`cost-center` 标签。

不要把 policy check 只放在 CI。绕过 CI 的 `kubectl apply`、其他自动化系统和误配置仍可能进入 API Server，因此准入控制是最后一道技术防线。

---

## 四、排障、发布与回滚

### 19. 应用一直处于 `OutOfSync`，你的排障步骤是什么？

**参考回答：**

按“渲染结果 -> 差异内容 -> 所有权 -> 集群事件”的顺序排查：

```bash
# 查看 Application 状态、目标 revision、资源状态和同步错误
argocd app get payment-api-prod

# 查看精确差异
argocd app diff payment-api-prod

# 检查源仓库、渲染与控制器日志
kubectl logs -n argocd deploy/argocd-repo-server --since=30m
kubectl logs -n argocd statefulset/argocd-application-controller --since=30m

# 检查实际资源及事件
kubectl get events -n payments-prod --sort-by=.lastTimestamp
kubectl get deploy payment-api -n payments-prod -o yaml
```

重点检查：

1. `repoURL`、revision、path、chart 版本、values 文件是否正确，是否存在渲染错误；
2. 是否由 mutating webhook、HPA、operator 修改了目标字段；
3. 是否有多个 Application 或其他工具共同管理同一资源；
4. 是否因为 ignore rule 写错而掩盖/制造了差异；
5. 是否因 RBAC、CRD 缺失、API 版本不兼容或 immutable field 修改导致 apply 失败。

修复应回到 Git 中完成。除紧急止血外，不要以手工修改生产资源作为“最终修复”，否则下次 reconciliation 会再次覆盖它。

---

### 20. 同步失败且 Deployment 无法就绪，如何区分 Argo CD 问题与应用问题？

**参考回答：**

先区分失败阶段：

| 阶段 | 典型现象 | 优先检查 |
|---|---|---|
| 获取/渲染源 | `ComparisonError`、仓库认证失败、Helm/Kustomize 模板失败 | repo-server 日志、仓库凭据、引用版本、模板语法。 |
| 应用资源 | `SyncFailed`、RBAC denied、CRD not found、immutable field | application-controller 日志、Kubernetes events、API 权限。 |
| 等待健康 | 已创建资源但 `Progressing`/`Degraded` | Pod 状态、探针、镜像、配置、依赖服务和资源配额。 |

例如：Argo CD 显示同步成功但 `payment-api` 为 `Degraded`，`kubectl get pods` 发现 `ImagePullBackOff`。此时 Argo CD 的 apply 逻辑并无故障，根因是镜像不存在、registry 凭据失效或网络无法访问 registry。修复镜像引用或 `imagePullSecret` 后，再让 Argo CD 收敛。

相反，若错误为 `namespaces "payments-prod" is forbidden`，说明目标集群中 Argo CD 所用身份没有创建/访问权限，需要按最小权限调整 RBAC 或先由平台流程预创建 namespace。

---

### 21. Argo CD 如何回滚？Git 回滚与 UI/CLI 回滚如何取舍？

**参考回答：**

**首选 Git 回滚。**将配置仓库恢复到上一已知良好的 commit 或镜像 digest，走同样的审批、审计和同步流程。因为 Git 回滚会让期望状态与实际状态重新一致，后续 reconciliation 不会再把故障版本部署回来。

Argo CD 也保留 Application 同步历史，可通过 UI 或 CLI 回滚到历史 revision。它适用于紧急恢复，但随后必须立即把 Git 修正到同一状态；否则下一次自动同步会将应用再推回 Git 中的故障声明。

数据库迁移是回滚难点。应采用 expand/contract 模式：

1. v1 引入新表/新列，但保留旧结构；
2. v2 双写或兼容读取；
3. 验证并切流；
4. 在独立、可恢复的变更中删除旧结构。

不要假设每个数据库 DDL 都能安全自动逆向执行。

---

### 22. 如何实现蓝绿、金丝雀等渐进式交付？Argo CD 与 Argo Rollouts 如何协作？

**参考回答：**

Argo CD 负责把 Git 中声明的资源同步到集群；Argo Rollouts 负责更精细的工作负载发布控制，例如金丝雀分批放量、蓝绿预览、分析指标和自动/手工暂停。二者职责互补。

一个 Rollout 声明在 Git 中，由 Argo CD 同步：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-api
  namespace: payments-prod
spec:
  replicas: 6
  strategy:
    canary:
      canaryService: payment-api-canary
      stableService: payment-api-stable
      steps:
        - setWeight: 10
        - pause:
            duration: 5m
        - setWeight: 30
        - pause: {}
        - setWeight: 100
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      containers:
        - name: api
          image: registry.example.com/payment/api@sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
          ports:
            - containerPort: 8080
```

对于支付服务，应把 Rollouts AnalysisTemplate 连接到 Prometheus 指标，例如 5xx 比例、支付授权错误率、p95 延迟和业务拒绝率。仅凭 Pod `Ready` 不足以证明发布安全。暂停点应由明确的 SLO 门槛和当班审批人决定。

---

### 23. 请描述一次真实的 Argo CD 生产事故及处置方案。

**示例答案：**

一次支付 API 发布后，Argo CD 同步显示成功，但金丝雀 10% 流量的授权失败率从 0.2% 升至 3%。排查发现配置 PR 同时更新了镜像 digest 和数据库连接池上限；新版本启动后对只读副本建立过多连接，触发数据库连接耗尽。

处置过程：

1. Rollouts 分析指标触发失败，停止扩大流量并中止 Rollout；
2. 通过 Git revert 将生产 overlay 恢复到上一 digest 和已验证配置；
3. Argo CD 自动收敛，稳定版本接管流量；
4. 检查 Argo CD 操作记录、Git PR、Prometheus、数据库连接数和应用日志，确认根因；
5. 修复连接池参数，增加连接数预算检查和预生产压测；
6. 将“镜像升级”和“高风险运行参数调整”拆分为两个 PR，后续发布使用自动分析门禁。

**为什么这是好答案：**它说明了 Git 作为回滚依据、渐进交付的止损价值、业务指标的重要性，以及事故后的工程性改进，而不只是“执行一次 rollback 命令”。

---

## 五、端到端生产案例

### 24. 请为一个跨区域支付服务设计完整的 Argo CD 方案。

**场景：**

`payment-api` 运行在新加坡和东京两个生产 Kubernetes 集群。服务依赖 PostgreSQL、Redis、外部卡组织 API；必须满足高可用、可审计发布、最小权限和快速回滚。开发、UAT、生产环境隔离。

**推荐架构：**

```text
Application repo
  -> CI: test, scan, build, sign, push image digest
  -> create PR in platform-config
  -> Git review + policy checks
  -> Argo CD (HA control plane)
       ├-> prod-sg cluster / payments-prod namespace
       └-> prod-jp cluster / payments-prod namespace
  -> Argo Rollouts + Prometheus analysis
  -> External Secrets -> Vault / cloud secret manager
```

**配置设计：**

1. `AppProject/payments` 只允许受信任配置仓库、两个生产集群的 `payments-prod` namespace，限制允许资源类型。
2. 用 Cluster generator 的 `ApplicationSet` 为 `prod-sg` 和 `prod-jp` 生成 Application；目标 cluster 带标签，例如 `region=apac`、`tier=prod`。
3. Git 中的 `base` 保存公共 Deployment/Rollout、Service、PDB、NetworkPolicy；每个区域 overlay 保存副本数、地域 endpoint、密钥引用和流量策略。
4. Argo CD 使用自动同步、自愈与 prune；生产目录使用受保护分支、双人审批和变更窗口。
5. `PreSync` Job 执行幂等、向后兼容的数据库迁移；失败则停止发布。
6. Rollouts 首先对一个区域 10% 放量，Prometheus 分析 5xx、授权失败率、延迟、数据库连接；人工批准后扩大，再发布另一地区。
7. Secret 使用 Vault/External Secrets，数据库迁移与应用运行采用不同 ServiceAccount 和最小权限。
8. Argo CD 组件部署多副本，控制器按集群或 Application 分片，配置 PDB、告警和灾备演练。

**可衡量的验收标准：**

| 目标 | 示例标准 |
|---|---|
| 可追溯性 | 每个生产运行镜像均可关联至 Git commit、PR、构建记录和签名。 |
| 发布安全性 | 金丝雀 10 分钟内 5xx < 0.5%、授权失败率未超过基线，才允许扩大流量。 |
| 恢复能力 | 从确认故障到 Git revert 合并并恢复稳定版本小于 15 分钟。 |
| 漂移控制 | 生产应用 5 分钟内检测到非授权资源变更，受管字段自动恢复或告警。 |
| 权限边界 | 业务 CI 不拥有目标集群写权限；团队不能部署到其他域 namespace。 |

---

## 六、面试速答清单

面试中可按以下顺序组织答案，避免只背概念：

1. **说明职责边界**：CI 构建/验证，Git 保存期望状态，Argo CD 持续部署和收敛，Kubernetes 负责运行。
2. **给出安全边界**：`AppProject`、OIDC/RBAC、集群最小权限、外部密钥、Git 审批和准入策略。
3. **给出发布控制**：不可变镜像 digest、PR 提升、Sync Hook/Wave、Argo Rollouts、指标门禁。
4. **说明异常处理**：先判断渲染、同步还是运行健康问题；通过 Git 回滚保证期望与实际一致。
5. **用业务指标收尾**：支付成功率、错误率、延迟、数据库连接、恢复时间和审计覆盖率。

### 高频追问

| 问题 | 一句话回答 |
|---|---|
| Argo CD 是否替代 CI？ | 不替代；它专注 Git 到 Kubernetes 的声明式持续交付。 |
| 为什么不让 CI 直接 `kubectl apply`？ | 降低生产凭据暴露，统一审计和 drift reconciliation。 |
| `prune` 是否一定危险？ | 它有删除能力，但配合单一所有者、审批、预生产验证和资源边界可以安全使用。 |
| HPA 为什么导致 OutOfSync？ | HPA 合法修改了 `spec.replicas`；需明确字段所有权并精确忽略该字段或不声明它。 |
| 紧急回滚该用什么？ | 首选 Git revert；若用 Argo CD 历史紧急回滚，随后必须同步修正 Git。 |
| 多集群如何避免权限过大？ | 每集群专用最小权限身份、项目目的地限制、网络隔离和分区控制平面。 |

---

## 推荐动手练习

1. 在 kind、minikube 或测试集群安装 Argo CD，并用一个 Kustomize Application 部署 NGINX。
2. 手工修改 Deployment 镜像，观察 `OutOfSync`，开启 `selfHeal` 后验证状态恢复。
3. 用 `AppProject` 限制一个团队只能部署到单一 namespace，并验证越权同步被拒绝。
4. 为 Deployment 加入 HPA，理解并正确处理 `spec.replicas` 的字段所有权。
5. 增加 `PreSync` 迁移 Job 和 `PostSync` smoke-test Job，模拟成功与失败路径。
6. 将普通 Deployment 替换为 Argo Rollouts 金丝雀发布，并接入 Prometheus 指标分析。

> 生产上线前应基于正在使用的 Argo CD、Kubernetes、Helm 和 Argo Rollouts 具体版本核对官方兼容矩阵与配置语法。本文的示例用于表达设计原则，不应在未经审查的情况下直接用于生产。
