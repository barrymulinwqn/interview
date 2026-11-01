# AWS Cloud Infrastructure 面试高频题与生产实战指南

> 适用岗位：Senior Cloud Engineer、Cloud Infrastructure Engineer、AWS Platform Engineer、DevOps/SRE。  
> 覆盖重点：多账户多 VPC 网络、IAM 治理、AWS 核心服务、Terraform/CloudFormation、CI/CD、Linux 生产排障。  
> 题目中的“AWS Cold build”通常指 **AWS CodeBuild**，本文按 CodeBuild 进行准备。  
> 裸金属和阿里云属于 preferred 能力，建议放在主线能力讲清楚之后作为加分项。

---

## 一、面试定位：面试官到底在验证什么

这类岗位不是考“能否背出 AWS 服务定义”，而是验证候选人能否把一项基础设施能力完整地交付并运营起来：

1. **设计**：为什么选择这个网络边界、路由、身份模型和服务组合。
2. **实施**：是否亲手使用 Terraform、CloudFormation、CLI 或控制台部署过。
3. **验证**：如何用命令、指标、日志和故障注入证明设计是有效的。
4. **运营**：出故障如何定位，如何回滚，如何控制成本和权限。
5. **治理**：如何让多个团队在不互相影响的情况下安全地交付。

### 1.1 Senior 级回答模板

几乎所有设计题和故障题都可以使用下面的结构：

```text
背景和约束 -> 目标和风险 -> 设计取舍 -> 实施步骤
-> 验证方法 -> 监控和告警 -> 故障/回滚方案 -> 复盘改进
```

例如被问到“如何让私有子网中的 EC2 访问 S3”，不要只回答“配置 NAT Gateway”。更完整的回答是：

> 如果访问目标主要是 S3，优先在每个需要访问的 VPC 中配置 S3 Gateway Endpoint，并通过 Endpoint Policy 限制允许的 bucket；这样流量不需要经过 NAT，降低成本并减少公网依赖。若还需要访问第三方公网地址，再为每个 AZ 配置 NAT Gateway，并让该 AZ 的私有路由表优先指向本 AZ NAT。验证时使用 `aws s3 cp`、VPC Flow Logs、CloudTrail 数据事件和 NAT/Endpoint 指标；如果失败，按 DNS、路由、Endpoint Policy、IAM、S3 Bucket Policy、KMS 权限逐层排查。

### 1.2 能力优先级

| 优先级 | 能力 | 面试中必须能够证明的内容 |
|---|---|---|
| P0 | AWS 网络 | 亲手设计过 VPC、Subnet、Route Table、TGW、NAT、Endpoint、SG，并能定位跨 VPC 不通 |
| P0 | IAM 治理 | 理解 Trust Policy、Permission Policy、SCP、Permission Boundary、跨账户 AssumeRole |
| P0 | IaC | Terraform state、module、provider alias、plan 审核、漂移、远程后端、回滚 |
| P0 | 生产运维 | Linux 指标和日志排查，能处理网络、权限、存储、节点失联等事故 |
| P1 | AWS 核心服务 | EC2、EBS、S3、CloudWatch、CloudTrail、Route 53、ELB/NLB、KMS、SSM |
| P1 | CI/CD | Jenkins、GitHub Actions、GitLab CI/CD、CodeBuild 的安全交付、审批和回滚 |
| P1 | 架构治理 | 多账户、共享服务、集中日志、成本分摊、合规和变更管理 |
| Preferred | 裸金属 | 驱动、固件、NUMA、GPU、网络、存储和故障隔离 |
| Preferred | 阿里云 | ECS、VPC、CEN、RAM、OSS、SLB、ActionTrail、CloudMonitor 的映射能力 |

---

## 二、多账户、多 VPC 网络设计

### 2.1 参考组织和网络拓扑

一个中大型企业通常不会把所有工作负载放在一个 AWS 账户和一个 VPC 中。一个可解释的基线如下：

```text
AWS Organization
|
|-- Security Account
|     |-- Organization CloudTrail
|     |-- Config Aggregator / Security Hub
|     |-- GuardDuty delegated administrator
|
|-- Log Archive Account
|     |-- S3 central log bucket
|     |-- CloudTrail / VPC Flow Logs / ALB logs
|
|-- Network Account
|     |-- Transit Gateway
|     |-- Network Firewall / inspection VPC
|     |-- Route 53 Resolver inbound/outbound endpoints
|     |-- Direct Connect / VPN termination
|
|-- Shared Services Account
|     |-- Directory / DNS / CI runners / artifact services
|
|-- Workload Accounts
      |-- Dev VPC
      |-- Test VPC
      |-- Prod VPC
      |-- Data / Analytics VPC

VPCs --(AWS RAM TGW attachments)--> Transit Gateway
                                      |
                                      |-- prod route table
                                      |-- non-prod route table
                                      |-- inspection route table
                                      |-- shared-services route table
```

关键不是“所有 VPC 都互通”，而是先定义通信矩阵：

| 来源 | 目标 | 是否允许 | 实现方式 | 记录与验证 |
|---|---|---:|---|---|
| Prod App | Prod DB | 是 | TGW 路由 + SG 引用/固定 CIDR | VPC Flow Logs、数据库连接指标 |
| Dev App | Prod DB | 默认否 | 不传播 prod 路由，必要时通过受控代理 | TGW 流日志、审批记录 |
| Workload VPC | S3 | 是 | S3 Gateway Endpoint + Endpoint Policy | CloudTrail Data Events |
| Workload VPC | 外部补丁源 | 受限允许 | NAT Gateway + egress allowlist | NAT 指标、代理日志 |
| On-prem | Shared DNS | 是 | Direct Connect/VPN + Resolver Endpoint | Resolver query logs |
| Internet | Private EC2 | 否 | 无 IGW 入站路径，使用 ALB/SSM | SG、NACL、Flow Logs |

### 2.2 高频题：为什么要拆多个账户和多个 VPC？

**推荐回答：**

- **账户是安全和计费边界**：生产、非生产、安全、日志、网络账户隔离，避免一个账户的管理员权限或配额事故影响全部环境。
- **VPC 是网络故障域和路由域**：不同应用、合规区域、环境或业务线使用不同 CIDR、路由表和安全策略。
- **最小互通原则**：通过 TGW 只发布必要路由，不因为“网络已经打通”就默认允许业务访问。
- **组织级治理**：用 SCP 禁止关闭 CloudTrail、禁止在不允许的区域创建资源，使用 StackSets、Terraform pipeline 或 Control Tower 建立基线。
- **代价**：账户和 VPC 增多后，CIDR、DNS、路由、日志、成本分摊和权限治理复杂度都会上升，因此需要网络账户、标准模块和自动化检查。

**真实场景示例：**

某企业将研发、测试和生产放在同一账户，研发人员拥有过大的权限，误改生产安全组导致业务中断。整改时把生产迁移到独立账户，通过 AWS Organizations/SCP 禁止生产账户关闭审计服务，开发人员只能从身份中心切换到受限的只读或运维角色；各环境 VPC 通过 TGW 进行明确的路由发布，而不是依赖大量 VPC Peering。

### 2.3 高频题：如何规划多 VPC 的 CIDR？

回答要覆盖“当前可用”和“未来可扩展”：

1. 先盘点已有 VPC、数据中心、分支机构、VPN/Direct Connect、合作方网段，确保没有重叠。
2. 按环境和区域分配大块地址，例如生产、非生产、共享服务分别保留连续网段。
3. 在 VPC 内按 AZ、层级和用途切分子网，不要把所有资源挤在一个 `/24`。
4. 预留扩容空间，考虑 EKS、RDS、接口型 Endpoint、第三方网络接入会大量消耗 ENI 和 IP。
5. 不能只考虑 AWS 内部，必须把企业 RFC1918 地址和未来并购/跨云网络纳入规划。
6. 通过 IPAM 管理地址池、分配记录和冲突检测，禁止团队随意申请 CIDR。

**易错点：** VPC Peering 和 TGW 都不支持通过路由“解决”重叠 CIDR；重叠网络通常需要重新编号、NAT、代理或隔离，而不是添加更具体的路由。

### 2.4 高频题：Transit Gateway 的 Association 和 Propagation 有什么区别？

- **Association**：一个 attachment 关联到一个 TGW route table，决定该 attachment 发出的流量查哪张 TGW 路由表。
- **Propagation**：把某个 attachment 的网段传播到一张或多张 TGW route table，使其他 attachment 可能学习到这些路由。
- **静态路由**：手工指定目标 CIDR 和 attachment，适合默认路由、黑洞路由或需要严格控制的特殊路径。
- **隔离的核心**：不是“所有 attachment 都挂上 TGW 就互通”，而是通过不同 TGW route table 的 association、propagation 和静态路由组合来实现分区。

