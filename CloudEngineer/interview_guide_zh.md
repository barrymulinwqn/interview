# AWS Cloud Engineer 面试高频题与实战指南

> 本资料根据 [jd.md](./jd.md) 整理，定位是医药/企业级环境中的 Senior Cloud Engineer 面试准备。  
> 重点不是背 AWS 服务名，而是能够说明：**为什么这样设计、如何用 IaC 部署、如何验证、出问题如何定位，以及如何满足安全与合规要求**。

## 一、职位要求拆解

### 1. 最重要的能力

| 优先级 | 能力 | 面试官希望听到的证据 |
|---|---|---|
| P0 | AWS Infrastructure | 亲手设计和部署过 VPC、Subnet、Route Table、IGW、NAT Gateway、Security Group、EC2、ALB、IAM |
| P0 | Terraform | 写过可复用 module，理解 state、remote backend、plan/apply、drift、import、CI/CD 中的审批 |
| P0 | CloudFormation | 能读写模板，理解 stack、change set、rollback、drift detection、nested stack |
| P0 | 网络排障 | 能从 DNS、路由、SG、NACL、NAT、端点、应用监听端口逐层定位 |
| P1 | 安全与合规 | 最小权限、私有子网、加密、Secrets Manager、CloudTrail、审计、数据分级 |
| P1 | 计算与存储 | EC2/ASG/ALB、EBS、S3、EFS、RDS 的选型和高可用设计 |
| P1 | DevOps | CodeBuild、ECR、GitHub Actions、OIDC、部署策略、回滚 |
| P1 | 容器与数据库 | Docker/Kubernetes、RDS PostgreSQL/MySQL、DynamoDB 的运维考虑 |
| P2 | 自动化与诊断 | Linux Shell/Python、日志和指标、根因分析、Runbook |
| P2 | 沟通与 AI | 跨团队协作、向非技术人员解释风险，理解 Generative AI/Agentic AI 的落地方式 |

### 2. Senior 级回答的共同特征

回答基础题时，建议始终补充以下四层：

1. **设计**：组件如何连接，流量和数据如何流动。
2. **实施**：用 Terraform/CloudFormation 或命令行如何落地。
3. **验证**：用哪些命令、指标、日志和测试确认成功。
4. **运营**：故障、扩容、回滚、成本、安全和合规怎么处理。

例如不要只说“私有子网更安全”，而要说明：

> 私有子网默认路由不指向 Internet Gateway；若实例需要访问公网更新源，通过 NAT Gateway 出站。生产环境按 AZ 部署 NAT，避免单 AZ 故障；如果只访问 S3/ECR 等 AWS 服务，则优先使用 VPC Endpoint，减少公网路径和 NAT 成本。安全组只允许应用真正需要的来源和端口，并通过 VPC Flow Logs、CloudTrail 和监控验证。

---

## 二、AWS 网络：最高频、最能区分 hands-on 能力的部分

### 2.1 VPC、Subnet、Route Table、IGW、NAT 的关系

**高频问题：请从零解释一个生产 VPC 的基本组成。**

一个常见的两可用区架构如下：

```text
                                      Internet
                                          |
                               Internet Gateway (IGW)
                                          |
┌─────────────────────────────────────────┴─────────────────────────────────────────┐
│ VPC 10.20.0.0/16                                                               │
│                                                                                 │
│  ┌──────────────────────────── AZ-a ────────────────────────────┐               │
│  │                                                              │               │
│  │  Public Subnet 10.20.1.0/24                                  │               │
│  │  ┌───────────────┐       ┌──────────────────────┐            │               │
│  │  │ ALB node      │       │ NAT Gateway-a        │            │               │
│  │  │ (public ENI)  │       │ EIP: 203.0.113.10    │            │               │
│  │  └───────┬───────┘       └──────────┬───────────┘            │               │
│  │          │                          │                        │               │
│  │  ┌───────▼──────────────────────────▼───────┐                │               │
│  │  │ RT-public-a (associated with public-a)  │                │               │
│  │  │ local 10.20.0.0/16 -> local             │                │               │
│  │  │ 0.0.0.0/0       -> IGW                  │                │               │
│  │  └──────────────────────────────────────────┘                │               │
│  │                                                              │               │
│  │  Private Subnet 10.20.11.0/24                                │               │
│  │  ┌───────────────┐       ┌──────────────────────┐            │               │
│  │  │ App/Worker-a  │──────▶│ RT-private-a         │────────────┐│               │
│  │  │ no public IP  │       │ local 10.20.0.0/16    │            ││               │
│  │  └───────────────┘       │ 0.0.0.0/0 -> NAT-a    │            ││               │
│  │                          └──────────────────────┘            ││               │
│  └──────────────────────────────────────────────────────────────┘│               │
│                                                                  │               │
│  ┌──────────────────────────── AZ-b ────────────────────────────┐│               │
│  │                                                              ││               │
│  │  Public Subnet 10.20.2.0/24                                  ││               │
│  │  ┌───────────────┐       ┌──────────────────────┐            ││               │
│  │  │ ALB node      │       │ NAT Gateway-b        │            ││               │
│  │  │ (public ENI)  │       │ EIP: 203.0.113.11    │            ││               │
│  │  └───────┬───────┘       └──────────┬───────────┘            ││               │
│  │          │                          │                        ││               │
│  │  ┌───────▼──────────────────────────▼───────┐                ││               │
│  │  │ RT-public-b (associated with public-b)  │                ││               │
│  │  │ local 10.20.0.0/16 -> local             │                ││               │
│  │  │ 0.0.0.0/0       -> IGW                  │                ││               │
│  │  └──────────────────────────────────────────┘                ││               │
│  │                                                              ││               │
│  │  Private Subnet 10.20.12.0/24                                ││               │
│  │  ┌───────────────┐       ┌──────────────────────┐            ││               │
│  │  │ App/Worker-b  │──────▶│ RT-private-b         │────────────┘│               │
│  │  │ no public IP  │       │ local 10.20.0.0/16    │             │               │
│  │  └───────────────┘       │ 0.0.0.0/0 -> NAT-b    │             │               │
│  │                          └──────────────────────┘             │               │
│  └──────────────────────────────────────────────────────────────┘               │
│                                                                                 │
│  每张子网只关联一张路由表；local 路由由 VPC 自动提供。                           │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 图中最重要的绑定关系

```text
Public Subnet-a  --关联--> RT-public-a  --0.0.0.0/0--> IGW
Public Subnet-b  --关联--> RT-public-b  --0.0.0.0/0--> IGW

Private Subnet-a --关联--> RT-private-a --0.0.0.0/0--> NAT Gateway-a
Private Subnet-b --关联--> RT-private-b --0.0.0.0/0--> NAT Gateway-b

NAT Gateway-a/b  --位于--> 对应 AZ 的 Public Subnet
NAT Gateway-a/b  --绑定--> 对应 EIP
ALB              --跨 AZ 部署--> 两个 Public Subnet
```

#### 三条典型流量路径

```text
1. 公网用户访问应用：
   Internet -> IGW -> ALB(public-a 或 public-b)
            -> Target Group -> App(private-a 或 private-b)

