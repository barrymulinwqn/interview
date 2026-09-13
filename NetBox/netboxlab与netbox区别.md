# NetBox、NetBox Labs 与 NetBox Lab 的区别

## 1. 名称说明

“netboxlab”可能指两个不同的概念：

1. **NetBox Labs**：围绕 NetBox 建立的商业公司和产品生态。
2. **NetBox Lab**：非正式说法，通常指用户自己搭建的 NetBox 学习、测试或验证环境。

正式品牌通常写作 **NetBox Labs**，中间有空格，并且是复数 `Labs`。NetBox Lab 则不是 NetBox Labs 的官方产品名称。

## 2. 什么是 NetBox？

NetBox 是一个开源的网络基础设施资源建模和文档平台，核心定位是网络领域的 **Source of Truth（权威数据源）**。

它主要提供以下能力：

- **IPAM**：IP 地址、前缀、VRF、VLAN、ASN 等。
- **DCIM**：区域、站点、位置、机柜、设备、接口、电源等。
- **网络连接**：电缆、配线架端口、无线连接和链路关系。
- **虚拟化资源**：集群、虚拟机和虚拟接口。
- **线路资源**：运营商、电路和电路端点。
- **自动化与扩展**：REST API、GraphQL API、Webhook、插件、自定义字段、自定义校验和自定义脚本。

例如，一台交换机可以在 NetBox 中关联到站点、机柜、设备角色、接口、管理 IP、VLAN 和电缆。自动化工具再通过 API 查询这些信息，生成配置或设备清单。

NetBox 通常保存网络基础设施的结构化事实和设计意图，但它本身不是以下系统：

- 传统网络监控系统（NMS）。
- 自动发现和性能监控平台。
- 网络设备配置下发平台。
- 完整的 ITSM、工单系统或企业 CMDB。

典型工作流程如下：

```text
NetBox 保存设备和网络意图
        ↓
Ansible、Python、Nornir 等工具查询 NetBox
        ↓
生成或下发设备配置
        ↓
监控系统检查实际运行状态
        ↓
发现状态漂移后触发审计或同步
```

NetBox 是基于 Python 和 Django 构建的开源项目，采用 Apache 2.0 许可证。它可以自行部署，也可以使用社区维护的 Docker 镜像或 Kubernetes Helm Chart 部署。

## 3. 什么是 NetBox Labs？

NetBox Labs 是围绕 NetBox 提供商业服务、企业支持和扩展产品的公司与平台生态。

它的产品和服务方向包括：

- NetBox Cloud 托管服务。
- NetBox Enterprise 企业部署方案。
- NetBox Discovery 基础设施发现能力。
- NetBox Assurance 运行状态与设计意图对比能力。
- 企业级支持、安全、运维和自动化能力。
- 面向 AI、自然语言查询和基础设施操作的相关产品。

NetBox Labs 不等于 NetBox 的另一个开源版本。更准确的关系是：

```text
NetBox
  = 开源核心平台

NetBox Labs
  = 围绕 NetBox 提供商业服务、托管平台和扩展产品的公司与生态
```

如果只从 GitHub 获取 NetBox 源码并自行部署，使用的是开源 NetBox；如果使用 NetBox Labs 提供的托管、企业支持或配套产品，则属于 NetBox Labs 的商业生态范围。

## 4. NetBox 与 NetBox Labs 的区别

| 对比项 | NetBox | NetBox Labs |
| --- | --- | --- |
| 本质 | 开源软件项目 | 商业公司和产品生态 |
| 核心内容 | IPAM、DCIM、网络资源建模、API 和扩展 | 托管、企业支持、发现、保障、AI 和平台能力 |
| 部署方式 | 用户自行部署和维护 | 可以使用云托管或企业服务 |
| 数据库与缓存 | 通常由用户自行维护 PostgreSQL 和 Redis/Valkey | 托管方案通常由服务商负责相关基础设施 |
| 升级与维护 | 用户自行规划和执行 | 可获得厂商协助或由服务商代为维护，具体取决于方案 |
| 许可证 | 开源，Apache 2.0 | 商业服务和产品按具体方案收费 |
| 适用场景 | 学习、自建平台、实验环境和自动化团队 | 需要 SLA、商业支持、托管服务和企业能力的组织 |
| 是否等价 | 不能等同于 NetBox Labs | 包含 NetBox 相关服务，但不等于 NetBox 本身 |

## 5. 什么是 NetBox Lab？

NetBox Lab 通常只是一个实验环境名称，不是一个独立的软件产品。例如，一个本地实验环境可能包含：

```text
实验主机
 ├── PostgreSQL
 ├── Redis 或 Valkey
 ├── NetBox Web
 └── NetBox Worker
```

它可以用于：

- 学习 NetBox 的数据模型。
- 测试 REST API 和 GraphQL API。
- 调试插件和 Custom Script。
- 测试 Ansible、Terraform 或 Nornir 集成。
- 演练数据导入、备份、迁移和升级。
- 验证 Webhook、权限和自动化流程。

实验环境与生产环境的主要区别在于可靠性和运维要求。实验环境可以使用本地 PostgreSQL、Django 开发服务器和单实例 Redis；生产环境则应考虑 HTTPS、反向代理、访问控制、备份、恢复演练、高可用、监控、密钥管理和版本固定。

## 6. 三者的关系

可以用下面的关系理解：

```text
NetBox
  └── 开源核心软件

NetBox Labs
  ├── NetBox Cloud
  ├── NetBox Enterprise
  ├── Discovery
  ├── Assurance
  └── 其他商业支持和扩展能力

NetBox Lab
  └── 用户自行搭建的学习或测试环境
```

也可以进行一个简单类比：

- **NetBox**：开源核心软件。
- **NetBox Labs**：围绕该软件提供商业服务和企业产品的厂商及生态。
- **NetBox Cloud**：厂商托管的云服务版本。
- **NetBox Lab**：用户自己搭建的实验环境。

## 7. 如何选择？

### 选择自建 NetBox

适合以下情况：

- 想学习 NetBox 原理和数据模型。
- 需要完全控制数据、网络和部署方式。
- 具备 Linux、Docker 或 Kubernetes 运维能力。
- 需要自行开发插件、脚本和自动化流程。
- 可以接受自己负责备份、升级和故障处理。

### 选择 NetBox Cloud 或企业方案

适合以下情况：

- 不希望自行维护 PostgreSQL、Redis、应用和备份系统。
- 需要厂商支持、SLA 或企业服务。
- 希望更快上线并降低平台运维成本。
- 需要企业级发现、状态保障、审计或扩展能力。
- 团队规模较大，期望使用标准化的商业运维方式。

## 8. 一句话总结

> **NetBox 是开源的网络基础设施事实库和自动化数据源；NetBox Labs 是围绕 NetBox 提供商业托管、企业支持和扩展能力的公司与平台生态；NetBox Lab 通常只是用户自己搭建的学习或测试环境。**

## 参考资料

- [NetBox 官方文档](https://netboxlabs.com/docs/netbox/en/stable/)
- [NetBox Labs 官方网站](https://netboxlabs.com/)
- [NetBox 开源项目](https://github.com/netbox-community/netbox)
- [本仓库的 NetBox 基础面试题](netbox-基础高频面试题.md)
- [本仓库的 NetBox 本地搭建文档](setup/setup.md)