**示例：**

```text
prod-vpc attachment -> prod-rt
nonprod-vpc attachment -> nonprod-rt
shared-services attachment -> shared-rt

prod-rt:
  10.50.0.0/16 -> prod VPC
  10.60.0.0/16 -> shared-services
  0.0.0.0/0   -> inspection VPC
  10.70.0.0/16 -> blackhole (禁止访问 nonprod)

nonprod-rt:
  10.60.0.0/16 -> shared-services
  10.70.0.0/16 -> nonprod VPC
```

**排障要点：**

1. 确认 VPC attachment 是 `available`。
2. 确认源 attachment 关联了预期 route table。
3. 确认目标 CIDR 已传播或存在静态路由。
4. 检查是否存在更具体的黑洞路由。
5. 检查两端 VPC route table 是否把目标 CIDR 指向 TGW。
6. 检查 SG、NACL、目标服务监听和回程路由。

### 2.5 高频题：TGW、VPC Peering、PrivateLink 怎么选？

| 机制 | 适合场景 | 优点 | 主要限制 |
|---|---|---|---|
| VPC Peering | 少量 VPC 的点到点互通 | 路径简单、延迟低 | 非传递路由，VPC 多时连接数和治理复杂 |
| Transit Gateway | 多账户、多 VPC、混合云中心网络 | 集中路由、分区和审计更容易 | 有小时费和数据处理费，需要严谨的路由设计 |
| PrivateLink | 向消费者暴露特定服务 | 服务级隔离，不暴露整个网络 | 不是通用网段互通，Endpoint/服务端配置和费用较多 |
| AWS RAM VPC Sharing | 同一组织内共享子网 | 减少 VPC 数量，集中网络管理 | 资源所有权、配额、权限和删除关系需要明确 |

**面试加分点：** 对外提供数据库、内部 API 或 SaaS 服务时，不要为了访问一个服务就开放整个 VPC 的路由。可以使用 PrivateLink，并在服务端通过 Endpoint Service 的 allowed principals 控制消费者账户。

### 2.6 高频题：跨账户 VPC Sharing 和 TGW Sharing 如何治理？

使用 AWS RAM 共享资源时，至少说明：

- 共享范围由 Organization、OU 或特定账户控制，必须使用 allow-list。
- 网络账户拥有 TGW、子网和路由等基础资源；工作负载账户拥有其业务资源。
- 明确谁能修改路由表、NACL、Endpoint、SG 和 DNS 记录。
- 共享资源删除有依赖关系，不能只考虑“创建”，还要设计拆除顺序。
- 通过 CloudTrail 记录 RAM、TGW、路由和安全组变更。
- 对生产共享资源使用 IaC，禁止个人在控制台做不可追溯变更。

### 2.7 高频题：NAT Gateway 和 VPC Endpoint 怎么选？

**NAT Gateway 适合：**

- 访问公网 API、软件仓库、第三方补丁源。
- 需要出站地址固定为 EIP，且对方做 IP allow-list。

**VPC Endpoint 适合：**

- 访问 S3、DynamoDB 等支持 Gateway Endpoint 的服务。
- 访问 ECR、SSM、CloudWatch Logs、STS 等支持 Interface Endpoint 的服务。
- 希望流量留在 AWS 网络内，减少公网依赖和 NAT 数据处理费。

**生产设计：**

- 每个 AZ 的私有子网优先使用同 AZ NAT，避免跨 AZ 数据处理费和单 AZ 故障。
- 只为确有访问需求的 VPC 和 AZ 创建 Interface Endpoint，统一设置 Endpoint Policy 和安全组。
- S3/DynamoDB Gateway Endpoint 要加入所有需要访问的私有路由表。
- 不要把 `0.0.0.0/0 -> NAT` 当成所有 AWS 服务的唯一出口。

**真实场景示例：**

某批处理集群每天从 S3 下载大量数据，私有子网通过 NAT 访问，月度 NAT 数据处理费用持续上升。改为 S3 Gateway Endpoint 后，S3 流量不再经过 NAT；同时保留 NAT 供系统访问外部补丁源，并使用 VPC Flow Logs 和 NAT 字节指标验证流量迁移。

### 2.8 高频题：Security Group 和 NACL 的区别

| 项目 | Security Group | Network ACL |
|---|---|---|
| 作用范围 | ENI/实例级 | 子网级 |
| 状态 | 有状态，允许回包 | 无状态，请求和回包都要放行 |
| 规则 | 只支持 allow | 支持 allow/deny，按规则号顺序匹配 |
| 典型用途 | 应用访问控制、最小来源 | 子网边界防护、显式拒绝、合规隔离 |
| 常见误区 | 认为放行出站就一定能通 | 忘记临时端口和回程规则 |

推荐做法：

- SG 使用“来源 SG”而不是大段 CIDR，例如 `app-sg -> db-sg:5432`。
- 不把 `0.0.0.0/0` 放到 SSH/RDP 入站，运维优先使用 SSM。
- NACL 规则变更要特别谨慎，保留临时端口和回程流量。
- 不把 SG 当作应用层认证；业务仍需 TLS、身份认证和授权。

### 2.9 高频题：Route 53 在多账户、混合云中如何设计？

推荐将 DNS 分为：

1. **公共 Hosted Zone**：互联网域名、证书验证和公共服务记录。
2. **私有 Hosted Zone**：内部服务域名，关联到指定 VPC。
3. **集中 Resolver**：网络账户提供 inbound endpoint 给数据中心查询 AWS 私有域名，outbound endpoint 将 AWS 查询转发到企业 DNS。
4. **Resolver Rule Sharing**：使用 RAM 共享规则给工作负载 VPC。
5. **查询日志**：集中发送到 CloudWatch Logs 或 S3，控制敏感域名数据访问权限。

排障 DNS 时区分：

- `dig` 是否命中预期 DNS；
- 私有 Hosted Zone 是否关联到正确 VPC；
- Resolver rule 是否关联且优先级正确；
- 安全组是否允许 UDP/TCP 53；
- 记录权重、延迟、故障转移策略是否符合预期；
- 应用是否缓存旧记录，TTL 是否过长。

### 2.10 高频题：跨 VPC 不通时你如何排查？

使用固定顺序，避免“先改安全组”的习惯：

```text
1. DNS：域名是否解析到正确的地址？
2. 路由：源子网 route table 是否有目标路由？
3. TGW/Peering/Endpoint：attachment、propagation、association 是否正确？
4. 回程：目标端到源端是否存在返回路由？
5. SG：源 ENI 出站、目标 ENI 入站是否允许目标端口？
6. NACL：两端子网的入站、出站和临时端口是否允许？
7. 目标服务：进程是否监听正确地址和端口，是否绑定到 localhost？
8. 主机防火墙：iptables/nftables/firewalld 是否拒绝？
9. 应用：TLS、认证、协议和超时是否正常？
10. 证据：Flow Logs、Reachability Analyzer、应用日志是否支持结论？
```

常用命令：

```bash
dig +short api.internal.example.com
ip route get 10.60.12.15
nc -vz -w 3 10.60.12.15 443
curl -vk --connect-timeout 3 https://api.internal.example.com/health
ss -lntp
sudo tcpdump -ni any host 10.60.12.15 and port 443
```

AWS 侧还应检查：

```bash
aws ec2 describe-route-tables --filters Name=vpc-id,Values=vpc-xxxxxxxx
aws ec2 describe-security-groups --group-ids sg-xxxxxxxx
aws ec2 describe-network-acls --filters Name=vpc-id,Values=vpc-xxxxxxxx
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id tgw-rtb-xxxxxxxx \
  --filters Name=route-search.exact-match,Values=10.60.12.15/32
```

**真实场景示例：**

应用 A 能访问应用 B 的域名，但 TCP 连接超时。排查发现 DNS 正常、源端 SG 正常，TGW 也有去程路由；最终在目标 VPC 的私有子网 route table 中漏配了返回源 VPC 的 TGW 路由。修复后又用 Reachability Analyzer 和双向 Flow Logs 验证，避免只凭“目标主机端口开放”下结论。

### 2.11 高频题：如何做集中网络安全和日志？

一种可落地的分层方案：

- **组织层**：Organization CloudTrail、AWS Config、GuardDuty、Security Hub。
- **网络层**：Network Firewall、TGW Flow Logs、VPC Flow Logs、Route 53 Resolver Query Logs。
- **负载均衡层**：ALB/NLB access logs、WAF logs。
- **主机层**：SSM Inventory、CloudWatch Agent、系统日志、EDR。
- **存储层**：S3 access logs 或 CloudTrail Data Events，按敏感 bucket 选择性开启。
- **中央日志账户**：S3 bucket 使用 Object Lock、KMS、生命周期和跨区域复制，日志账户对工作负载账户只允许写入不允许删除。