2. 私有应用访问公网 API：
   App-a -> RT-private-a -> NAT-a -> RT-public-a -> IGW -> Internet
   App-b -> RT-private-b -> NAT-b -> RT-public-b -> IGW -> Internet

3. VPC 内部通信：
   App-a -> local 10.20.0.0/16 -> App-b/数据库/内部负载均衡
   不需要经过 IGW 或 NAT Gateway
```

公网用户不能根据 NAT Gateway 的 EIP 主动连接回私有应用：NAT Gateway 只提供私有子网发起的出站地址转换和返回流量。应用实例没有公有 IP，且私有路由表没有指向 IGW 的默认路由，因此应用不会被直接暴露到 Internet。

#### 为什么建议每个 AZ 一个 NAT Gateway

生产环境通常让每个 AZ 的私有子网使用同 AZ 的 NAT Gateway：

- AZ-a 故障时，AZ-b 的应用仍可通过 NAT-b 出站。
- 避免所有私有子网依赖单一 NAT，降低故障域耦合。
- 避免私有实例跨 AZ 访问 NAT 产生额外的跨 AZ 流量和费用。
- 通过独立的私有路由表，能够清楚表达“本 AZ 出站走本 AZ NAT”。

成本敏感的非生产环境可以让两个私有路由表都指向一台 NAT Gateway，但必须明确这是成本与可用性的取舍：该 NAT 所在 AZ 故障时两个 AZ 的私有实例都可能失去公网出站能力，且另一个 AZ 的流量会跨 AZ 到达 NAT。

#### Public/Private 的判断标准

“Public Subnet”不是 AWS 中单独的布尔属性，而是由路由和资源共同决定的：

| 子网 | 关键路由 | 典型资源 | 是否直接公网可达 |
|---|---|---|---|
| Public-a/b | `0.0.0.0/0 -> IGW` | ALB、NAT Gateway、必要的 Bastion | 只有资源有公有 IPv4/EIP 且 SG/NACL 允许时才可达 |
| Private-a/b | `0.0.0.0/0 -> NAT-a/b` | App、Worker、内部服务 | 无法被公网主动连接，可主动出站 |
| Isolated（可选） | 没有公网默认路由 | 数据库、缓存、内部数据服务 | 不访问公网；按需使用 VPC Endpoint |

因此，不能只因为实例放在名为 `public` 的子网就认定它能上网；还需要同时检查公有 IP、路由表、IGW、Security Group、NACL 和实例操作系统防火墙。

#### 关键概念

- **VPC**：区域内隔离的逻辑网络，CIDR 不能与要互联的网络冲突。
- **Subnet**：属于单个 AZ；“public/private”不是 Subnet 的属性，而是由路由决定的。
- **Internet Gateway（IGW）**：提供 VPC 与 Internet 的双向路径；实例还必须具有公有 IPv4/EIP，且安全组允许流量，才能真正被公网访问。
- **Route Table**：决定下一跳。路由匹配时使用最长前缀匹配。
- **NAT Gateway**：允许私有子网主动访问公网，公网不能通过 NAT Gateway 主动连接回私网实例。NAT Gateway 本身必须放在公有子网并绑定 EIP。
- **VPC Endpoint**：私网访问 AWS 服务。Gateway Endpoint 常用于 S3/DynamoDB；Interface Endpoint 使用 ENI 和私网 DNS，适用于 ECR、Secrets Manager、CloudWatch 等服务。
- **AZ**：一个区域内相互隔离的故障域。高可用设计通常至少跨两个 AZ。

#### 面试中的隐藏考点

1. 只部署一台 NAT Gateway 会形成单点故障，且跨 AZ 使用会产生跨 AZ 流量成本。
2. “实例在 public subnet”不等于它一定可上网；要同时检查公有 IP、路由、SG、NACL 和操作系统防火墙。
3. NAT Gateway 不是入站防火墙，不应把它当成安全边界。
4. CIDR 规划要为扩容、VPC Peering、Transit Gateway、VPN/Direct Connect 预留空间。
5. 医药公司可能涉及 PHI/PII 或受监管数据，生产数据库通常不应有公网路径。

### 2.2 Security Group 与 NACL

**高频问题：Security Group 和 Network ACL 有什么区别？**

| 项目 | Security Group | Network ACL |
|---|---|---|
| 作用范围 | ENI/实例级 | Subnet 级 |
| 状态 | Stateful，允许返回流量自动放行 | Stateless，入站和出站规则都要显式配置 |
| 规则 | 只有 allow | 支持 allow/deny，按规则编号顺序匹配 |
| 常见用途 | 应用之间的最小端口授权 | 子网级额外边界控制、显式拒绝 |
| 排障优先级 | 通常先查 | 规则复杂时再查 NACL 和临时端口 |

推荐的三层 SG 模型：

```text
alb-sg:
  Inbound 443 from approved corporate CIDRs/CloudFront
  Outbound to app-sg on application port

app-sg:
  Inbound application port from alb-sg only
  Outbound to db-sg and approved endpoints

db-sg:
  Inbound 5432/3306 from app-sg only
  No public inbound rule
```

不要把 `0.0.0.0/0` 作为数据库、SSH 或 RDP 的长期来源。运维访问优先使用 SSM Session Manager、堡垒机或企业 VPN，并采用短时权限。

### 2.3 网络排障题

**场景：EC2 在私有子网，不能访问外部 API，如何排查？**

按数据路径由近到远检查，避免一上来就修改安全组：

1. 应用是否真的监听目标端口：`ss -lntp`、应用日志。
2. DNS 是否能解析：`dig api.example.com`；检查 VPC DNS support/hostnames。
3. 默认路由是否存在：私有路由表是否有 `0.0.0.0/0 -> NAT Gateway`。
4. NAT Gateway 是否为 `Available`，是否位于公有子网，是否绑定 EIP。
5. 公有子网路由是否指向 IGW。
6. 实例 SG 出站、NACL 入站/出站是否允许目标端口和返回临时端口。
7. 目标 API 是否限制来源 IP；若来源是 NAT EIP，应确认白名单。
8. OS 防火墙、代理变量、证书、应用重试和超时配置是否正常。
9. 查看 VPC Flow Logs、NAT Gateway metrics（ErrorPortAllocation、BytesOutToDestination）以及应用指标。

**场景：ALB 返回 504，后端 EC2 看起来健康。**

- 检查 Target Group health check 的协议、端口、路径和成功码。
- 检查 `alb-sg -> app-sg` 是否允许实际监听端口。
- 检查后端进程是否绑定 `0.0.0.0`，而不是只绑定 `127.0.0.1`。
- 检查应用响应时间是否超过 ALB idle timeout。
- 检查路由表、NACL、实例 CPU/内存、连接数和应用日志。
- 区分 4xx、5xx、504：它们对应客户端、后端应用和网络/超时的不同方向。

---

## 三、Terraform：必须讲清楚状态、模块和交付流程

### 3.1 Terraform 与 CloudFormation 怎么选

**高频问题：Terraform 和 CloudFormation 的区别？**

建议不要回答成“Terraform 更好”，而要按组织边界回答：

- **CloudFormation**：AWS 原生，和 AWS 服务及 IAM 集成紧密，Stack/Change Set/Drift/回滚体验成熟；多 AWS 账户治理时易与 AWS 组织体系结合。
- **Terraform**：provider 生态更广，模块和表达能力适合统一管理 AWS、Azure、GitHub、Datadog 等资源；但必须认真管理 state、provider 版本和第三方 provider 风险。
- **选择依据**：团队现有标准、云平台范围、模块复用、状态存储、审批流程、灾备和运维能力，而不是个人偏好。
- **不要混管同一资源**：同一个资源不能同时由 Terraform 和 CloudFormation 管理，否则容易发生漂移和互相覆盖。

### 3.2 Terraform State 的核心考点

**高频问题：state 是什么？丢了或冲突怎么办？**

- State 是 Terraform 对配置和真实资源之间映射关系的记录，不是普通缓存。
- 生产环境应使用远程 backend，并启用加密、版本保留和并发锁定；S3 backend 可使用状态锁定机制，具体方式要遵循当前 Terraform 版本和组织标准。
- State 可能包含敏感属性，即使变量标记为 `sensitive`，也不能把 state 当作安全存储。
- 每次变更前执行 `plan`，通过 CI 做审批；禁止多人直接在本地对生产执行 `apply`。
- 误删资源或 state 映射异常时，先备份并确认资源归属，再考虑 `state rm`、`import` 或从版本恢复；不要直接手改 state JSON。
- Terraform state lock 卡住时，先确认没有仍在运行的 apply，再按组织流程解除锁，不要为了绕过错误强制解锁。

### 3.3 Module 设计原则

可复用 module 不应把所有资源和业务逻辑塞在一个文件中。建议：

```text
modules/
  vpc/
    main.tf
    variables.tf
    outputs.tf
    versions.tf
