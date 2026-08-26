# ServiceNow 基础知识与运维面试题

面向 ServiceNow 管理员、ITSM 运维工程师、平台支持工程师和实施顾问岗位的面试复习资料。回答问题时，建议同时体现四个维度：**业务流程理解、平台配置能力、故障排查方法、变更与合规意识**。

---

## 一、ServiceNow 平台定位

### 1. ServiceNow 是什么

ServiceNow 是基于云的企业级工作流平台，使用表驱动的数据模型承载 ITSM、ITOM、ITBM、HRSD、CSM、SecOps 等业务能力。它的核心思想不是单纯记录工单，而是把请求、事件、问题、变更、配置项、知识和审批组织成可追踪、可自动化的业务流程。

面试中可以这样概括：

- **数据层**：表、字段、引用关系、继承关系、字典配置。
- **流程层**：Flow Designer、Workflow、审批、任务分派、SLA。
- **逻辑层**：Business Rule、Script Include、Client Script、UI Policy。
- **展现层**：表单、列表、Workspace、Service Portal、报表和仪表盘。
- **集成层**：REST、SOAP、Import Set、Transform Map、MID Server、消息队列等。
- **治理层**：角色权限、审计、更新集、应用管理、发布和升级控制。

### 2. 实例与应用范围

- **Instance**：一个独立的 ServiceNow 环境，例如开发、测试、预生产和生产实例。
- **Application**：功能域或应用作用域，例如 Incident、Change、CMDB、Service Catalog。
- **Scope**：应用的命名空间和隔离边界。应用作用域可以限制脚本、表和资源的访问范围。
- **Domain Separation**：在同一实例中为不同租户或组织隔离数据、流程和配置。
- **Update Set**：迁移配置变更的传统机制；它主要记录配置元数据，不是完整的软件版本管理方案。

---

## 二、平台数据模型基础

### 1. 表、字段与继承

ServiceNow 的数据以表为中心。表可以有父表和子表，子表会继承父表的字段和行为。

常见继承关系：

```text
task
├── incident
├── problem
├── change_request
├── sc_request
├── sc_req_item
└── sc_task
```

`task` 是很多工作任务表的基础表，因此任务编号、描述、分配组、负责人、状态、优先级等字段常见于多个模块。

需要掌握的概念：

- **Dictionary Entry**：定义字段的数据类型、长度、默认值、引用表和属性。
- **Reference Field**：保存另一张表的 `sys_id`，界面显示被引用记录的可读字段。
- **Choice Field**：使用选项值表示状态、类型或分类；修改 choice 的 value 需要谨慎。
- **Display Value**：引用记录对用户展示的字段，不等于数据库实际保存的 `sys_id`。
- **`sys_id`**：记录的 32 位唯一标识。集成和脚本中应优先使用稳定的 `sys_id` 或明确的业务键。
- **Journal Field**：如 Work notes、Additional comments，通常以追加方式记录，不应被当作普通字符串覆盖。
- **Audit**：记录字段变化的审计信息；需结合审计配置和保留策略判断是否完整。

### 2. 重要系统表

| 表 | 用途 |
|---|---|
| `sys_user` | 用户、登录名、部门、经理、语言及时区 |
| `sys_user_group` | 用户组、分派组、审批组 |
| `sys_user_grmember` | 用户与组的关联关系 |
| `sys_dictionary` | 表和字段的字典定义 |
| `sys_db_object` | 表定义 |
| `sys_properties` | 系统属性 |
| `sys_script` | Business Rule 等服务器脚本定义 |
| `sys_script_client` | Client Script 定义 |
| `sys_ui_policy` | UI Policy 定义 |
| `sys_security_acl` | ACL 规则 |
| `syslog_transaction` | 事务执行、耗时和错误信息 |
| `sys_update_xml` | 更新集中的配置变更记录 |
| `sys_attachment` | 附件元数据和关联关系 |
| `sys_audit` | 被审计字段的历史变化 |

生产环境中直接修改系统表或批量更新数据前，应先确认影响范围、备份策略、审计要求和回滚方式。

### 3. 查询与脚本常识

服务器端 GlideRecord 的基本结构：