面试时强调：日志本身也是敏感数据，需要最小权限、加密、保留周期、检索权限和成本控制。

---

## 三、IAM 身份与权限治理

### 3.1 高频题：IAM User、Role、IAM Identity Center 如何选择？

- **人访问 AWS**：优先 IAM Identity Center，与企业 IdP 联邦，使用短期凭证和 Permission Set。
- **EC2、Lambda、CodeBuild 等 AWS 工作负载**：使用 IAM Role 和实例/服务身份，不在代码中保存 Access Key。
- **第三方 SaaS 或跨组织访问**：使用受限 Role，必要时使用 `sts:ExternalId`、Source Identity、Session Tags 和条件键。
- **长期 IAM User**：只保留给无法支持联邦或自动化的特殊场景，并强制 MFA、密钥轮换、告警和到期治理。

### 3.2 高频题：跨账户 AssumeRole 的最小权限链路

跨账户访问至少有两层策略：

```text
调用方账户:
  身份权限策略允许 sts:AssumeRole 到 arn:aws:iam::TARGET:role/DeployRole

目标账户:
  DeployRole 的 Trust Policy 信任调用方角色或组织条件
  DeployRole 的 Permissions Policy 允许实际资源操作
```

调用方角色示例：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AssumeOnlyApprovedDeploymentRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::123456789012:role/DeploymentRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "ci-prod-deploy"
        }
      }
    }
  ]
}
```

目标角色 Trust Policy 示例：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TrustCiRoleFromToolsAccount",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::210987654321:role/GitHubActionsProduction"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-exampleorg"
        },
        "StringLike": {
          "sts:SourceIdentity": "github-*"
        }
      }
    }
  ]
}
```

**注意：** `ExternalId` 主要用于第三方代表客户访问，不能把它当作唯一安全控制；还要限制 Principal、组织、会话时长、Source Identity 和具体资源。

### 3.3 高频题：Permission Policy、Trust Policy、SCP、Permission Boundary 的区别

| 机制 | 回答重点 |
|---|---|
| Identity/Resource Policy | 决定身份或资源允许什么操作 |
| Trust Policy | 决定谁可以对某个 Role 执行 `sts:AssumeRole` |
| SCP | Organizations 的权限上限，不授予任何权限 |
| Permission Boundary | 某个 User/Role 能拥有的权限上限 |
| Session Policy | AssumeRole 时进一步缩小当前会话权限 |
| Explicit Deny | 只要命中，通常会覆盖 Allow；排障要找 deny 来源 |

可用下面的逻辑说明权限最终结果：

```text
有效权限 =
Identity Allow
与 Resource Allow 的交集
再受 Permission Boundary、SCP、Session Policy 限制
并且不能命中显式 Deny
```

### 3.4 高频题：如何实现最小权限，而不是简单地“全部 Deny”？

推荐闭环：

1. 先按工作负载动作建模：读 S3、写 CloudWatch Logs、读取 KMS、调用 STS。
2. 用 CloudTrail、IAM Access Analyzer、服务最后访问信息观察真实调用。
3. 将资源限制到具体 ARN，加入 Region、标签、VPC Endpoint、Source Account 等条件。
4. 给开发者权限边界，防止其创建一个权限过大的角色。
5. 用 SCP 禁止组织级高风险操作，例如关闭审计、修改安全基线、使用不允许的区域。
6. 在 CI 中用 policy lint、IaC 扫描和 Access Analyzer 检查。
7. 定期删除未使用权限，并把紧急权限做成短时的 break-glass 流程。

**不要做：**

- 生产环境直接给 `AdministratorAccess`；
- 用 `Resource: "*"` 解决所有部署失败；
- 把权限错误简单归因于“缺少 S3 权限”，不检查 bucket policy、KMS、SCP 和 Endpoint Policy；
- 让 CI/CD 使用个人 Access Key。

### 3.5 高频题：Permission Boundary 什么时候最有用？

场景是“允许团队自助创建角色，但不能创建可越权的角色”。例如平台团队为开发账户设置 Boundary，允许业务角色访问自己的 S3 前缀、日志和部分部署服务，但 Boundary 不允许：

- 删除审计资源；
- 修改组织安全配置；
- 给其他角色授予 `AdministratorAccess`；
- 修改生产账户的 IAM 或 KMS 管理策略。

**面试加分点：** Boundary 不是给角色授权；如果身份策略没有 Allow，即使 Boundary 允许也不能执行。Boundary 只限制上限。

### 3.6 高频题：为什么 S3/KMS 跨账户访问经常失败？

跨账户访问常常需要同时满足：

1. 调用方 Role 的身份策略允许 S3/KMS 动作。
2. 目标 S3 Bucket Policy 或 KMS Key Policy 信任调用方账户/角色。
3. KMS key 使用场景允许对应动作，例如 `kms:Decrypt`、`kms:GenerateDataKey`。
4. SCP、Endpoint Policy、VPC Endpoint、网络和资源条件没有拒绝。
5. 资源 ARN、Region、对象前缀和加密上下文正确。

排障思路：

```bash
aws sts get-caller-identity
aws s3api head-object --bucket example-prod-data --key reports/a.csv
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=Decrypt
```

使用 IAM Policy Simulator 时要注意，它不一定覆盖所有资源策略、SCP、Endpoint Policy 和运行时条件，最终仍需结合 CloudTrail 和实际最小化测试。

### 3.7 高频题：如何设计生产 break-glass 访问？

- 建立单独的 break-glass Role 或账户，不用于日常操作。
- 强制 MFA、硬件密钥或企业 IdP 强认证，限制会话时长。
- 权限预先审核，默认不分配给普通用户。
- 使用工单、审批、双人复核和短期授权。
- CloudTrail、CloudWatch Alarm、Security Hub 对 AssumeRole 告警。
- 事后自动检查使用者、命令、资源和变更结果。
- 不把静态 root Access Key 放在任何 pipeline 或主机上。

### 3.8 IAM 真实场景题：部署失败提示 AccessDenied，如何定位？

**推荐回答：**

1. 用 `aws sts get-caller-identity` 确认当前身份和账户，排除凭据串用。
2. 看具体 API、资源 ARN、Region 和请求条件，而不是只看“权限不足”。
3. 检查身份策略是否 Allow，再检查目标资源策略。
4. 检查 Permission Boundary、SCP、Session Policy 是否有隐式或显式限制。
5. 如果经过 Endpoint，检查 Endpoint Policy 和网络。
6. 对 KMS/S3 尤其检查 key policy、bucket policy 和加密上下文。
7. 用 CloudTrail 找到真实失败事件和 `errorCode`。
8. 修复时添加单一、可解释的最小权限，不直接给管理员权限。

**真实场景示例：**

Terraform pipeline 可以创建 S3 bucket，却不能开启默认 KMS 加密。最初团队只给了 S3 权限；CloudTrail 显示失败 API 是 `kms:CreateGrant`。最终将部署 Role 限制到指定 KMS Key，并在 Key Policy 中信任部署角色，验证创建、读写和销毁流程后再合并变更。

---

## 四、AWS 核心服务高频面试题

### 4.1 EC2、AMI、ASG 和实例生命周期

**问题：如何设计一个可扩展且可回滚的 EC2 服务？**

推荐回答：

- 使用 Golden AMI 或启动脚本构建不可变实例，避免每台机器手工配置不同。
- 用 Launch Template 固化 AMI、实例类型、IMDSv2、网络接口、SG、EBS 加密和标签。
- 用 Auto Scaling Group 跨多个 AZ，配置健康检查、实例刷新和容量边界。
- 前面使用 ALB，应用通过 Target Group 健康检查摘除异常实例。
- 发布使用新 Launch Template 版本，先小比例验证，再滚动替换。
- 启动失败查看 EC2 console output、SSM、CloudWatch Agent、user-data 日志。
- 回滚是恢复旧 Launch Template/AMI，而不是在实例上手工改回去。

**高频追问：**

- 为什么不把数据写在实例本地？因为实例可被替换，持久数据应放在 EBS、S3、RDS 或其他托管存储。
- IMDSv2 解决什么问题？通过 session-oriented token 降低 SSRF 直接获取实例元数据凭证的风险。
- ASG 健康检查为什么会误判？EC2 状态健康不等于应用健康，应配置 ELB 健康检查并合理设置宽限期。

### 4.2 EBS：容量、吞吐、IOPS 和数据保护

面试时要区分：