envs/
  dev/
  staging/
  prod/
```

好的 module 应当：

- 明确定义输入、输出、默认值和校验规则。
- 不硬编码账号、区域、CIDR、AMI 或密码。
- 通过 provider version 和 Terraform version 约束保证可重复性。
- 用 `for_each` 表达有稳定键的集合，避免因列表顺序变化造成资源重建。
- 输出下游真正需要的 ID，而不是输出过多内部细节。
- 对危险操作使用 `lifecycle`、审批或策略检查，而不是盲目 `prevent_destroy`。
- 版本化发布，变更先在 dev/staging 验证，再进入生产。

### 3.4 Terraform 高频题

**Q：`count` 和 `for_each` 如何选择？**

- `count` 适合按数量创建、实例之间没有稳定身份的资源。
- `for_each` 适合有业务键的 map/set，例如按 AZ 或环境创建资源，删除中间元素时不容易引起索引漂移。

**Q：如何处理 Terraform drift？**

1. `terraform plan` 观察配置、state 和真实资源的差异。
2. 判断是人工改动、AWS 自动变更，还是代码/Provider 行为变化。
3. 应将期望配置写回代码；若人工变更是正确的，则更新代码后 plan。
4. 只有明确接受外部管理时，才考虑 `ignore_changes`，并记录原因，否则会掩盖真实漂移。

**Q：如何把已有资源纳入 Terraform？**

1. 先写出与真实资源一致的资源块。
2. 使用当前版本支持的 `import` 流程导入。
3. `plan` 反复调整配置，直到不再计划意外修改。
4. 先在非生产环境演练；导入不是“自动生成完美代码”，仍需补齐依赖、变量和模块边界。

### 3.5 Terraform VPC 实操骨架

以下示例演示网络核心链路，适合面试现场解释。真实生产代码还应补充标签规范、VPC Flow Logs、Endpoint、NAT 监控、策略检查和成本标签。

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

variable "region" {
  type    = string
  default = "ap-southeast-1"
}

variable "vpc_cidr" {
  type    = string
  default = "10.20.0.0/16"
}

variable "azs" {
  type    = list(string)
  default = ["ap-southeast-1a", "ap-southeast-1b"]
}

locals {
  public_subnet_cidrs  = ["10.20.1.0/24", "10.20.2.0/24"]
  private_subnet_cidrs = ["10.20.11.0/24", "10.20.12.0/24"]
}

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = { Name = "interview-vpc" }
}

resource "aws_subnet" "public" {
  for_each = {
    for index, az in var.azs : az => {
      cidr = local.public_subnet_cidrs[index]
    }
  }

  vpc_id                  = aws_vpc.this.id
  availability_zone       = each.key
  cidr_block              = each.value.cidr
  map_public_ip_on_launch = false
  tags                    = { Name = "public-${each.key}" }
}

resource "aws_subnet" "private" {
  for_each = {
    for index, az in var.azs : az => {
      cidr = local.private_subnet_cidrs[index]
    }
  }

  vpc_id            = aws_vpc.this.id
  availability_zone = each.key
  cidr_block        = each.value.cidr
  tags              = { Name = "private-${each.key}" }
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
}

resource "aws_route_table_association" "public" {
  for_each       = aws_subnet.public
  route_table_id = aws_route_table.public.id
  subnet_id      = each.value.id
}

resource "aws_eip" "nat" {
  for_each = aws_subnet.public
  domain   = "vpc"
}

resource "aws_nat_gateway" "this" {
  for_each      = aws_subnet.public
  allocation_id = aws_eip.nat[each.key].id
  subnet_id     = each.value.id

  depends_on = [aws_internet_gateway.this]
}

resource "aws_route_table" "private" {
  for_each = aws_subnet.private
  vpc_id   = aws_vpc.this.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.this[each.key].id
  }
}

resource "aws_route_table_association" "private" {
  for_each       = aws_subnet.private
  route_table_id = aws_route_table.private[each.key].id
  subnet_id      = each.value.id
}

resource "aws_security_group" "app" {
  name        = "app-sg"
  description = "Application traffic from the load balancer"
  vpc_id      = aws_vpc.this.id

  ingress {
    description     = "Application traffic from ALB SG"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    protocol    = "-1"
    from_port   = 0
    to_port     = 0
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = aws_vpc.this.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["203.0.113.0/24"]
  }

  egress {
    protocol    = "-1"
    from_port   = 0
    to_port     = 0
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

#### 这段代码面试时必须主动指出的点

- `aws_security_group.app` 引用了 `aws_security_group.alb`，Terraform 会建立依赖；生产中要避免循环 SG 引用。
- 示例为了突出网络链路，允许 ALB 全出站；实际应按服务和端口收紧。
- 两个 NAT Gateway 换取跨 AZ 故障隔离，但成本更高；非生产环境可以在明确接受风险后只部署一个。
- `map_public_ip_on_launch = false` 只是不自动分配公有 IP，不代表子网天然私有。
- 需要补充 `aws_vpc_endpoint` 访问 S3/ECR/CloudWatch/Secrets Manager，并使用 VPC Flow Logs。

#### 标准操作步骤

```bash
terraform fmt -check
terraform init
terraform validate
terraform plan -out=tfplan
terraform show tfplan
# 经过审批后
terraform apply tfplan
terraform output
# 实验环境确认不再需要后
terraform destroy
```

验证不能只看 `apply` 成功，还应检查：

- 子网是否分布在多个 AZ。
- 公有路由表是否指向 IGW，私有路由表是否按预期指向 NAT。
- ALB/EC2/数据库 SG 是否形成最小权限链路。
- 私有实例能否访问允许的外部地址，公网不能直接入站。
- CloudTrail、Flow Logs、监控和成本标签是否生效。

---

## 四、CloudFormation 高频题与模板思维

### 4.1 核心概念

- **Stack**：一组由模板管理的资源生命周期。
- **Change Set**：执行前查看创建、修改、替换和删除，适合审批。
- **Rollback**：创建或更新失败时回滚；需要知道失败资源和事件原因。
- **Drift Detection**：发现模板期望状态与实际资源的差异，但不会自动修复。
- **Nested Stack**：拆分网络、共享安全、应用等模板，便于复用和边界管理。
- **Parameters/Outputs/Exports**：让模板可复用，并向其他 Stack 暴露资源 ID。
- **Stack Policy/Termination Protection**：保护关键资源和生产 Stack。

### 4.2 CloudFormation 面试题

**Q：CloudFormation 更新时为什么会替换资源？**

资源属性有的支持 in-place update，有的更新需要 replacement。替换可能改变资源 ID、丢失临时数据或造成短暂中断。回答时应：

1. 先查资源属性的更新行为。
2. 查看 Change Set 是否显示 `Replacement: True`。
3. 对数据库、EIP、Security Group 等关键资源设置备份、保留策略或拆分生命周期。
4. 使用蓝绿/滚动策略和变更窗口降低风险。

**Q：Stack 更新失败如何处理？**

1. 查看 Stack Events，找到第一条根因，而不是只看最后的 `UPDATE_ROLLBACK_FAILED`。
2. 检查 IAM、配额、依赖、资源命名、CIDR 冲突和不可更新属性。
3. 保留现场，确认是否需要 `continue-update-rollback`。
4. 修复模板或权限后重新创建 Change Set，验证不再替换关键资源。
5. 记录故障和改进措施，避免直接手工改资源导致更多 drift。

### 4.3 CloudFormation VPC 简化示例

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Highly available VPC foundation

Parameters:
  VpcCidr:
    Type: String
    Default: 10.30.0.0/16
  PublicSubnetCidr:
    Type: String
    Default: 10.30.1.0/24
  PrivateSubnetCidr:
    Type: String
    Default: 10.30.11.0/24

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: cfn-interview-vpc

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnetCidr
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: public-subnet

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet

Outputs:
  VpcId:
    Description: VPC identifier
    Value: !Ref VPC
    Export:
      Name: !Sub '${AWS::StackName}-VpcId'
```