```javascript
var incident = new GlideRecord('incident');
incident.addQuery('active', true);
incident.addQuery('priority', 1);
incident.query();

while (incident.next()) {
    gs.info(incident.getValue('number'));
}
```

常见方法：

- `addQuery()`：增加条件。
- `addEncodedQuery()`：使用列表过滤器生成的 encoded query。
- `addActiveQuery()`：查询活动记录。
- `setLimit()`：限制返回数量。
- `orderBy()`、`orderByDesc()`：排序。
- `get()`：按 `sys_id` 或字段值取得单条记录。
- `insert()`、`update()`、`deleteRecord()`：写入或删除记录。
- `setValue()`、`getValue()`、`getDisplayValue()`：处理字段值和显示值。
- `setWorkflow(false)`：在明确评估副作用后，控制是否触发工作流相关逻辑。
- `autoSysFields(false)`：在特殊数据修复中控制系统字段更新，必须经过审批。

脚本原则：

1. 查询时明确条件，避免无条件扫描或无条件更新整张表。
2. 循环中避免重复查询同一引用记录，必要时使用缓存、批量策略或聚合查询。
3. 不要把用户输入直接拼接进脚本或查询逻辑。
4. 生产修复脚本必须可审计、可重入、有限量，并先在非生产实例验证。
5. 记录关键日志，但不要输出密码、令牌、个人敏感信息或完整请求体。

---

## 三、ITSM 核心流程

### 1. Incident Management（事件管理）

目标是尽快恢复受影响的服务，而不是立即找出永久根因。

典型流程：

```text
报告/监控发现
    -> 记录 Incident
    -> 分类与优先级评估
    -> 初步诊断
    -> 分派与升级
    -> 恢复服务
    -> 用户确认/自动关闭
    -> 复盘并关联 Problem
```

关键字段与概念：

- **Impact**：影响范围。
- **Urgency**：解决的紧迫程度。
- **Priority**：通常由 Impact 和 Urgency 矩阵计算得出。
- **Assignment group**：当前负责处理的团队。
- **Major Incident**：高影响事件，需要更严格的沟通、指挥和升级机制。
- **SLA**：响应和解决的服务承诺计时。
- **Knowledge**：将已验证的解决方案沉淀为知识文章。

事件管理的好答案应区分：先恢复业务、再分析根因；高优先级事件要先建立统一沟通渠道和责任人。

### 2. Problem Management（问题管理）

Problem 关注一个或多个事件背后的根因，以及如何减少重复发生。常见活动包括：

- 关联重复 Incident。
- 记录症状、影响和已知错误。
- 使用 RCA（Root Cause Analysis）分析根因。
- 创建并跟踪 Known Error。
- 通过 Change 实施永久修复。
- 关闭前验证事件复发率和知识沉淀情况。

### 3. Change Management（变更管理）

常见变更类型：

- **Standard Change**：低风险、频繁、已预批准且有明确步骤的变更。
- **Normal Change**：需要评估风险、审批、排期和实施验证。
- **Emergency Change**：为快速处理重大故障或安全风险而执行，仍需保留最小必要审批和事后复盘。

变更记录至少应能回答：

- 为什么改，影响什么服务和配置项？
- 谁批准，谁实施，谁验证？
- 什么时候实施，维护窗口是什么？
- 失败时如何回滚？
- 成功或失败的验收标准是什么？

### 4. Request、Catalog 与 Knowledge

- **Catalog Item**：服务目录中的可申请项目。
- **Record Producer**：通过目录变量创建目标表中的记录，例如 Incident。
- **Order Guide**：一次申请多个相关目录项目。
- **RITM**：Requested Item，申请中的具体目录项。
- **Catalog Task**：交付目录项目时分派给执行团队的任务。
- **REQ**：Request，用户的一次总体请求。
- **Variable / Variable Set**：目录项输入变量及可复用变量集合。

目录流程设计应关注申请人、审批人、交付组、SLA、变量校验、权限、通知和失败补偿，不能只完成“提交后创建一条记录”。

### 5. CMDB 与服务模型

CMDB 用于管理 CI（Configuration Item）及其关系。常见 CI 类型包括服务器、数据库、应用、网络设备和业务服务。

关键概念：