- **容量**：磁盘大小。
- **IOPS**：每秒随机 I/O 操作数。
- **吞吐**：每秒传输 MiB 数，顺序读写更关注。
- **延迟**：应用感知的单次操作时间。
- **Burst**：某些卷型存在突发能力，不能当成长期保证。

排障命令：

```bash
lsblk -f
df -hT
df -ih
iostat -xz 1 5
sudo nvme list
sudo dmesg -T | egrep -i 'nvme|xfs|ext4|I/O error|timeout'
```

生产设计：

- 关键卷开启 KMS 加密，快照策略和跨区域复制按 RPO/RTO 设计。
- 文件系统扩容后要分别完成块设备和文件系统扩容。
- 不要只看 `df`，inode 用尽时也会表现为无法写入。
- 对数据库和高并发应用根据 IOPS/吞吐/延迟压测，而不是只按容量选型。
- EBS 快照是增量的，但第一次快照和恢复时的预热行为要纳入恢复时间测试。

### 4.3 S3：安全、可靠性和性能

**高频题：如何设计生产 S3 bucket？**

- Block Public Access 默认开启。
- 使用 IAM、Bucket Policy、VPC Endpoint Policy 和必要的组织条件控制访问。
- 默认 SSE-KMS 或 SSE-S3，敏感数据使用客户管理 KMS Key。
- 开启版本控制，按数据分类配置生命周期、归档和删除保护。
- 关键 bucket 启用 Object Lock/WORM，集中审计日志写入 Log Archive。
- 上传使用 multipart、校验和、幂等对象 key；大规模数据访问考虑前缀和并发策略。
- 限制跨账户写入的 `aws:PrincipalOrgID`、Source Account、Source Arn 和对象前缀。

**S3 AccessDenied 排查顺序：**

```text
Caller identity -> IAM policy -> Bucket policy -> KMS key policy
-> Endpoint policy -> SCP/Boundary -> 资源 ARN/Region/前缀
```

### 4.4 CloudWatch、CloudTrail 和审计闭环

| 服务 | 主要回答 |
|---|---|
| CloudWatch Metrics | 系统和服务现在是否健康 |
| CloudWatch Logs | 进程、应用、系统日志发生了什么 |
| CloudWatch Alarms | 是否达到告警阈值，需要什么动作 |
| CloudTrail | 谁在什么时间调用了什么 AWS API，结果如何 |
| AWS Config | 资源配置是否符合规则，何时发生偏离 |
| VPC Flow Logs | 某个网络接口的流量是否被接受/拒绝 |

**真实场景示例：**

生产实例 CPU 正常但请求延迟上升。CloudWatch 显示 EBS queue length 和 `VolumeQueueLength` 上升，主机 `iostat -xz` 显示 await 和 `%util` 接近上限；CloudTrail 同时显示没有网络变更，因此将根因定位为卷吞吐不足，而不是先扩大实例或修改 SG。

### 4.5 Route 53、ALB 和 NLB 如何选

**ALB：**

- HTTP/HTTPS 七层路由；
- Host/path/header 路由、TLS 终止、重定向；
- 适合 Web API、微服务入口；
- 依赖目标组健康检查。

**NLB：**

- TCP/UDP/TLS 四层转发；
- 高吞吐、低延迟、静态 IP/EIP、需要保留连接语义的场景；
- 适合非 HTTP 协议、数据库代理、极高连接量；
- 应用层错误不一定能被 L4 健康检查发现。

**Route 53：**

- 简单、加权、延迟、故障转移、地理位置和多值回答策略；
- DNS 切换不是即时的，TTL、客户端缓存和连接复用要纳入故障演练；
- 灰度发布可使用加权记录和逐步调整权重。

### 4.6 KMS：加密设计和密钥治理

面试要说清楚：

- KMS Key Policy 和 IAM Policy 的组合授权；
- key admin、key user、应用运行角色分离；
- 自动轮换不等于历史密文自动重新加密；
- 备份、跨账户、跨区域和灾备需要明确 key 使用策略；
- CloudTrail 记录 `Encrypt`、`Decrypt`、`GenerateDataKey` 等调用；
- 不把明文密钥、数据密钥或 KMS 管理权限写入应用代码。

应用访问 KMS 失败时，检查：

```text
应用 Role 是否有 kms:Decrypt
-> Key Policy 是否信任该 Role
-> 密钥 ARN/Region 是否正确
-> Encryption Context 是否匹配
-> SCP、Endpoint Policy 和资源条件是否拒绝
```

### 4.7 Systems Manager：生产运维的优先入口

相比开放 SSH，SSM 的优势包括：

- 不需要给实例分配公网 IP；
- 通过 IAM、MFA、Session Manager 记录访问；
- 支持 Run Command、Patch Manager、Automation、Inventory、Parameter Store；
- 可用 Port Forwarding 访问私有数据库而不开放入站端口。

前提包括：

- SSM Agent 正常运行；
- 实例 Instance Profile 有所需权限；
- 实例能够访问 SSM、SSMMessages、EC2Messages 的公网或 Interface Endpoint；
- DNS、路由和 Endpoint SG 正常；
- 操作系统时间、证书和代理配置正常。

常用命令：

```bash
aws ssm describe-instance-information
aws ssm start-session --target i-0123456789abcdef0
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --targets Key=InstanceIds,Values=i-0123456789abcdef0 \
  --parameters commands='["uname -a","df -h","systemctl --failed"]'
```

### 4.8 ELB/EC2 真实故障题：Target unhealthy 怎么办？

1. 查看 Target Group 的具体 health reason，而不是只看“不健康”。
2. 从实例内部 `curl localhost:port/health`，确认服务监听和返回码。
3. 检查实例 SG 是否允许来自 ALB SG 的健康检查端口。
4. 检查 NACL、子网路由、主机防火墙和服务绑定地址。
5. 确认 ALB 使用的协议、端口、Host header、TLS 证书和健康路径。
6. 检查应用启动时间与 ASG health check grace period。
7. 用 ALB access logs、TargetResponseTime、HTTPCode_Target_5XX_Count 定位应用还是网络问题。

---

## 五、Terraform 与 CloudFormation

### 5.1 Terraform State 高频题

**State 的作用：**

- 记录配置和实际资源之间的映射；
- 允许 Terraform 计算 plan；
- 保存资源 ID、依赖关系和部分属性；
- 不是普通缓存，也不应被当作可随意编辑的业务数据。

生产远程后端基线：

- S3 backend，开启版本控制和默认加密；
- 使用支持版本的 state locking 机制；新版本可使用 S3 lockfile，旧环境可能仍使用 DynamoDB；
- 限制 state bucket 的访问，禁止公共访问；
- state 中可能有敏感值，按敏感数据处理；
- 通过单一 pipeline 执行生产 apply，避免多人本地并发；
- 对 state 变更和解锁建立审计流程。

示例：

```hcl
terraform {
  backend "s3" {
    bucket       = "company-tfstate-prod"
    key          = "network/prod/terraform.tfstate"
    region       = "ap-southeast-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

### 5.2 高频题：Terraform state 被锁住或损坏怎么办？

1. 先确认是否真的有另一个 apply 在运行，不能直接强制解锁。
2. 查看锁信息、执行者、时间和 pipeline 状态。
3. 如果确认是已失效的锁，再按组织流程使用 `terraform force-unlock`。
4. 不手工删除 state，不用 `rm` 破坏锁文件。
5. 如果 state 被误改，使用 S3 版本恢复前先复制当前版本并评估影响。
6. 恢复后执行 `terraform plan`，逐项核对是否会重建生产资源。

### 5.3 Module 设计和目录结构

推荐分离“可复用模块”和“环境组合”：

```text
terraform/
  modules/
    vpc/
    tgw-attachment/
    iam-deployment-role/
    ec2-service/
  live/
    dev/
      network/
      application/
    prod/
      network/
      application/
  policies/
  tests/
```

Module 设计原则：

- 输入变量有类型、描述、校验和合理默认值；
- 输出只暴露下游真正需要的值；
- 不在 module 内硬编码账户、Region、CIDR 和业务名称；
- 资源命名、标签、加密和审计策略保持组织标准；
- 用 `for_each` 处理有稳定 key 的集合，避免 `count` 删除中间元素引起地址变化；
- 对破坏性资源谨慎使用 `prevent_destroy`，不要让它掩盖正确的变更流程；
- Provider 版本、模块版本和镜像版本锁定并定期升级。

### 5.4 高频题：Terraform provider alias 如何部署多账户资源？

```hcl
provider "aws" {
  region = var.home_region
}

provider "aws" {
  alias  = "network"
  region = var.home_region

  assume_role {
    role_arn = "arn:aws:iam::210987654321:role/TerraformNetwork"
  }
}

module "tgw" {
  source = "../../modules/tgw"