这个模板只展示基础链路；生产模板需要按 AZ 完整部署私有子网、NAT/VPC Endpoint、Flow Logs、标签、加密和安全策略。面试时应明确说明“示例为了教学而简化”，不要把单 AZ 模板描述为生产高可用方案。

---

## 五、计算、存储和高可用设计

### 5.1 EC2、ALB、ASG

**高频问题：如何设计一个可扩展的 Web 应用？**

建议回答：

1. Route 53 将域名解析到 ALB；HTTPS 在 ALB 终止，证书由 ACM 管理。
2. ALB 跨多个 public subnet，Target Group 连接 private subnet 中的 EC2。
3. EC2 使用 Launch Template、IAM Instance Profile、SSM Agent 和最小权限。
4. ASG 跨 AZ，使用健康检查和目标追踪扩缩容；部署使用 rolling、canary 或蓝绿。
5. 无状态会话放到 Redis/DynamoDB 等外部存储，文件放 S3/EFS，避免绑定单台实例。
6. 日志发送到 CloudWatch/S3，指标和告警覆盖 5xx、延迟、CPU、内存、磁盘和容量。
7. 数据库使用 Multi-AZ，应用和数据库 SG 分层。

**为什么不建议给生产 EC2 直接开放 SSH？**

长期开放 22 端口会扩大攻击面并增加密钥管理负担。优先使用 SSM Session Manager；若必须 SSH，应限制到企业网络/堡垒机、使用短期凭证、审计会话并定期轮换。

### 5.2 EBS、EFS、S3 的选型

| 服务 | 适用场景 | 关键面试点 |
|---|---|---|
| EBS | 单实例块存储、数据库磁盘、低延迟 | 与 AZ 绑定、快照、IOPS/吞吐、加密、扩容 |
| EFS | 多 AZ 多实例共享 POSIX 文件系统 | 吞吐/性能模式、挂载目标、SG、成本和权限 |
| S3 | 对象、备份、静态文件、数据湖 | Block Public Access、版本化、生命周期、KMS、Bucket Policy、跨区域复制 |

S3 安全回答至少包括：Block Public Access、默认加密、最小 Bucket Policy、访问日志/CloudTrail、版本化、生命周期、Object Lock（有不可篡改要求时）和数据分级。

### 5.3 RDS 与 DynamoDB

**RDS Multi-AZ 与 Read Replica 的区别：**

- Multi-AZ 主要解决实例级故障和高可用，通常用于同步备用和自动故障转移，不是主要读扩展方案。
- Read Replica 主要解决读扩展和部分灾备需求，数据复制存在延迟，应用需要处理读后写一致性。
- 生产数据库还应考虑备份保留、PITR、参数组、加密、补丁窗口、连接池、监控和恢复演练。

**DynamoDB 适合什么场景？**

访问模式和主键设计优先，适合高并发、低延迟、键值/文档型访问。面试时应谈分区键基数、热点分区、GSI 成本与一致性、TTL、容量模式和备份恢复，而不是只说“无需维护服务器”。

---

## 六、安全、合规与医药行业背景

### 6.1 IAM 与数据安全

- 使用 IAM Role 而不是长期 Access Key；GitHub Actions 使用 OIDC 联邦。
- 权限遵循最小权限，并通过 permission boundary、SCP、条件键和资源策略加强边界。
- Secrets Manager/Parameter Store 保存密钥，不写入 Git、镜像、Terraform variables 或日志。
- KMS 管理静态加密密钥；传输使用 TLS；说明密钥轮换、访问审计和跨账户授权。
- CloudTrail 组织级记录管理事件和数据事件；CloudWatch/Config/Security Hub 用于检测和告警。
- 生产账户与开发账户隔离，采用多账户、集中日志、网络和安全基线。

### 6.2 医药公司的隐含考点

面试官可能关心的不只是“系统能运行”，还包括：

- PHI、PII、临床试验或实验室数据的分类、最小访问和保留期限。
- 变更是否可审批、可追溯、可回滚；谁在什么时间访问过什么数据。
- 备份是否加密，是否做过恢复演练，RPO/RTO 是否明确。
- 是否需要区域限制、跨境数据控制、供应商风险评估和审计证据。
- 生产与非生产数据是否脱敏，日志是否意外记录敏感信息。

回答合规问题时，不要武断地声称“某个 AWS 服务天然合规”。应说：