- **CI Class**：CI 的类别和继承结构。
- **CI Relationship**：例如“运行于”“依赖于”“连接到”。
- **CSDM**：Common Service Data Model，用于统一服务、应用和技术对象的建模方式。
- **Discovery**：自动发现基础设施并更新 CI。
- **Service Mapping**：发现业务服务的技术依赖关系。
- **IRE**：Identification and Reconciliation Engine，负责识别重复 CI 并按数据源优先级协调更新。
- **Data Certification**：定期确认 CI 数据的准确性和完整性。
- **CMDB Health**：检查重复、孤立、过期、缺少关系或必填字段不完整等问题。

面试中需要强调：CMDB 不是简单资产清单，价值在于支持事件影响分析、变更风险评估、服务可用性分析和审计。

---

## 四、ServiceNow 管理员必须掌握的配置

### 1. 用户、组和角色

基本授权链路通常是：

```text
User -> Group membership -> Role -> ACL / Application access
```

- 角色遵循最小权限原则。
- 组主要用于分派、通知和审批，不等同于授权本身。
- 对管理员角色、脚本角色和安全角色进行严格控制。
- 离职、转岗用户应及时冻结或移除不再需要的角色。
- 共享账号会削弱审计追踪，应尽量避免。

### 2. ACL（Access Control List）

ACL 可以控制表、记录或字段的访问，常见操作包括 `create`、`read`、`write`、`delete` 和 `execute`。

排查权限问题时按以下顺序：

1. 确认用户实际登录身份、角色和组成员关系。
2. 确认访问的是哪张表、哪条记录、哪个字段和哪个操作。
3. 检查表级、字段级和记录级 ACL。
4. 检查条件脚本、Required role、条件和脚本的返回值。
5. 使用安全调试工具验证命中的 ACL。
6. 检查 Domain Separation、应用作用域和数据过滤条件。

不要用“给用户加管理员角色”作为权限问题的正式解决方案；这会绕过真正的授权设计并扩大风险。

### 3. UI Policy、Client Script 与 Business Rule

| 机制 | 执行位置 | 典型用途 | 常见风险 |
|---|---|---|---|
| UI Policy | 客户端 | 显示、隐藏、只读、必填 | 只影响界面，不能代替安全控制 |
| Client Script | 客户端 | 表单加载、字段变化、提交校验 | 过多脚本导致表单慢或逻辑冲突 |
| Business Rule | 服务端 | 插入、更新、删除前后处理 | 递归更新、性能问题、隐式副作用 |
| Data Policy | 服务端及数据入口 | 统一字段校验 | 与客户端规则不一致 |
| Script Include | 服务端，可按配置调用 | 复用业务逻辑 | 作用域、权限和客户端可调用边界不清 |
| Flow Designer | 平台流程层 | 自动化、审批、集成 | 触发条件不严谨导致重复执行 |

原则：客户端逻辑改善用户体验，服务端逻辑保证数据和业务规则；安全控制必须放在服务端。

### 4. Flow Designer、Workflow 与事件

新开发通常优先考虑 Flow Designer，但要根据现有版本、应用范围和流程复杂度评估。排查自动化时重点查看：

- 触发条件是否满足。
- 流程是否处于激活状态。
- Flow 是否失败、等待审批或被暂停。
- 运行用户是否有访问目标表和字段的权限。
- 是否因更新记录再次满足触发条件而重复执行。
- 事件、通知、Script Action 或集成步骤的输入是否正确。
- 是否设置了超时、重试和失败通知。

### 5. SLA 与计时

SLA 排查必须先区分：SLA 定义、SLA 实例和日历计时。

- **SLA Definition**：定义何时启动、暂停、停止、取消，以及目标时长。
- **Task SLA**：某条任务上的实际计时实例。
- **Schedule**：工作时间、节假日和时区配置。
- **Pause condition**：例如等待用户信息时暂停计时。
- **Breach**：超过目标时间。

典型排查路径：

1. 确认目标任务满足 SLA 的 start condition。
2. 确认是否存在排除条件或重复 SLA 定义。
3. 查看 Task SLA 的 stage、planned end time、business elapsed time。
4. 核对 schedule、时区、节假日和暂停条件。
5. 确认优先级、服务、组等动态条件在任务生命周期中是否改变。
6. 检查后台 SLA 引擎执行和相关日志。

---