  providers = {
    aws = aws.network
  }

  name = "org-tgw"
}
```

面试追问：

- Terraform 执行身份如何进入网络账户；
- 每个 provider alias 的权限和账户如何验证；
- state 是否按账户/区域/生命周期拆分；
- plan 中是否可能误操作错误账户；
- provider 凭据是否来自 OIDC/短期 STS，而不是静态密钥。

### 5.5 高频题：如何做 Terraform 变更审核？

推荐 pipeline：

```text
Pull Request
  -> fmt / validate
  -> tflint / checkov 或组织安全扫描
  -> terraform plan
  -> plan 文件作为不可变 artifact
  -> reviewer 审核资源、权限、网络和成本
  -> 合并受保护分支
  -> 重新生成 plan
  -> 人工审批生产 apply
  -> apply 后输出和漂移检查
```

不要直接把 PR 中生成的任意 artifact 交给生产 apply，因为代码、变量或 state 可能已改变。生产应该在受保护环境中重新生成并审核 plan。

### 5.6 高频题：如何处理漂移？

**发现：**

- 定期运行只读 `terraform plan`；
- CloudTrail、Config、资源标签和平台基线发现控制台变更；
- 对生产资源开启必要的配置规则和告警。

**处理：**

1. 确认漂移是否是紧急修复或未经批准的改动。
2. 评估保留配置变更还是恢复 IaC 声明。
3. 如果资源本来就存在，使用 `terraform import` 或新的 import block 纳入 state。
4. 更新代码和模块，避免下次 plan 再次反向修改。
5. 对高风险资源先做 plan、备份和回滚演练。

`terraform refresh` 不应被当作“修复漂移”的万能命令；它只会刷新 state，不会替你决定期望配置，也可能让后续 plan 的行为更难理解。

### 5.7 CloudFormation 高频题

要能解释以下概念：

- Stack：一组 CloudFormation 管理的资源。
- Change Set：应用前预览变更，适合审核 replacement、删除和权限变化。
- StackSet：跨账户/跨区域批量部署组织基线。
- Nested Stack：拆分模板，适合复用和控制模板复杂度。
- Drift Detection：发现资源与模板期望的差异，但不是所有属性都支持完全检测。
- Rollback：资源创建或更新失败后的回滚；需要考虑不可回滚资源和保留策略。
- `DeletionPolicy` / `UpdateReplacePolicy`：数据库、日志桶等资源的删除和替换保护。

**CloudFormation 与 Terraform 怎么选？**

| 维度 | Terraform | CloudFormation |
|---|---|---|
| 多云/多厂商 | 生态更广 | AWS 原生 |
| AWS 新服务支持 | 取决于 provider 发布 | 通常与 AWS 原生资源支持紧密 |
| State | 外部 state，需自行治理 | AWS 托管 stack 状态 |
| 组织基线 | module + pipeline | StackSets、Organizations 集成好 |
| 团队现状 | 看现有技能和模块资产 | 看 AWS 原生治理和审批 |

高质量回答不是说某个工具“永远更好”，而是根据组织标准、团队能力、资源范围、state 治理和审批要求选择。

### 5.8 IaC 真实场景：网络模块改动导致大规模重建

**推荐回答：**

1. 先停止 apply，保存 plan 和 state 版本。
2. 检查是否因为资源地址变化、`count` 顺序变化、模块重构或 Provider 升级造成 diff。
3. 使用 `moved` block 或 `terraform state mv` 保留资源地址，而不是销毁重建。
4. 对路由、NAT、TGW、数据库等资源使用分阶段变更。
5. 在非生产账户应用同一 plan 结构并做连通性验证。
6. 生产执行前审核所有 `destroy` 和 `replace`，准备恢复路径。

---

## 六、CI/CD：Jenkins、GitHub Actions、GitLab CI/CD、AWS CodeBuild

### 6.1 统一的安全流水线模型

```text
代码提交
  -> 依赖/源代码扫描
  -> 单元测试与 lint
  -> 构建不可变 artifact
  -> 镜像扫描并推送 ECR
  -> Terraform plan / CloudFormation change set
  -> 审批
  -> 部署到 dev
  -> 自动验证
  -> staging / production
  -> 健康检查、指标验证、回滚
```

核心原则：

- PR 阶段不授予生产写权限；
- 使用 OIDC/短期 STS，而不是把 AWS Access Key 放在 CI Secret；
- 依赖版本和构建镜像锁定，生成 SBOM 和 provenance；
- artifact 以 digest/版本号引用，不使用可变的 `latest`；
- 生产 apply/deploy 只能由受保护分支、审批和具名环境触发；
- pipeline 日志不得输出 Secret、Token、私钥和敏感业务数据；
- 每次部署有 deployment ID，能关联 commit、artifact、审批人、CloudTrail 和监控。

### 6.2 工具选型题

| 工具 | 适合场景 | 关键治理点 |
|---|---|---|
| Jenkins | 企业已有大量插件、复杂流程、私有网络 runner | Controller/Agent 隔离、插件升级、凭据管理、agent 清理 |
| GitHub Actions | 代码在 GitHub、需要 PR 原生集成 | OIDC trust 条件、环境保护、第三方 Action 固定版本 |
| GitLab CI/CD | 代码、Registry、Runner、审批集中在 GitLab | Runner 隔离、protected variables、环境和权限 |
| AWS CodeBuild | AWS 内部构建、私有 VPC、ECR/CodePipeline 集成 | service role 最小权限、日志和缓存、构建镜像治理 |

回答“哪个最好”会显得片面。应说明现有代码托管、网络位置、审计、团队技能、插件维护、构建并发和多云需求。

### 6.3 GitHub Actions OIDC 示例

```yaml
name: terraform-plan

on:
  pull_request:
    paths:
      - "terraform/**"

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsTerraformPlan
          aws-region: ap-southeast-1

      - name: Verify identity
        run: aws sts get-caller-identity

      - name: Terraform plan
        working-directory: terraform/live/dev
        run: |
          terraform init -input=false
          terraform plan -input=false -out=tfplan
```

对应 IAM Trust Policy 要限制：

- `aud` 是 `sts.amazonaws.com`；
- `sub` 只允许特定 repository、branch 或 environment；
- 生产 Role 不信任任意 PR 分支；
- 通过 GitHub Environment protection 和 required reviewers 控制生产部署。

### 6.4 Jenkins 高频题：如何避免 Jenkins 成为单点和高权限跳板？

- Controller 不直接执行不可信构建，使用短生命周期 Agent。
- Agent 按 trusted/untrusted、网络区域和工具链隔离。
- Jenkins 凭据存储使用 Credentials Binding 或外部 Secret Manager，禁止写入 Jenkinsfile 明文。
- AWS 访问使用实例 Role、OIDC 或短期 AssumeRole。
- 对插件做版本清单、漏洞扫描和定期升级。
- pipeline 代码纳入 PR review，禁止任意用户修改生产 job。
- 构建产物写入受控 Artifact Repository，部署按 digest/版本引用。
- Controller、Agent、Artifact、日志和备份分别监控。

### 6.5 GitLab CI/CD 高频题

需要能够解释：

- `protected` branch/tag 防止未授权生产部署；
- `protected variables` 防止普通 pipeline 读取生产凭据；
- Runner 使用 ephemeral、隔离网络和最小权限；
- `needs` 优化 DAG，避免所有 job 串行；
- artifacts 和 cache 的生命周期区别；
- environment、manual job、approval rule 与回滚关系；
- Merge Request pipeline 不应继承生产部署权限。

### 6.6 AWS CodeBuild 高频题

CodeBuild 运行时至少涉及：

- Project 的 service role；
- build image、计算类型和超时；
- VPC 配置及其子网、SG、NAT/Endpoint；
- CloudWatch Logs、S3 artifact、缓存；
- Parameter Store/Secrets Manager 的读取权限；
- `buildspec.yml` 中 install/pre_build/build/post_build 阶段。

示例：

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.12
  pre_build:
    commands:
      - set -euo pipefail
      - aws sts get-caller-identity
      - aws ecr get-login-password | docker login --username AWS --password-stdin "$ECR_REGISTRY"
  build:
    commands:
      - docker build --pull -t "$IMAGE_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION" .
      - docker push "$IMAGE_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION"
  post_build:
    commands:
      - printf '{"imageUri":"%s"}' "$IMAGE_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION" > image-definitions.json

artifacts:
  files:
    - image-definitions.json
```

如果 CodeBuild 放在私有子网，排障优先检查：

```text
DNS -> 路由 -> NAT 或 Interface Endpoint
-> Endpoint SG -> CodeBuild service role
-> ECR/S3/CloudWatch Logs/KMS 权限
```

### 6.7 部署策略和回滚题