> 先确认适用法规、合同和公司控制要求，再核对 AWS 服务的合规范围、责任共担模型和组织内部控制。云服务合规不等于应用自动合规，应用仍需正确配置访问、加密、审计、保留和响应流程。

---

## 七、CI/CD、容器与自动化

### 7.1 GitHub Actions、CodeBuild、ECR

**推荐流水线：**

```text
Pull Request
  -> fmt/lint/unit test/security scan
  -> terraform validate/plan
  -> 代码评审和审批
  -> 构建 Docker 镜像
  -> 推送 ECR（不可变 tag + digest）
  -> 部署到测试环境
  -> 集成/安全/冒烟测试
  -> 生产审批
  -> canary/rolling/blue-green
  -> 指标验证与自动/人工回滚
```

关键实践：

- GitHub Actions 不保存长期 AWS 密钥，使用 OIDC AssumeRole，并限制 repository、branch、environment。
- ECR 开启镜像扫描、生命周期清理和加密；部署使用 digest 或不可变版本，不使用 `latest` 作为生产发布标识。
- Terraform plan 是构建产物，需要审计和审批；apply 使用同一份已审批 plan，避免重新计算造成差异。
- CodeBuild 使用最小 IAM Role、固定构建镜像、依赖锁定和安全日志处理。
- 失败时保留足够诊断信息，但不要在日志打印 token、密码或敏感环境变量。

### 7.2 Docker/Kubernetes 高频题

- Docker 镜像使用多阶段构建、非 root 用户、最小基础镜像、固定依赖版本和漏洞扫描。
- 容器应尽量无状态；配置和密钥通过运行时注入，不打包进镜像。
- Kubernetes 要理解 Deployment、Service、Ingress、ConfigMap、Secret、readiness/liveness probe、requests/limits、HPA 和 RBAC。
- 不能只说“重启 Pod”；要区分镜像拉取失败、探针失败、资源不足、网络策略、DNS、权限和应用本身崩溃。

---

## 八、故障排查与 RCA 高频题

### 8.1 通用排查框架

1. **确认影响范围**：哪些账户、区域、AZ、服务、用户和时间段受影响。
2. **建立时间线**：最后一次成功、变更、告警、故障开始和缓解时间。
3. **先恢复服务**：回滚已知风险变更、扩容、切流或启用备用路径。
4. **按层定位**：DNS → 网络路由 → SG/NACL → 负载均衡 → 主机/容器 → 应用 → 数据库/依赖。
5. **用证据验证假设**：日志、指标、Flow Logs、CloudTrail、部署记录，而不是凭感觉改配置。
6. **完成 RCA**：根因、触发因素、检测缺口、恢复动作、长期修复和负责人。

### 8.2 高频场景

**S3 AccessDenied：**

- 检查调用身份和实际 AssumeRole 链路。
- 检查 identity policy、bucket policy、SCP、permission boundary、VPC endpoint policy。
- 检查 KMS key policy 和加密上下文。
- 检查 bucket 是否要求 TLS、特定 VPC Endpoint 或组织账号。
- 用 CloudTrail 查真实拒绝事件，不要只凭错误文字猜测。

**EC2 磁盘满：**

- `df -h` 查文件系统，`du` 查目录，`lsblk` 查块设备。
- 检查日志、删除文件仍被进程占用、inode、临时文件和 Docker layer。
- 扩大 EBS 后还要扩展 partition/filesystem；确认快照和变更窗口。
- 修复日志轮转、容量告警和自动清理策略，避免只做一次性删除。

**应用发布后错误率上升：**

- 对比发布前后的 4xx/5xx、延迟、依赖错误、容器重启和数据库连接数。
- 确认是否只影响新版本目标组、某个 AZ 或某类请求。
- 停止继续扩散，先暂停 rollout 或回滚。
- 保留版本、配置、镜像 digest、Terraform plan 和日志，便于 RCA。

---

## 九、面试实操题：从零部署一个安全的双 AZ Web 基础设施

### 题目

请使用 Terraform 部署一个 AWS 环境，要求：

- 一个 VPC，CIDR 为 `10.20.0.0/16`。
- 至少两个 AZ。
- 每个 AZ 一个 public subnet 和一个 private subnet。
- public subnet 可承载 ALB，private subnet 承载应用实例。
- private subnet 可出站访问公网，但公网不能直接访问应用实例。
- 使用 Security Group 限制 ALB → App → DB 的流量。
- 输出 VPC、子网、路由表和安全组 ID。
- 说明如何在 CI/CD 中验证和发布。

### 期望的实施步骤

1. 先画数据流和 CIDR，不急着写资源。
2. 创建 VPC，开启 DNS support/hostnames。
3. 创建跨 AZ 的 public/private subnets。
4. 创建 IGW、公有路由表和关联。
5. 每个 AZ 创建 NAT Gateway、私有路由表和关联；说明成本与单 NAT 取舍。
6. 创建分层 Security Groups，避免 `0.0.0.0/0` 访问应用和数据库。
7. 如访问 S3/ECR/Secrets Manager，补充 VPC Endpoints。
8. 加上标签、Flow Logs、加密、监控和输出。
9. `fmt → init → validate → plan → review → apply`。
10. 用实际连接测试、AWS Console/CLI、Flow Logs 和指标验证。
11. 演练失败、回滚和 `destroy`；生产环境还要说明备份、锁定和审批。

### 评分标准

| 维度 | 通过标准 |
|---|---|
| 网络 | 能解释 CIDR、AZ、路由和 IGW/NAT 的关系，而不是只复制代码 |
| 安全 | SG 最小权限，数据库不公网暴露，能说明 IAM/Secrets/KMS |
| Terraform | 能解释 state、module、for_each、plan/apply、drift 和远程锁 |
| 高可用 | 至少跨两个 AZ，能讨论 NAT、ALB、ASG 和数据库的故障域 |
| 验证 | 有实际命令、连通性测试、日志和指标，而非只看 apply 成功 |
| 运营 | 能说明成本、监控、备份、回滚、RCA 和变更审计 |

### 常见扣分点

- 把“有公有 IP”当成 public subnet 的唯一条件。
- 在数据库 SG 中开放 `0.0.0.0/0`。
- 只部署一台 NAT，却声称架构无单点。
- 把 Terraform state 放在本地并让多人直接 apply 生产。
- 只会 `terraform apply`，不会 plan、import、drift 和回滚。
- 只说“加权限/重启实例”，没有基于证据的排查过程。
- 忽略加密、审计、备份恢复、数据分类和合规边界。

---

## 十、行为面试和沟通题

### 高频问题

1. 讲一个你亲自部署 VPC、子网和安全组的项目。
2. 讲一次 Terraform 变更导致故障或 plan 与预期不一致的经历。
3. 你如何向安全团队解释一个网络例外请求？
4. 当开发团队要求快速开放公网访问时，你如何处理？
5. 讲一次跨团队故障排查和 RCA。
6. 如何向非技术领导解释 RTO、风险和成本？
7. 你如何评估 Generative AI/Agentic AI 是否适合引入生产流程？