## 五、集成与数据导入

### 1. REST 与 SOAP

ServiceNow 常见集成模式：

- **Inbound REST/SOAP**：外部系统调用 ServiceNow API。
- **Outbound REST/SOAP**：ServiceNow 调用外部系统。
- **Import Set**：先把外部数据导入暂存表，再通过 Transform Map 写入目标表。
- **MID Server**：让云端实例安全访问企业内网资源或执行 Discovery。
- **IntegrationHub**：通过 Spoke 和动作连接外部系统。

设计接口时需要明确：

- 认证方式：OAuth、Basic Auth、API key 或互证书，优先使用安全且可轮换的方案。
- 数据契约：字段类型、必填字段、枚举值、时区和编码。
- 幂等键：避免重试导致重复创建。
- HTTP 状态码、错误体、超时和重试策略。
- 速率限制、分页、批量大小和并发量。
- 敏感数据脱敏、日志保留和审计要求。
- 失败消息的告警、补偿和人工重放方式。

### 2. Import Set 与 Transform Map

标准导入流程：

```text
外部文件/API
    -> Import Set Table（暂存表）
    -> Transform Map
    -> Coalesce 匹配已有记录
    -> Field Map / Transform Script
    -> 目标表
    -> 导入日志与异常处理
```

**Coalesce** 用于决定如何识别已有记录。选择业务唯一键时，要避免使用会变化的显示名称。导入前需要验证编码、日期格式、空值策略、引用字段匹配、重复数据和错误行处理。

### 3. MID Server

MID Server 是安装在企业网络内的代理组件，用于 Discovery、Orchestration 和访问内网系统。运维重点包括：

- 服务运行状态和版本兼容性。
- 到 ServiceNow 实例的出站网络连通性。
- Java 运行环境和服务账号权限。
- ECC Queue 中的输入、输出和错误状态。
- 证书、代理、防火墙和 DNS 配置。
- 多个 MID Server 的 Capability、负载和 failover 配置。

排查时不要只看 MID Server 主机“进程还在”，还要确认任务是否进入队列、是否被正确的 MID Server 选中、命令是否在目标网段成功执行。

---

## 六、平台运维、发布与升级

### 1. 实例健康与日常巡检

建议建立可量化的巡检清单：

- 关键业务模块可用性和登录情况。
- 系统错误、集成失败、Flow 失败和导入异常。
- 事务耗时、慢请求和异常增长趋势。
- 事件、变更、目录请求的积压和 SLA breach。
- 计划任务、Scheduled Job 和数据同步是否按时运行。
- MID Server、Discovery、Service Mapping 状态。
- 磁盘/附件/日志/审计保留策略是否符合要求。
- 账号、角色、证书、OAuth token 和密钥的到期情况。
- 近期发布、更新集和失败回滚情况。

### 2. 更新集与应用发布

传统更新集发布流程：

```text
开发实例创建变更
    -> Review / 依赖检查
    -> 标记完成
    -> 在测试实例预览
    -> 解决 Preview Problems
    -> 提交并测试
    -> 生产窗口发布
    -> 验收、监控、必要时回滚
```

注意事项：

- 更新集是配置迁移机制，不应当替代代码审查、测试和发布审批。
- 检查依赖更新集、顺序、作用域和目标实例差异。
- 生产发布前准备验证脚本、回滚方案和业务联系人。
- 对数据修复、用户、组、凭证和外部配置进行单独管理。
- 对应用开发使用适合的应用仓库或源代码管理流程。
- 不要把临时调试配置、测试数据和敏感凭证带入生产。

### 3. 升级管理

升级前：

- 阅读版本说明、弃用项和插件依赖变化。
- 盘点自定义脚本、UI、集成、报表和关键流程。
- 建立升级测试范围和业务验收标准。
- 备份或确认平台恢复能力，锁定变更窗口。
- 在子生产实例执行升级并记录问题。

升级后：

- 验证登录、核心 ITSM 流程、通知、SLA、目录和集成。
- 检查自定义对象、系统错误和性能变化。
- 执行关键业务回归和接口端到端测试。
- 跟踪升级后问题，必要时通过官方修复或回退方案处理。

### 4. 性能排查

一个可靠的性能排查顺序：