| 策略 | 适用场景 | 风险 |
|---|---|---|
| Rolling | 普通 EC2/ASG 更新 | 新旧版本同时存在，需保证兼容 |
| Blue/Green | 需要快速切换和回滚 | 资源成本和数据库兼容要求更高 |
| Canary | 高风险变更、逐步放量 | 需要可靠指标和自动停止 |
| Feature Flag | 功能与部署分离 | Flag 生命周期和权限治理复杂 |

回滚不能只回答“重新部署旧版本”，还要说明：

- 旧 artifact/AMI 是否仍保留；
- 数据库 schema 是否向后兼容；
- 流量切换依据什么指标；
- 如何停止错误 rollout；
- 回滚后如何验证连接、错误率、延迟和业务交易；
- 如何保留事故现场和变更审计。

### 6.8 CI/CD 真实场景：部署后 5xx 上升

**推荐回答：**

1. 先停止继续放量，保留当前版本和日志。
2. 按 deployment ID 对比新旧 Target Group、实例、容器和应用版本。
3. 查看 ALB `HTTPCode_Target_5XX_Count`、TargetResponseTime、应用错误日志和依赖服务指标。
4. 如果是代码回归，切回旧 artifact；如果是配置/Schema，先确认回滚兼容性。
5. 验证核心交易、健康检查和错误率恢复。
6. 复盘为何自动测试、canary 指标或审批没有拦截。

---

## 七、裸金属服务器与 GPU 运维（Preferred）

### 7.1 裸金属 EC2 与普通虚拟化 EC2 的区别

| 维度 | 普通 EC2 | Bare Metal EC2 |
|---|---|---|
| CPU/NUMA | 通过虚拟化呈现，拓扑视实例类型 | 操作系统更直接看到物理 CPU/NUMA 拓扑 |
| Hypervisor | 使用 Nitro 等虚拟化能力 | 无传统 guest hypervisor 层，但仍由 AWS 平台管理 |
| 驱动 | 主要关注虚拟设备和内核兼容 | 更关注 PCIe、NIC、GPU、NVMe、固件和内核 |
| 启动时间 | 通常较快 | 可能更慢，容量和回收行为要纳入设计 |
| 故障域 | 实例级虚拟化隔离 | 物理硬件故障和资源占用影响更直接 |
| 性能 | 稳定，适合大多数工作负载 | 适合许可绑定、低延迟、特殊硬件或虚拟化嵌套场景 |
| 运维方式 | SSM、CloudWatch、自动替换 | 同样优先 SSM/自动化，但需额外检查硬件可见性 |

**重要边界：** AWS Bare Metal 不等于传统机房裸机。通常仍通过 EC2 API、SSM、CloudWatch 和 AWS 工单管理底层硬件，不能假设可以使用传统服务器的 BMC/IPMI 进行主板级操作。

### 7.2 裸金属高频题：如何排查性能突然下降？

1. 检查实例类型、AZ、事件和底层维护通知。
2. 检查 NUMA 和 CPU 拓扑：

```bash
lscpu
numactl --hardware
numastat -p <pid>
```

3. 检查 PCIe、网卡、NVMe 和错误计数：

```bash
lspci -nn
sudo dmesg -T | egrep -i 'pcie|aer|nvme|mlx|ixgbe|error|timeout'
sudo nvme list
sudo ethtool -S eth0 | egrep -i 'drop|error|timeout|crc'
```

4. 检查 CPU steal、IRQ、软中断、内存带宽和 I/O：

```bash
vmstat 1 5
mpstat -P ALL 1 5
pidstat -dru 1 5
iostat -xz 1 5
```

5. 对高性能应用确认进程是否跨 NUMA 访问远端内存，是否发生 CPU 绑核或 IRQ 不均衡。
6. 通过同版本 AMI 在另一台实例复现，区分软件、配置和硬件宿主问题。
7. 保留诊断包和 CloudWatch 指标，必要时联系 AWS Support，而不是直接重启丢失证据。

### 7.3 GPU 高频题：`nvidia-smi` 不正常怎么办？

```bash
nvidia-smi
lsmod | grep nvidia
lspci | grep -i -E 'nvidia|amd'
sudo dmesg -T | egrep -i 'nvrm|xid|gpu|pcie'
systemctl status nvidia-persistenced
```

排查顺序：

1. GPU 是否被操作系统识别；
2. 驱动版本是否与内核、CUDA、容器运行时兼容；
3. 是否有 NVIDIA Xid 错误或 PCIe 错误；
4. GPU 是否被其他进程占用、显存耗尽或温度/功耗受限；
5. 容器是否正确挂载 GPU（例如 NVIDIA Container Toolkit）；
6. 应用、CUDA、驱动和 AMI 是否来自兼容矩阵；
7. 新实例和旧实例是否表现一致；
8. 只有在采集日志后，才考虑重载驱动、重启或替换实例。

### 7.4 裸金属面试的稳妥回答

如果候选人实际接触较少，可以诚实回答：

> 我在普通 EC2、VPC、EBS、SSM 和 Linux 生产排障方面有完整经验。裸金属方面我重点理解 CPU/NUMA、PCIe、NVMe、网卡和 GPU 驱动层次，知道它和普通虚拟化实例的排查边界；遇到硬件/固件级问题会先保留证据、用同 AMI 做对照，并通过 AWS 的实例事件和 Support 流程处理，而不会把所有问题都归因于应用。

---

## 八、Linux 生产运维与故障排查

### 8.1 CPU 高：不要只看 load average

```bash
uptime
top -H -o %CPU
mpstat -P ALL 1 5
pidstat -u -r -d 1 5
ps -eo pid,ppid,stat,pcpu,pmem,comm --sort=-pcpu | head
```

判断维度：

- user 高：应用计算或忙循环；
- system 高：系统调用、网络、I/O、驱动；
- iowait 高：存储或依赖 I/O；
- steal 高：虚拟化 CPU 争用或实例类型压力；
- load 高但 CPU 不高：不可运行任务可能在等待 I/O；
- 单核打满：线程/中断/绑核问题，不能只看总体平均值。

处理方式是先识别进程、线程和请求来源，再结合应用 profiling、GC、数据库和 I/O 指标，不要直接 kill 或扩容。

### 8.2 内存不足、Swap 和 OOM

```bash
free -h
vmstat 1 5
sar -r 1 5
ps -eo pid,pmem,rss,vsz,comm --sort=-rss | head
dmesg -T | egrep -i 'oom|out of memory|killed process'
journalctl -k --since "30 min ago"
```

要区分：

- 进程真正泄漏；
- page cache 正常占用；
- cgroup/container memory limit；
- Swap 抖动；
- 内核 OOM killer；
- 单进程地址空间或文件句柄问题。

### 8.3 磁盘满、inode 满和 I/O 慢

```bash
df -hT
df -ih
du -xhd1 /var | sort -h
sudo lsof +L1
iostat -xz 1 5
find /var/log -type f -size +1G -ls
```

常见根因：

- 日志轮转失效；
- 已删除但仍被进程打开的文件；
- inode 用尽；
- 容器 overlay2 占满；
- EBS IOPS/吞吐不足；
- 文件系统只读；
- 快照、备份或应用批处理造成突发 I/O。

扩容步骤要区分：

```text
EBS volume 修改大小 -> OS 识别新块设备
-> 分区扩容（如需要） -> 文件系统扩容
-> df 验证 -> 业务写入验证
```

### 8.4 网络不通、连接重置和连接耗尽

```bash
ip addr
ip route
ip rule
ss -s
ss -lntp
ss -ant state time-wait | wc -l
resolvectl status
dig +trace example.com
curl -v --connect-timeout 3 https://example.com
sudo tcpdump -ni any host 10.0.10.25 and port 443
```

区分：

- DNS 解析失败；
- 无路由或错误路由；
- SYN 发出但无 SYN-ACK：网络、SG、NACL、目标监听；
- TCP 建立后 reset：应用、代理、TLS 或中间设备；
- 连接数耗尽：`ulimit -n`、conntrack、线程池、连接池；
- 只有大包失败：MTU、PMTUD、VPN/TGW 或防火墙问题。

### 8.5 进程、服务和系统日志

```bash
systemctl status myservice
systemctl list-units --failed
journalctl -u myservice --since "1 hour ago"
journalctl -p err..alert --since today
ps -ef
pstree -ap <pid>
lsof -p <pid>
cat /proc/<pid>/limits
```

排障时记录：

- 发生时间（统一时区）；
- 主机、实例 ID、AZ、版本和部署 ID；
- 首个错误而不是最后一个连锁错误；
- 是否能在另一台同版本实例复现；
- 变更前后的指标和日志；
- 采取的每一个动作及结果。