### 推荐回答结构：STAR + 工程细节

- **Situation**：业务、规模、限制和风险是什么。
- **Task**：你的责任边界和成功标准是什么。
- **Action**：具体做了哪些设计、命令、IaC、验证和沟通。
- **Result**：可量化结果，如部署时间、故障率、成本、恢复时间或审计通过率。
- **反思**：如果重做，会改善什么；如何把经验沉淀为 module、Runbook 或 guardrail。

### AI 相关回答方向

可以谈：

- 用大模型辅助生成 Terraform 草稿，但必须经过 provider 文档核对、`validate/plan`、安全扫描和人工审批。
- 用 Agent 读取受控的监控、日志和 Runbook，生成诊断建议；执行变更必须有权限边界、人工确认、审计和回滚。
- 医药数据不能因为使用 AI 就绕过数据分类、脱敏、访问控制、保留和供应商风险评估。
- 区分“生成建议”和“自动执行生产操作”，高风险动作应默认 fail-safe。

---

## 十一、面试前最后检查清单

### 必须能白板画出

- 双 AZ VPC：public/private subnet、IGW、NAT、路由表。
- ALB → App → DB 的 Security Group 引用关系。
- 私有应用访问 S3/ECR/Secrets Manager 的 Endpoint 路径。
- Terraform module、remote state、CI plan/apply 和审批流程。
- CloudFormation Stack、Change Set、Rollback 和 Drift 的关系。

### 必须能实际操作或解释

- `terraform fmt/init/validate/plan/apply/destroy`
- `terraform import` 或当前版本推荐的资源导入流程
- `aws ec2 describe-route-tables`、`describe-security-groups`
- `dig`、`curl -v`、`ss -lntp`、`df -h`、`journalctl`
- CloudTrail、VPC Flow Logs、ALB access logs、CloudWatch metrics
- SSM Session Manager、IAM Role、OIDC、ECR 镜像发布

### 最佳自我介绍重点

用 2 分钟说明一个真实项目：

1. 业务目标和约束。
2. VPC/CIDR/AZ/子网设计。
3. Terraform module 和 state 管理。
4. 安全、加密、审计和合规控制。
5. CI/CD、发布和回滚。
6. 一次真实故障、你的排查证据和最终改进。

最后应让面试官感受到：你不是只会调用 AWS API，而是能把**网络、计算、存储、安全、IaC、交付和运营**连成一个可维护、可审计、可恢复的生产系统。

---

## 十二、交付管理高频面试题：从方案到稳定运营

> 资深交付岗位不只是“按时把功能做出来”，还要把业务目标、范围、进度、质量、风险、成本、合规和上线后的运营结果连接起来。回答交付问题时，建议同时说明：**目标和边界、计划和责任、风险与决策、验证与验收、上线与移交、结果与复盘**。

### 12.1 交付经理必须建立的全局视角

| 交付维度 | 面试官关注点 | 应准备的证据 |
|---|---|---|
| 业务目标 | 为什么做、解决什么问题、成功如何衡量 | 业务指标、收益假设、目标用户 |
| 范围 | 做什么、不做什么、边界是否经过确认 | Scope baseline、WBS、验收标准 |
| 进度 | 里程碑、关键路径、依赖和缓冲是否可信 | 项目计划、依赖清单、趋势数据 |
| 资源 | 人员、技能、环境和预算是否匹配 | RACI、容量计划、资源缺口方案 |
| 质量 | 如何在上线前发现问题并阻止缺陷逃逸 | DoD、测试证据、质量门禁 |
| 风险 | 不确定性是否被提前识别和处置 | RAID Log、风险责任人、应对计划 |
| 变更 | 需求变化是否有影响分析和正式决策 | Change Request、决策记录 |
| 上线运营 | 是否可回滚、可观测、可支持、可恢复 | Go/No-Go、Runbook、Hypercare 计划 |
| 结果 | 是否实现业务价值，而不是只完成任务清单 | KPI、SLA、成本、稳定性和复盘报告 |

### 12.2 交付生命周期与治理

**高频问题 1：你如何管理一个从零开始的云项目交付？**

建议按阶段回答，并给出每个阶段的出口条件：

1. **发现与定义**：确认业务目标、用户、合规约束、范围边界、预算和成功指标。
2. **计划与设计**：拆分 WBS，识别关键路径、依赖和资源，完成架构、网络、安全和数据方案评审。
3. **构建与集成**：按迭代交付基础设施、应用和自动化流水线；所有变更进入版本控制和评审。
4. **测试与验收**：完成功能、集成、性能、安全、备份恢复、故障演练和用户验收。
5. **上线准备**：确认迁移/切换方案、监控告警、权限、Runbook、支持排班、沟通和回滚条件。
6. **上线与稳定期**：执行 Go/No-Go，按批准的切换计划发布，进入 Hypercare 并每日跟踪缺陷和指标。
7. **移交与复盘**：完成运维接管、文档和知识转移，核对收益指标并沉淀 RCA、改进项和资产清单。

每个阶段都应有明确的 **Entry Criteria、Exit Criteria、Owner 和审批人**，而不是到了上线前才集中发现缺口。

**高频问题 2：如何判断项目是否真正“准备好上线”？**

我会把上线准备拆成可检查的门禁，而不是由团队感觉决定：

- 业务：关键用户验收通过，业务窗口和客服/运营通知已确认。
- 技术：功能、集成、性能和安全测试达到阈值，严重缺陷为零或有书面豁免。
- 基础设施：Terraform/CloudFormation 变更已审批，环境配置、容量、配额和依赖已验证。
- 数据：迁移校验、备份、恢复和数据一致性检查完成；RPO/RTO 有测试证据。
- 运维：监控、日志、告警、Dashboard、Runbook、值班和升级路径已就绪。
- 发布：切换步骤、预计时长、观察指标、回滚触发条件和回滚演练已确认。
- 治理：Go/No-Go 会议完成，风险、遗留问题、负责人和截止时间均有记录。

**高频问题 3：如何平衡速度、质量和风险？**

不把它们当成简单的三选一。先区分不可妥协的安全、合规和数据完整性门禁，再把功能拆成最小可交付增量。对可接受的风险采用小范围、可观测、可回滚的方式发布，例如 canary、feature flag 或分批切换；对不可逆变更，则安排备份、演练、审批和更长观察窗口。任何为了提前上线而接受的风险，都应记录**影响、概率、缓解措施、责任人和到期时间**。

### 12.3 范围、需求与变更管理

**高频问题 4：如何防止需求蔓延（Scope Creep）？**

1. 在项目启动时建立范围基线：目标、交付物、排除项、假设、约束和验收标准。
2. 将需求拆成可验收的 Epic、User Story 或交付包，并为每项指定业务 Owner。
3. 新需求不能只问“要不要做”，而要评估对进度、成本、资源、架构、风险和运营的影响。
4. 通过正式 Change Request 记录变更，给出“增加范围、延长时间、增加资源或降低其他范围”的选项。
5. 由有授权的决策人批准后更新基线、计划、风险和沟通材料；未批准的需求进入 backlog，而不是默默加入本期。