1. 明确是全局慢、单模块慢、单用户慢还是单条记录慢。
2. 确认发生时间、请求 URL、用户、浏览器和操作步骤。
3. 查看事务日志、慢事务、系统日志和服务器端脚本耗时。
4. 判断瓶颈在客户端脚本、服务端查询、流程、集成等待还是外部系统。
5. 检查是否有无索引查询、大量循环查询、同步远程调用或批量更新。
6. 在非生产环境复现并测量改动前后效果。
7. 发布修复后观察错误率、响应时间和业务指标。

常见优化方向：

- 收窄 GlideRecord 查询条件并限制返回量。
- 避免在循环内反复查询引用表。
- 减少表单加载时的同步 GlideAjax 和不必要字段。
- 将大批量处理改为分批、异步或定时执行。
- 避免 Business Rule 递归触发和重复更新。
- 对高频查询字段评估索引，但索引会增加写入成本，需用数据验证。

---

## 七、运维面试高频题与参考答案

### Q1：Incident、Problem 和 Change 有什么区别？

**参考答案：** Incident 的目标是尽快恢复服务；Problem 的目标是识别和消除一个或多个事件的根因；Change 是对生产环境、服务或配置项进行受控修改。一个 Incident 可以关联到 Problem，而 Problem 的永久修复通常通过 Change 实施。高质量回答还应说明三者在优先级、SLA、审批和关闭条件上不同。

### Q2：Priority 如何确定？Impact 和 Urgency 有什么区别？

**参考答案：** Impact 描述受影响的用户、服务或业务范围，Urgency 描述需要多快处理。Priority 通常由二者通过矩阵计算，不能只由报告人的主观感受决定。实际运维中还要确认 Major Incident 标准、业务关键性、服务等级和升级路径。

### Q3：如何处理一个 P1 生产事件？

**参考答案：**

1. 确认影响范围、开始时间和受影响服务，建立或更新 P1 Incident。
2. 任命 Incident Commander、技术负责人和沟通负责人。
3. 建立统一桥接会议或协作频道，避免多人重复操作。
4. 先采取低风险措施恢复服务，例如回滚、切换、扩容或禁用故障功能。
5. 按固定频率向业务和管理方更新事实、影响、行动和下一次更新时间。
6. 记录每个操作、时间、操作者和结果，避免未经评估的临时改动。
7. 服务恢复后验证监控、交易/业务链路和用户体验。
8. 创建 Problem，完成 RCA、时间线、改进项和必要的永久修复 Change。

### Q4：用户说某条 Incident 看不到，你如何排查？

**参考答案：** 先确认用户、记录编号、目标表和期望操作。然后检查用户角色、组关系、列表过滤器、域隔离、表级/字段级 ACL，以及记录是否处于用户可见的条件范围。使用安全调试工具确认命中的 ACL，修复应针对具体权限需求设计，而不是直接授予管理员角色。

### Q5：ACL 的执行逻辑如何理解？

**参考答案：** ACL 按对象和操作进行评估，可能涉及表、记录和字段层级。需要同时满足所需角色、条件和脚本判断。排查时先确定访问动作，再检查具体资源和规则命中情况。字段级写权限不能简单用表级权限替代；如果用户需要写入字段，还要确保字段 ACL 允许。

### Q6：Client Script、UI Policy 和 Business Rule 怎么选？

**参考答案：** UI Policy 用于表单上的显示、只读和必填；Client Script 用于客户端动态行为和提交前校验；Business Rule 用于服务器端数据处理和业务规则。客户端规则不能保证 API、导入或后台脚本写入的数据安全，因此关键校验要放在服务端，可结合 Data Policy 或 Business Rule。

### Q7：Business Rule 导致系统变慢或重复更新，怎么处理？

**参考答案：** 先从事务日志和系统日志确定具体请求及耗时，确认 Business Rule 的表、时机、条件和执行顺序。检查是否在 `after` 或 `update` 中再次更新同一记录，是否存在递归触发、无条件查询、循环内查询或同步外部调用。修复时收紧条件、减少查询、改用异步处理或重新设计触发方式，并在测试环境验证数据一致性和性能。

### Q8：SLA 没有启动或计时不正确，怎么排查？