### 8.6 节点失联 Runbook

```text
1. 检查 CloudWatch StatusCheckFailed、EC2 system/instance status
2. 检查实例事件、自动恢复/替换策略
3. 尝试 SSM，区分网络/SSH 问题还是主机完全失联
4. 检查最近部署、内核、驱动、磁盘和安全组变更
5. 检查 VPC 路由、NACL、SG、ENI 和 DHCP/DNS
6. 若仍可访问，收集 dmesg、journal、磁盘和网络诊断
7. 若不可访问，使用自动化恢复、快照/AMI、替换实例
8. 验证负载均衡健康、依赖连接、数据一致性和告警恢复
```

### 8.7 权限拒绝 Runbook

```bash
id
groups
namei -l /path/to/file
getfacl /path/to/file
ls -Z /path/to/file        # SELinux 环境
sudo -l
findmnt -T /path/to/file
```

重点区分：

- Unix owner/group/mode；
- POSIX ACL；
- SELinux/AppArmor；
- 文件系统只读或挂载参数；
- NFS/EFS 身份映射；
- 应用使用的实际用户与 shell 用户不同；
- AWS 侧 IAM/S3/KMS 权限与主机权限是两层问题。

### 8.8 生产排障题的回答原则

成熟回答应包括：

- 先降低影响：停止放量、摘除故障节点、保护数据；
- 保留证据：日志、指标、进程、网络抓包和变更记录；
- 从现象到假设，再用可证伪的检查验证；
- 每次只改变一个关键变量；
- 先恢复服务，再做根因修复；
- 事后补充监控、自动化检查、Runbook 和演练。

---

## 九、阿里云能力（Preferred）

### 9.1 AWS 与阿里云服务映射

| AWS | 阿里云 | 面试回答重点 |
|---|---|---|
| EC2 | ECS | 实例规格、镜像、弹性伸缩、云盘和安全组 |
| VPC | VPC | 专有网络、交换机、路由表、NAT 网关 |
| Transit Gateway | CEN/Transit Router | 多地域、多 VPC、跨账号/专有网络互联 |
| IAM/STS | RAM/STS | RAM 用户、角色、策略、角色扮演和临时凭证 |
| S3 | OSS | Bucket、RAM、访问策略、KMS、生命周期 |
| ALB/NLB | ALB/NLB/SLB | 七层/四层负载均衡、健康检查、监听器 |
| CloudTrail | ActionTrail | API 审计、集中投递、保留和检索 |
| CloudWatch | CloudMonitor/日志服务 | 指标、日志、告警和自动化 |
| KMS | KMS | 密钥、权限、轮换、审计和跨账户使用 |
| Systems Manager | 云助手/运维编排/OOS | 远程运维、补丁、命令执行和资产管理 |

### 9.2 多云面试题：如何避免只会“服务名称映射”？

回答要从能力抽象出发：

- 网络：地址规划、路由传播、跨地域、专线/VPN、DNS、出口控制；
- 身份：人和工作负载身份、短期凭证、权限边界、审计；
- 计算：不可变镜像、自动扩缩容、健康检查和故障替换；
- 存储：加密、备份、生命周期、跨区域复制和恢复演练；
- 交付：IaC、模块、远程 state、变更审批和漂移；
- 运维：指标、日志、追踪、Runbook、告警和成本。

### 9.3 阿里云真实场景示例

如果需要把 AWS 的生产 VPC 与阿里云上的业务网络互通，回答可包含：

1. 先确认两边 CIDR 不重叠，明确路由和安全域。
2. 通过 VPN 或专线建立连接，使用 TGW/CEN 等中心化网络而不是大量点到点。
3. 设计双隧道/BGP 或主备路由，定义故障切换和收敛时间。
4. 在两边分别配置安全组、NACL/网络 ACL、防火墙和 DNS 转发。
5. 跨云服务调用使用 TLS、应用认证和超时/重试，不能只依赖网络连通。
6. 在 AWS VPC Flow Logs、阿里云流量日志、CloudTrail/ActionTrail 中建立关联查询。
7. 用真实业务流量和故障切换演练验证，而不是只测试 ping。

如果没有真实阿里云生产经历，建议明确说“我有 AWS 生产经验，了解 RAM、CEN、ECS、OSS 与 AWS 对应的治理模型，并能按网络、身份、审计和 IaC 的共性迁移；不会把熟悉 AWS 服务名等同于已经完成阿里云落地”。

---

## 十、综合场景题和参考答案

### 10.1 场景一：设计一个三账户生产平台

**题目：** 网络账户、生产账户、共享服务账户分离，生产应用需要访问共享日志、S3 和数据中心，如何设计？

**参考答案要点：**

1. 网络账户托管 TGW、Network Firewall、DX/VPN、Resolver。
2. 生产和共享服务 VPC 通过 RAM 共享 TGW attachment，路由表按生产/共享/检查区隔离。
3. 生产私有子网通过本 AZ NAT 出站，S3 使用 Gateway Endpoint，ECR/SSM/Logs 使用必要的 Interface Endpoint。
4. 到数据中心的路由只发布必要业务网段，默认路由经过检查 VPC。
5. 生产 Role 不允许修改网络基线；部署通过跨账户 AssumeRole。
6. 组织 CloudTrail、Config、GuardDuty、Flow Logs 集中到日志账户。
7. 用 Terraform 分层管理 network、shared-services、workload，分别管理 state。
8. 通过通信矩阵、Reachability Analyzer、合成探针和故障演练验证。

### 10.2 场景二：Terraform Apply 后生产 API 全部超时

**参考答案要点：**

1. 立即停止 pipeline 和后续 rollout，确认影响范围。
2. 对比 apply 前后 plan、state 版本、CloudTrail 和路由变更。
3. 检查 ALB Target 健康、SG/NACL、子网 route table、TGW propagation。
4. 检查是否替换了 ENI、路由、NAT、Endpoint 或 Launch Template。
5. 若变更明确且有安全回滚 plan，恢复上一个已验证版本；不要手工拼接未知配置。
6. 通过一个测试请求、Target 健康、5xx、延迟和业务交易确认恢复。
7. 修复模块测试、变更保护和 pipeline 审批，补充不可破坏资源策略。

### 10.3 场景三：只允许 GitHub Actions 部署生产

**参考答案要点：**

- GitHub OIDC 到工具账户 Role；
- 工具账户 Role 只能 AssumeRole 到生产部署 Role；
- Trust Policy 限制组织、仓库、分支和 environment；
- 生产 environment 要求 reviewer；
- PR 只做 plan，不允许 apply；
- apply 在 protected branch 重新生成 plan；
- 生产 Role 使用 Permission Boundary/SCP 限制 IAM、网络和审计高风险操作；
- CloudTrail 通过 SourceIdentity/Session Tags 关联 commit、workflow 和执行者；
- 部署 Role 不拥有长期 Access Key。

### 10.4 场景四：EC2 能访问 S3，但不能访问 ECR 拉取镜像

**排查要点：**

1. S3 Gateway Endpoint 正常不代表 ECR 可用，ECR 通常需要 ECR API、ECR DKR、S3、Logs、STS 等访问路径。
2. 检查私有子网 route table 是否有 NAT 或必要 Interface Endpoint。
3. 检查 Endpoint 的 DNS、Private DNS、路由和安全组。
4. 检查实例 Role 是否有 `ecr:GetAuthorizationToken`、拉取镜像所需权限。
5. 检查 ECR repository policy、KMS 权限和镜像 Region。
6. 验证：

```bash
aws ecr get-login-password --region ap-southeast-1
curl -vk https://api.ecr.ap-southeast-1.amazonaws.com/
docker pull 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/app@sha256:...
```

### 10.5 场景五：生产节点磁盘没有满，但应用写入失败

**参考答案：**

- 先看 `df -hT` 和 `df -ih`；
- 查 `dmesg` 是否有 I/O error、文件系统只读；
- 查 `mount`/`findmnt` 挂载参数；
- 查 `lsof +L1` 已删除但仍占用的文件；
- 查应用用户权限、ACL、SELinux；
- 查 inode、quota、容器 overlay2 和 EBS 状态；
- 以应用实际用户执行最小写入测试；
- 恢复时先保证数据安全和服务可用，再扩容或修复文件系统。

---

## 十一、面试实操题：白板和 Live Coding

### 11.1 白板题：画出多账户生产网络

至少画出：

```text
Internet
  |
WAF/ALB -> Public Subnet
               |
          Private App Subnet -> Private DB Subnet
               |
      VPC Endpoint / NAT / TGW
               |
     Shared Services / On-prem / Inspection
```

并在图旁写明：