**高频问题 5：需求不清晰但业务要求立即开始，你怎么做？**

先记录已知事实、假设、开放问题和决策截止时间，不把不确定性伪装成确定计划。组织短周期需求澄清会，用原型、架构草图或薄切片验证关键路径；对不能立即确认的部分设置假设和退出条件。可以先开展低风险、可复用的基础工作，但不能在关键设计和生产数据变更上无条件开工。最终让业务 Owner 对范围和验收标准签字或在系统中确认。

**高频问题 6：业务临时要求提前两周上线，如何处理？**

- 先确认“提前上线”对应的业务价值和真正的最晚日期。
- 对照关键路径分析哪些任务可以并行、拆分或缩小首发范围。
- 评估增加资源（crashing）、并行执行（fast-tracking）、分阶段发布和降低范围分别带来的风险。
- 明确不能压缩的安全、数据迁移、性能和回滚验证。
- 把选项、额外成本、残余风险和需要的决策写成一页纸，提交 Sponsor 决策，而不是由交付经理单方面承诺。

### 12.4 计划、依赖、资源与进度控制

**高频问题 7：你如何制定一个可信的交付计划？**

从可验收的交付物反推 WBS，再为任务补充 Owner、前置条件、估算、完成定义和验证方式。用历史数据或类比估算，关键任务使用三点估算并加入合理的风险缓冲；识别跨团队、供应商、环境、权限、网络、数据和审批依赖，计算关键路径。计划发布后不能只看“完成百分比”，还要跟踪里程碑趋势、关键路径偏差、剩余工作量和风险暴露。

云项目常见的前置依赖包括：账号和组织基线、网络 CIDR、DNS、证书、IAM 权限、配额、第三方白名单、测试数据、迁移窗口和运维值班。依赖清单应至少包含**依赖项、提供方、接收方、承诺日期、验证方式、当前状态和升级路径**。

**高频问题 8：项目已经延期，你会怎么处理？**

1. 先确认事实：延期影响哪个里程碑、是否在关键路径、根因是估算偏差、依赖延误、资源不足还是技术风险。
2. 重新计算剩余工作和最早可交付日期，区分可恢复的延误和不可恢复的约束。
3. 提供可比较的恢复方案：增加资源、调整顺序、分批上线、缩减范围、延长工作时间或接受新日期，并说明副作用。
4. 与 Sponsor 和受影响团队重新确认基线，更新风险、预算、沟通和承诺。
5. 对根因安排具体改进，例如提前完成环境准备、增加技术 Spike、补充缓冲或建立依赖 SLA。

不要通过隐藏未完成工作、虚报进度或压缩必要测试来“保住绿色状态”。

**高频问题 9：关键人员突然离岗，如何确保交付不受控外单点影响？**

先识别关键知识和关键路径任务，安排临时 Owner 和交接窗口；从设计、代码、Pipeline、环境、凭证管理、决策记录和 Runbook 中恢复可交接信息。重新评估容量和日期，必要时调整范围或寻求支援。长期措施包括结对工作、关键模块文档化、轮岗、备份负责人和减少只有一个人掌握的生产权限。

### 12.5 风险、问题、决策和升级

**高频问题 10：风险（Risk）和问题（Issue）有什么区别？你如何管理？**

- **风险**是可能发生的未来事件，管理重点是概率、影响、触发器和预防/缓解措施。
- **问题**是已经发生并影响交付的事实，管理重点是临时处置、责任人、解决期限和升级。
- **依赖**是需要其他团队或外部方按约定提供的输入。
- **决策**是需要有授权的人在选项之间做取舍。

使用统一的 RAID Log 管理它们，每项至少包含描述、影响、Owner、截止日期、状态、下一步和升级条件。高风险不能只标红，还应有可执行的应对方案，例如规避、降低、转移、接受或准备应急路径。

**高频问题 11：什么时候应该升级问题？**

当问题超出个人授权范围、可能影响关键路径或合规门禁、需要跨团队资源、预计超过约定 SLA，或已有措施无法在截止日期前恢复时应升级。升级时带上事实、影响、已尝试措施、可选方案、推荐方案、需要谁在何时做什么决定；不要只发送“项目有风险”而不给决策材料。

**高频问题 12：如何向高管汇报项目状态？**

使用一页式、趋势化的报告，内容包括：

- 本周期已完成的业务结果和下一周期目标。
- 关键里程碑、计划日期、实际/预测日期和趋势。
- 需要高管决策或资源支持的事项。
- Top 风险/问题、影响、Owner、缓解措施和到期日。
- 范围、预算、质量、依赖和上线准备度。

状态颜色必须有客观规则。例如“红色”应对应明确的阈值或需要管理层介入的事实，而不是为了维持表面上的绿色。报告应区分“已完成”“预计完成”和“等待外部输入”。

### 12.6 干系人、沟通与跨团队协作

**高频问题 13：开发、运维、安全和业务对方案意见不一致，你如何推进？**

先把各方的目标、约束和不可妥协项写清楚，避免把技术偏好当成事实。组织结构化评审，基于风险、成本、交付日期、合规和可运营性比较选项；必要时用 PoC 或数据验证争议点。明确最终决策人和决策截止时间，记录决策理由、前提和后续动作。决策后即使不是所有人首选，也要让执行团队知道边界和成功标准。

**高频问题 14：你如何建立 RACI，避免“大家都以为别人负责”？**

对每个关键交付物和决策点指定：

- **R（Responsible）**：实际执行的人或团队。
- **A（Accountable）**：对结果负责且有决策权的人，原则上每项只有一个。
- **C（Consulted）**：需要在决策前提供意见的人。
- **I（Informed）**：需要及时获知结果的人。

RACI 不应替代沟通。发布前还要补充工作流、审批人、升级路径、值班人和验收人，并在项目变化时更新。

**高频问题 15：如何处理一个不配合的关键干系人？**

先判断是目标不一致、信息不足、资源冲突、风险担忧还是职责不清，而不是直接贴标签。用对方关心的业务影响说明需要的输入，给出明确日期、最小决策包和不响应的影响；仍无法解决时，按事先约定的治理路径升级。全过程保留事实和决定，避免公开指责，保护长期合作关系。

### 12.7 质量、验收与合规交付

**高频问题 16：你如何定义“完成”（Definition of Done）？**

“代码合并”或“资源创建成功”不等于完成。对云交付，一个可用的 DoD 至少包括：

- 需求和验收标准已满足，自动化测试和必要的人工测试通过。
- IaC、代码、配置和依赖已评审并可重复部署。
- 安全、权限、加密、漏洞、合规和数据分类检查通过。
- 性能、容量、备份恢复、监控、日志和告警已验证。
- 文档、Runbook、架构图、CMDB/资产信息和知识转移完成。
- 已知缺陷有等级、Owner、截止日期和业务豁免记录。

**高频问题 17：UAT 失败但上线日期不能变，怎么处理？**

先按影响和严重度分类，确认失败是否阻断核心业务、数据完整性、安全或合规。对可隔离的问题，可以通过关闭受影响功能、分批发布、feature flag 或延后非核心范围来降低影响；对核心路径或高风险问题，不应仅因为日期压力而强行上线。组织业务、技术、安全和运营共同做 Go/No-Go 决策，形成书面豁免、补救计划和回滚条件。