**参考答案：** 检查 SLA Definition 是否激活，以及任务是否满足 start condition。再检查 Task SLA 是否生成、stage 是否正确、暂停/停止条件是否被触发，并核对日历、工作时间、节假日及时区。若任务字段发生变化，还要判断 SLA 是否应取消、重启或继续。最后查看后台处理和相关日志，避免只从表单表象判断。

### Q9：一个目录申请提交后没有生成任务，怎么办？

**参考答案：** 确认 REQ、RITM 是否创建，变量是否保存，审批是否完成，以及 Catalog Item 是否处于激活状态。检查 Flow/Workflow 的触发条件、当前运行状态、审批节点、Catalog Task 创建动作、分派组和执行用户权限。查看流程运行记录和系统日志，修复后用新的测试申请验证完整链路，并考虑失败重试和补偿。

### Q10：如何设计一个可靠的外部系统集成？

**参考答案：** 先定义业务事件和数据契约，再确定同步或异步模式。为每个请求设计认证、超时、重试、幂等键、分页、速率限制、错误分类和监控。敏感字段要最小化传输并保护凭证。失败记录需要可追踪、可告警、可重放，不能因为接口重试就重复创建 Incident 或 Change。

### Q11：Import Set 和 Transform Map 的作用是什么？

**参考答案：** Import Set Table 作为外部数据的暂存区，Transform Map 负责把暂存数据映射到目标表。通过 Coalesce 识别已有记录，从而决定更新还是插入。排查导入失败时检查数据格式、字段映射、引用匹配、脚本、Coalesce、权限和导入日志，不能只看目标表里是否出现了数据。

### Q12：CMDB 数据重复或质量很差，如何治理？

**参考答案：** 先定义 CI 的业务用途、唯一识别规则、数据所有者和数据源优先级。使用 IRE、Discovery 和规范化流程减少重复；检查重复、孤立、过期和缺少关系的 CI，建立定期认证和质量指标。治理应与 Incident、Change 和服务模型结合，否则 CMDB 只会成为无人维护的资产列表。

### Q13：Discovery 运行失败，你如何排查？

**参考答案：** 先确定是计划未触发、MID Server 不可用、网络/凭证失败、探测器执行失败，还是识别与写入失败。检查 MID Server 状态、Capability、ECC Queue、凭证、DNS、防火墙、目标主机权限和 Discovery 日志。确认任务是否分配到了正确 MID Server，并在修复后验证 CI 更新、关系发现和重复率。

### Q14：如何发布一个更新集到生产？

**参考答案：** 在开发环境完成配置、代码审查和单元测试，检查更新集内容和依赖；在测试环境预览并解决冲突，完成回归和业务验收；生产发布前准备窗口、审批、验证清单和回滚步骤。发布后检查核心流程、通知、SLA、集成和错误日志。敏感配置、数据修复和外部凭证要使用适合的独立流程管理。

### Q15：更新集 Preview 出现冲突怎么办？

**参考答案：** 先查看冲突对象、当前生产版本和更新集版本，判断哪一方是正确的业务实现。不要无差别接受全部更新。保留生产必要改动，把需要发布的内容合并到目标配置中；在非生产环境验证依赖、脚本顺序和回归结果，记录决策后再提交更新集。

### Q16：ServiceNow 升级前后你会做什么？

**参考答案：** 升级前盘点插件、集成、自定义脚本、关键流程和弃用项，在子生产实例演练并准备回归清单。升级后验证登录、事件、变更、目录、知识、审批、通知、SLA、报表、Discovery 和外部集成，检查系统错误和性能趋势。对失败项记录复现步骤、影响和临时方案，并按平台支持流程处理。

### Q17：怎样定位一次慢请求？

**参考答案：** 先确定请求边界和用户感知时间，再结合事务日志、系统日志、客户端脚本、服务端脚本和流程运行记录拆分耗时。判断是否为查询、锁等待、同步集成、脚本递归或浏览器渲染问题。用可测量的改动验证，例如减少查询结果、改为异步、收窄触发条件或分批处理，发布后持续观察指标。

### Q18：后台脚本修复生产数据时要注意什么？