- 每个 AZ 的 NAT；
- 子网 route table；
- SG 来源和端口；
- TGW route table 的隔离；
- DNS Resolver；
- Flow Logs 和 CloudTrail；
- 跨账户 Role；
- 失败时的回滚和观测点。

### 11.2 Terraform 小题

可能要求：

- 创建两个 AZ 的 VPC、私有子网和路由；
- 为 S3 配置 Gateway Endpoint；
- 使用变量和 `for_each`；
- 输出 subnet IDs；
- 使用 provider alias 跨账户部署；
- 添加 tags、加密和最小权限；
- 解释 state、plan 和 apply 的治理方式。

面试时不要追求“现场写出所有资源”，应先说清楚假设、依赖、模块边界和验证方法。

### 11.3 Linux Live Troubleshooting

常见任务：

```text
CPU 100%：top -> pidstat -> 线程/日志 -> 应用根因
内存不足：free/vmstat -> OOM -> 进程/cgroup -> 泄漏或容量
磁盘满：df/df -i/du/lsof -> 日志/删除文件/扩容
端口不通：ss -> ip route -> dig -> nc/curl -> tcpdump
节点失联：EC2 状态 -> SSM -> 日志 -> 网络/磁盘/内核
GPU 异常：nvidia-smi -> lspci -> dmesg -> 驱动/CUDA/应用
```

优秀答案会边执行边说明“这个命令要验证的假设是什么”，而不是无目的地堆命令。

---

## 十二、常见失分点和反例

### 12.1 网络失分点

- 只说“放开 0.0.0.0/0”；
- 不考虑回程路由；
- 认为 TGW 自动传递所有路由；
- 把 VPC Endpoint 当成万能网络互通；
- 只会检查 SG，不看 DNS、NACL、Flow Logs 和应用监听；
- 不考虑 CIDR 重叠和跨 AZ NAT 成本。

### 12.2 IAM 失分点

- 给所有 pipeline `AdministratorAccess`；
- 不区分 Trust Policy 和 Permissions Policy；
- 认为 SCP 能直接授予权限；
- 把 Permission Boundary 当成授权策略；
- 使用个人长期 Access Key；
- 只看 IAM policy，不看资源策略、KMS、SCP 和 Endpoint Policy。

### 12.3 IaC 失分点

- 不理解 state 和 locking；
- 生产直接本地 apply；
- 不审核 `destroy/replace`；
- 用 `count` 导致资源地址重排；
- 发生漂移就直接 `refresh` 或手工改 state；
- state、plan artifact 或 Secret 暴露在日志中。

### 12.4 CI/CD 失分点

- 构建和部署混在一个不可回滚脚本；
- 使用 `latest` 镜像；
- PR 可直接拿到生产凭据；
- 没有审批、部署窗口和自动回滚；
- 只关注 pipeline 成功，不验证业务指标；
- 忽略 Runner/Agent 的隔离和插件供应链风险。

### 12.5 生产排障失分点

- 上来就重启，导致现场和日志丢失；
- 没有记录时间、变更和影响范围；
- 看到 CPU 高就扩容，未判断 I/O、锁或下游依赖；
- 只会 `ping`，不会验证 TCP、DNS、路由和应用协议；
- 修复后不做回归验证和复盘。

---

## 十三、建议准备的真实项目故事

至少准备以下五个 STAR 故事，每个故事能在 2-3 分钟内讲清楚：

### 故事 A：多账户网络治理

- Situation：账户和 VPC 增长导致路由、权限和审计不可控。
- Task：建立网络账户、TGW、路由隔离和集中日志。
- Action：CIDR/IPAM、RAM、TGW route table、Endpoint、Flow Logs、IaC pipeline。
- Result：减少点到点连接，降低越权访问，故障定位时间缩短。

### 故事 B：Terraform 漂移或危险变更

- Situation：控制台改动导致生产和代码不一致。
- Task：识别漂移并安全恢复。
- Action：plan、CloudTrail、state 版本、import/moved、审批和基线检查。
- Result：恢复一致性且没有破坏生产数据。

### 故事 C：CI/CD 事故与回滚

- Situation：部署后 5xx/延迟上升。
- Task：降低影响并恢复服务。
- Action：停止放量、指标定位、Blue/Green 或回滚 artifact、数据兼容验证。
- Result：恢复时间、避免再次发生的测试和告警。

### 故事 D：Linux/网络生产事故

- Situation：服务超时、节点失联、磁盘或权限异常。
- Task：在保留证据的同时恢复业务。
- Action：指标、日志、路由、Flow Logs、SSM、进程和文件系统检查。
- Result：明确根因，补充 Runbook/监控/自动修复。

### 故事 E：安全权限收敛

- Situation：团队依赖管理员权限，审计风险高。
- Task：在不阻塞交付的情况下最小化权限。
- Action：Access Analyzer、CloudTrail、Permission Set、Boundary、SCP、break-glass。
- Result：减少长期凭据和高风险权限，保留可追踪的应急通道。

讲故事时不要泄露客户名称、账户 ID、IP、Token、内部域名或敏感业务数据；可以使用脱敏后的架构和相对指标。

---

## 十四、面试前检查清单

### 14.1 必须能口述

- VPC、Subnet、Route Table、IGW、NAT、Endpoint 的关系；
- TGW association/propagation 和路由隔离；
- SG/NACL/Network Firewall 的边界；
- 多账户、RAM、共享 VPC 和跨账户 Role；
- Trust Policy、IAM Policy、SCP、Boundary 的差异；
- S3/KMS 跨账户授权链路；
- EC2/ASG/ALB/NLB/EBS/S3/SSM 的生产取舍；
- Terraform state、locking、module、provider alias、漂移；
- CloudFormation change set、StackSet、drift、rollback；
- Jenkins/GitHub Actions/GitLab/CodeBuild 的安全流水线；
- Linux CPU、内存、磁盘、网络和日志排查；
- 裸金属/GPU 的驱动、NUMA、PCIe、NVMe 排查；
- AWS 与阿里云网络、身份、计算和审计服务映射。

### 14.2 必须能现场演示或解释

- 用 `aws sts get-caller-identity` 验证当前账户和角色；
- 用 `dig`、`ip route`、`nc`、`curl`、`ss`、`tcpdump` 排查网络；
- 用 `top`、`vmstat`、`iostat`、`df`、`journalctl` 定位 Linux 问题；
- 解释一份 Terraform plan 中的 `create/update/replace/destroy`；
- 解释一条 GitHub OIDC Trust Policy；
- 从 CloudTrail 找到一次 AccessDenied 或网络配置变更；
- 为一个问题设计验证、回滚和复盘。

### 14.3 90 秒自我介绍参考结构

> 我主要从事 AWS 基础设施和生产运维，重点是多账户、多 VPC 网络、Terraform/CloudFormation、IAM 治理和 Linux 故障排查。我做过从 VPC、子网、路由、TGW、NAT、Endpoint、ALB/EC2 到审计和监控的完整交付，也把基础设施变更接入 CI/CD，通过 plan 审核、短期跨账户角色和 OIDC 控制生产权限。遇到生产问题时，我会先控制影响并保留证据，再从 DNS、路由、安全策略、主机和应用逐层验证。裸金属/GPU和阿里云方面我能把驱动、NUMA、跨云网络和身份治理纳入排查；对没有实际落地过的范围，我会明确边界，并通过标准化模块、实验和演练快速补齐。

---

## 十五、面试官评分参考

| 维度 | 1 分 | 3 分 | 5 分 |
|---|---|---|---|
| AWS 网络 | 只能背服务定义 | 能画单 VPC 并排障 | 能设计多账户 TGW、隔离、DNS、成本和证据链 |
| IAM | 只会加 Allow | 理解跨账户 Role | 能结合 SCP、Boundary、KMS、审计和 break-glass |
| IaC | 会复制资源代码 | 会 module 和 state | 能治理 plan、漂移、回滚、跨账户和变更风险 |
| CI/CD | 只会写脚本 | 能完成构建部署 | 能做 OIDC、审批、不可变 artifact、灰度和自动回滚 |
| Linux | 会看 CPU/磁盘 | 能定位常见故障 | 能保留证据、控制影响、跨云/主机/应用做 RCA |
| 生产意识 | 先重启、无记录 | 有基本 Runbook | 关注 RTO/RPO、审计、成本、复盘和防复发 |
| 交流表达 | 罗列术语 | 有结构地回答 | 能说明取舍、证据、风险，并诚实划分经验边界 |

最终通过标准不是“每道题都答得像文档”，而是能够在一个模糊的生产场景中：

```text
先澄清约束 -> 画出边界 -> 做出可解释的设计
-> 用 IaC 和安全身份交付 -> 用指标和日志验证
-> 出故障能恢复 -> 事后能治理和复盘
```
