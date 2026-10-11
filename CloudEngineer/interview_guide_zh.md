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