**参考答案：** 先确认问题范围、业务批准、备份/恢复能力和回滚方案。脚本必须有明确过滤条件、执行上限、日志、预览模式和幂等设计，先在子生产环境使用脱敏或等价数据验证。执行时分批处理并记录成功、跳过和失败数量；执行后抽样核对、触发必要的下游同步，并保留审计记录。

### Q19：如何减少 ServiceNow 中的权限安全风险？

**参考答案：** 使用最小权限和职责分离，定期审查高权限角色、共享账号、闲置用户和 API 凭证。所有关键数据访问使用 ACL 控制，敏感信息不写入普通日志。对集成使用可轮换凭证、限制范围和审计；对生产变更使用审批、测试、发布窗口和事后复核。

### Q20：你如何定义 ServiceNow 运维成功？

**参考答案：** 不只看实例是否能登录，还要看关键业务流程的可用性和稳定性。可使用平台可用性、P1/P2 事件数量、平均恢复时间、SLA 达成率、变更成功率、重复事件率、集成成功率、CMDB 数据质量、自动化失败率和用户满意度等指标，并用趋势推动持续改进。

---

## 八、场景题答题模板

面对开放式故障题，可以按 **I-D-A-R-C** 结构回答：

1. **Identify**：确认现象、范围、影响、开始时间和优先级。
2. **Diagnose**：查看日志、事务、流程、权限、集成和数据，建立可验证假设。
3. **Act**：优先恢复服务，选择风险可控、可回滚的措施。
4. **Record**：记录时间线、证据、操作、结果和沟通内容。
5. **Correct**：完成根因分析、永久修复、测试、变更和复盘。

面试官通常会特别关注以下风险意识：

- 是否先评估影响，再执行操作。
- 是否区分 Incident 的恢复目标和 Problem 的根因目标。
- 是否知道生产操作需要审批、审计和回滚。
- 是否会使用日志和指标，而不是凭经验盲改。
- 是否考虑权限、敏感数据、重复执行和数据一致性。
- 是否能把临时恢复转化为后续的永久改进。

---

## 九、常见命令、页面与排查入口

不同 ServiceNow 版本和插件的菜单名称可能不同，面试时应说明“以实例版本和启用模块为准”。常见入口包括：

| 目标 | 常见入口或方法 |
|---|---|
| 查看系统错误 | System Logs / System Log > Errors |
| 查看事务耗时 | System Logs / Transactions |
| 查看用户与角色 | User Administration / Users、Groups、Roles |
| 排查 ACL | System Security / Access Control、Security Debugging |
| 查看服务器脚本 | System Definition / Business Rules、Script Includes |
| 查看客户端逻辑 | Client Scripts、UI Policies、Data Policies |
| 查看流程运行 | Flow Designer / Executions，或 Workflow Context |
| 查看导入 | System Import Sets / Import Set Runs、Transform History |
| 查看集成队列 | ECC Queue、REST/SOAP 日志、Integration logs |
| 查看更新集 | System Update Sets / Retrieved Update Sets |
| 查看字段定义 | Dictionary、表定义、Configure Dictionary |
| 查询记录 | 列表过滤器、条件构建器、encoded query |
| 调试脚本 | Background Scripts（仅限授权人员和受控环境） |

---

## 十、面试前速记清单

- 能解释 Instance、Application Scope、表继承、`sys_id`、Reference 和 Display Value。
- 能区分 Incident、Problem、Change、Request、RITM、Catalog Task。
- 能说明 Impact、Urgency、Priority、SLA 的关系。
- 能解释 ACL、角色、组，以及为什么不能用管理员角色绕过权限问题。
- 能区分 UI Policy、Client Script、Business Rule、Data Policy、Script Include 和 Flow Designer。
- 能讲清 Import Set、Transform Map、Coalesce、MID Server 和 ECC Queue。
- 能说明 CMDB、CI、关系、CSDM、Discovery、Service Mapping 和 IRE。
- 能描述更新集发布、冲突处理、测试、审批、回滚和升级回归。
- 能用日志、事务、流程记录和指标定位性能或集成故障。
- 任何生产场景都要主动提到影响评估、最小权限、可回滚、审计和复盘。

> 面试回答建议：先给结论，再给排查或实施步骤，最后补充风险控制和验证方式。ServiceNow 版本、插件和组织流程可能不同，遇到具体产品行为时应以目标实例文档和实际配置为准。