**高频问题 18：如何保证医药或强监管环境中的交付可审计？**

建立从需求、设计、代码/IaC、测试、审批、发布、变更、访问和故障到验收的可追溯链路。采用职责分离和最小权限，保留不可抵赖的审批、CloudTrail、部署日志、测试报告、数据迁移校验和恢复演练证据。合规不是最后补一份文档，而是把控制点嵌入需求评审、Pipeline、变更窗口和运行监控。

### 12.8 上线、回滚与生产稳定期

**高频问题 19：请说明一次标准的生产上线流程。**

1. 冻结版本、配置和 Terraform plan，确认变更范围与审批记录。
2. 核对前置条件：备份、容量、权限、依赖、证书、DNS、白名单和维护窗口。
3. 发布前通知受影响方，建立指挥频道、值班表和升级链路。
4. 按 Runbook 执行，记录每一步的开始时间、执行人和验证结果。
5. 用健康检查、错误率、延迟、吞吐、资源、业务交易和依赖指标进行分阶段验证。
6. 在预设观察窗口内判断继续、暂停或回滚；达到回滚阈值时优先恢复服务。
7. 发布结束后更新版本、资产、文档和状态报告，进入 Hypercare。

**高频问题 20：什么情况下应该回滚？回滚和修复有什么区别？**

当核心业务不可用、错误率或延迟超过阈值、数据完整性受威胁、安全门禁失败，或问题不能在批准窗口内安全定位时，应优先执行已验证的回滚或切换。回滚是恢复稳定版本，修复是解决根因；不能把“现场临时改几项配置”当成可审计的回滚。数据库 schema、数据迁移和不可逆基础设施变更必须在设计阶段准备前向兼容、备份恢复或补偿方案。

**高频问题 21：上线后如何判断项目真正稳定？**

预先定义 Hypercare 的进入和退出条件，例如连续若干个业务高峰周期内，错误率、延迟、容量、告警、工单、数据校验和批处理均达到目标；严重缺陷已关闭或有批准的后续计划；运维团队已能独立处理常见告警。稳定期不是无限延长的“项目尾巴”，应有明确期限、每日/每周指标和最终移交签字。

**高频问题 22：如何做项目到运维的移交？**

移交包至少包含架构和数据流图、资源清单、依赖关系、SLA/SLO、监控和告警说明、Runbook、备份恢复与灾备方案、权限和密钥轮换方式、发布/回滚流程、常见故障、供应商联系人、已知问题和培训记录。通过演练验证接收团队能独立完成部署、排障、扩容和恢复，而不是只召开一次演示会议。

### 12.9 成本、供应商和交付指标

**高频问题 23：你如何在交付中控制云成本？**

先建立预算基线和成本归属标签，再按环境、团队、产品和项目跟踪实际消耗与预测。设计阶段做容量和架构权衡，例如 NAT Gateway 与 VPC Endpoint、实例规格与弹性、日志保留、存储生命周期和非生产环境自动关停。每次优化都要确认不会破坏安全、可用性、RTO/RPO 或交付日期，并把节省金额、一次性投入和运营影响写进决策记录。

**高频问题 24：如何管理云厂商或第三方供应商？**

在合同或 SOW 中明确交付物、接口、责任边界、里程碑、验收标准、SLA、数据和安全要求、升级路径及变更流程。不要只依赖供应商口头承诺；将外部依赖纳入 RAID Log，定期检查可交付证据、环境访问、漏洞修复、备份和退出方案。关键供应商应准备替代路径或降级方案，避免形成不可替代的单点。

**高频问题 25：你用哪些指标评价交付质量？**

指标应覆盖结果、过程和稳定性，而不是只看“是否按时完成”：

- 计划：里程碑按时率、关键路径偏差、计划完成趋势。
- 范围：需求变更数量、未批准范围、验收一次通过率。
- 质量：缺陷逃逸率、严重缺陷数、返工率、自动化测试覆盖。
- 发布：Change Failure Rate、回滚率、发布频率、Lead Time。
- 运营：可用性、SLO、MTTR、告警噪声、工单量。
- 财务：预算偏差、云成本偏差、单位业务成本和节省收益。
- 客户：业务满意度、上线采用率、目标收益实现率。

指标必须有数据来源、统计口径、Owner 和改进动作，避免为了好看而选择无法验证的数字。

### 12.10 典型交付场景题

**场景一：上线前一天发现关键依赖团队没有完成白名单。**

先确认影响的真实范围和最早完成时间，暂停依赖该白名单的步骤，不要绕过安全控制临时开放公网。召集网络、安全、供应商和业务 Owner，给出完成、降级、分批上线和延期选项；若采用临时例外，必须限定来源、时长、审批人、监控和自动撤销时间。更新 Go/No-Go 结论和风险记录。

**场景二：发布过程中错误率持续上升，但业务负责人要求继续观察。**

依据发布前约定的指标阈值和回滚条件执行，而不是现场重新争论标准。暂停扩散、保留证据、评估影响并按 Runbook 回滚或切流；同步业务影响和预计恢复时间。服务恢复后再通过 RCA 判断是代码、配置、容量、依赖还是发布流程问题，并补充测试或门禁。

**场景三：项目范围和预算都已锁定，但安全评审新增强制控制。**

先确认控制要求的适用范围和截止日期，评估架构、工期、成本和上线风险。与 Sponsor 讨论增加预算/时间、缩减其他范围、分阶段合规或采用临时补偿控制的方案；不能把未评估的安全要求悄悄塞进现有承诺。批准后同步更新基线、计划、风险、验收标准和合同边界。

**场景四：项目按时上线，但上线后三周业务采用率很低。**

这说明交付完成不等于价值实现。分析用户流程、性能、培训、数据质量、权限、功能匹配和运营支持，结合采用率、成功交易率、工单和访谈定位原因。与业务 Owner 定义改进 Backlog 和收益目标，安排短迭代验证，直到达到事先约定的价值指标，再完成正式收尾。

### 12.11 交付行为题的推荐回答结构

对于“讲一次成功/失败交付经历”，可使用 **STAR + Delivery Controls**：

1. **Situation**：业务目标、项目规模、时间约束、团队和技术背景。
2. **Task**：你的职责、授权范围和成功标准。
3. **Plan**：如何拆范围、排计划、分配责任、识别风险和设计质量门禁。
4. **Action**：你做了哪些关键决策、沟通、协调、验证、上线和风险处置。
5. **Result**：按时率、成本、缺陷、SLA、恢复时间、采用率或审计结果等量化结果。
6. **Governance**：哪些决策经过谁批准，如何留下证据和可追溯记录。
7. **Learning**：复盘后把什么沉淀为模板、自动化检查、Runbook、架构标准或流程改进。

避免只说“我协调了很多团队”或“项目最终上线了”。资深交付回答应让面试官听到：你如何做取舍、如何让风险透明、如何建立可验证的承诺，以及如何保证上线后的结果持续达标。
