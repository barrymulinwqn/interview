# Databricks 高频面试题与详细解答

> 本文面向 Databricks 数据工程师、数据平台工程师、Analytics Engineer、数据分析工程师和 Lakehouse 架构师面试，覆盖 Apache Spark、Delta Lake、流式处理、Unity Catalog、性能优化、工程化部署与生产故障排查。
>
> 面试时应以真实项目为基础回答：先给出结论和业务背景，再解释底层原理、技术取舍、验证方式和结果。不要只列出 Notebook、PySpark、Delta Lake 等工具名。

## 目录

1. [岗位核心能力与高分回答框架](#一岗位核心能力与高分回答框架)
2. [Lakehouse 与 Databricks 平台架构](#二lakehouse-与-databricks-平台架构)
3. [Apache Spark 高频题](#三apache-spark-高频题)
4. [Delta Lake 高频题](#四delta-lake-高频题)
5. [数据摄取、ETL 与流式处理](#五数据摄取etl-与流式处理)
6. [Unity Catalog、安全与数据治理](#六unity-catalog安全与数据治理)
7. [性能优化与成本控制](#七性能优化与成本控制)
8. [工程化、测试与部署](#八工程化测试与部署)
9. [综合场景与项目经历题](#九综合场景与项目经历题)
10. [面试前自检清单](#十面试前自检清单)

---

## 一、岗位核心能力与高分回答框架

### 1. 面试官真正考察什么

| 能力维度 | 面试官希望确认的能力 | 可准备的证明材料 |
| --- | --- | --- |
| 数据架构 | 能按数据域、时效、成本和治理目标设计 Lakehouse | 分层架构图、数据流、表粒度与 SLA |
| Spark | 能正确编写并调优分布式计算，而非把 Spark 当作单机 Pandas | Shuffle、倾斜、分区、执行计划优化案例 |
| Delta Lake | 能利用事务、版本、合并和表维护构建可靠数据集 | `MERGE`、时间旅行、Schema 演进、回滚案例 |
| 流式处理 | 能设计幂等、可恢复、有延迟约束的实时数据管道 | checkpoint、watermark、去重、迟到数据处理 |
| 治理与安全 | 能用 Unity Catalog 管理身份、权限、血缘和敏感数据 | 三层命名空间、外部位置、行列权限、审计 |
| 工程化 | 能将探索 Notebook 演进为可测试、可发布、可运维的产品 | Git、CI/CD、Jobs、配置管理、告警 |
| 业务沟通 | 能把数据技术选择和业务价值、数据质量联系起来 | 指标口径、对账、需求澄清和用户案例 |

### 2. 高分回答结构：结论 -> 原理 -> 取舍 -> 验证

以“为什么选择 Delta Lake”举例：

> 我选择 Delta Lake 作为 Lakehouse 的表格式，因为它在对象存储上的 Parquet 文件之上增加了事务日志，能提供 ACID 写入、Schema enforcement、版本历史和可扩展的 `MERGE`。对于订单 CDC 管道，这使得重试和并发写入更可控，也能通过版本回溯排查错误。它不是自动解决数据质量的问题：主键去重、迟到数据、分区策略和表维护仍需设计。上线前我会用源端与目标端行数、金额、主键重复率对账，模拟 Job 重试，并检查 Delta 历史和读写性能。

### 3. 项目题回答结构：STAR + 技术决策 + 量化结果

- **Situation（背景）**：业务领域、原有数据痛点、数据量、数据源和时效目标。
- **Task（任务）**：你负责的范围、上线时间、质量和安全约束。
- **Action（行动）**：分层设计、数据摄取、建模、性能调优、监控与协作。
- **Result（结果）**：用端到端延迟、刷新成功率、数据质量通过率、作业成本或人工工时说明价值。
- **Decision（取舍）**：为什么选择批处理或流处理、文件表或 Delta 表、预聚合或按需计算。
- **Learning（沉淀）**：留下哪些框架、测试、运行手册、监控指标或治理规范。

**注意：** 结果数字应可解释其统计范围和测量方式。面试中宁可给出可信的区间，也不要给出无法追溯的“提升 90%”。

---

## 二、Lakehouse 与 Databricks 平台架构

### Q1：什么是 Lakehouse？它解决了什么问题？

Lakehouse 是在低成本、可扩展的云对象存储上，同时提供数据湖的开放性和数据仓库的可靠性、性能与治理能力的架构。

传统数据湖常存放大量 Parquet、JSON、CSV 文件，但可能缺少事务、模式管理、可靠更新和统一权限；传统数据仓库则适合高性能 SQL 分析，但在多样化数据、机器学习和大规模低成本存储方面可能受限。Lakehouse 通过 Delta Lake 等表格式补上关键能力：

- ACID 事务与并发控制；
- Schema enforcement 和受控 Schema evolution；
- 批流一体的读写接口；
- 版本历史、时间旅行和可审计性；
- 面向 BI、数据工程和机器学习的一致数据层；
- 集中的元数据、权限与血缘治理。

Lakehouse 并不意味着取消所有数据仓库或数据集市。面向特定 BI 高并发、强语义或遗留系统集成的场景，仍可能需要 SQL Warehouse、语义层或专用服务。

### Q2：Databricks 的核心组件有哪些？

| 组件 | 主要职责 | 面试可说明的使用场景 |
| --- | --- | --- |
| Workspace | Notebook、Repos、Jobs、Dashboard 等工作空间 | 团队协作、权限分配与资产组织 |
| Compute | 运行 Spark、SQL 或机器学习任务的计算资源 | 交互开发、定时 ETL、SQL 查询 |
| Databricks SQL / SQL Warehouse | 面向 SQL 分析和 BI 的计算端点 | 报表、仪表板、数据分析师查询 |
| Delta Lake | 对象存储上的事务表格式 | 可靠的 Bronze/Silver/Gold 表 |
| Unity Catalog | 统一的元数据、安全、审计和血缘层 | 多工作区、多团队的数据治理 |
| Workflows / Jobs | 编排、调度、重试和监控任务 | 生产 ETL、依赖编排和 SLA 运行 |
| Auto Loader | 大规模增量发现和加载云对象存储文件 | 日志、文件投递和持续摄取 |
| MLflow | 实验、模型和模型生命周期管理 | 特征实验、模型注册与部署 |

回答时不要把这些组件说成彼此独立的产品。一个典型生产链路是：Auto Loader 摄取原始文件到 Delta Bronze 表，Spark 清洗并写入 Silver/Gold 表，Unity Catalog 管理表与权限，Jobs 编排运行，SQL Warehouse 为 BI 提供查询。

### Q3：什么是 control plane 和 data plane？

- **Control plane（控制平面）**：管理工作区、身份验证、Notebook 元数据、作业配置、集群控制等平台服务。
- **Data plane（数据平面）**：实际执行计算、读取或写入企业数据的资源与网络边界。

不同云和 Databricks 部署模式的具体网络拓扑不同，但面试中应体现安全意识：需要明确计算资源部署位置、对象存储位置、私有连接/网络隔离、密钥管理以及哪些数据会经过平台管理服务。不能笼统假设“数据永远不会离开某个网络”。

### Q4：Notebook 适合什么，不适合什么？

Notebook 是可交互执行代码、SQL、Markdown 和可视化结果的文档，适合：

- 数据探索、问题排查和原型开发；
- 讲解数据逻辑、展示样例和协作分析；
- 小型或中等规模的任务实现；
- 由 Workflows/Jobs 调用的参数化任务。

但生产代码不应只依赖一个巨大的 Notebook。以下情况更适合模块化代码项目：

- 多个管道共享业务规则或工具函数；
- 需要单元测试、静态检查、代码评审和版本化发布；
- 复杂依赖、领域模型或大团队长期维护；
- 需要精确的发布版本和回滚。

较成熟的方式是：Notebook 用于探索和演示，核心逻辑放到可测试的 Python/SQL 包或源码文件中；Notebook 或 Job 只负责传参和编排。

### Q5：Medallion Architecture 是什么？Bronze、Silver、Gold 分别做什么？

Medallion Architecture 将数据按质量和业务成熟度分层：

```text
原始系统 / 文件 / 事件流
             |
             v
Bronze：原始、可追溯、追加式数据
             |
             v
Silver：清洗、标准化、去重、可关联的数据
             |
             v
Gold：面向业务消费的聚合、指标或数据产品
```

| 层级 | 主要目标 | 常见处理 |
| --- | --- | --- |
| Bronze | 保留原始事实、支持重放与审计 | 添加摄取时间、来源文件、原始负载；少做业务变换 |
| Silver | 建立可信、可复用的标准数据 | 类型转换、去重、CDC 合并、主数据关联、质量校验 |
| Gold | 面向特定分析、BI 或应用 | 聚合、星型模型、业务 KPI、宽表或特征表 |

这是一种指导性模式，不是强制“三张表”规则。关键是每层责任清楚、可重跑、可追溯；不要把未经验证的原始数据直接供财务或管理层使用。

---

## 三、Apache Spark 高频题

### Q6：Spark 的 Driver、Executor、Job、Stage 和 Task 分别是什么？

```text
Spark application
     |
     +-- Driver：解析代码、构建执行计划、协调任务
     |
     +-- Executors：在工作节点上执行计算、缓存数据、返回结果
              |
              +-- Tasks：作用于一个数据分区的最小并行执行单元
```

- **Driver**：运行应用主程序，生成逻辑计划和物理计划，并向集群调度任务。
- **Executor**：运行在工作节点上的 JVM 进程，实际执行 Task 并缓存数据。
- **Job**：由一个 action（如 `count`、`write`、`collect`）触发的一次计算。
- **Stage**：Job 按 shuffle 边界划分出的并行计算阶段。
- **Task**：Stage 中针对单个分区执行的最小工作单位。

理解它有助于排障：Driver OOM 通常与 `collect()`、过大的结果或元数据有关；少量超慢 Task 常指向数据倾斜；大量小 Task 则可能导致调度开销过大。

### Q7：什么是 transformation 和 action？为什么 Spark 是惰性执行？

- **Transformation**：描述如何从一个数据集得到另一个数据集，如 `select`、`filter`、`join`、`groupBy`。它构建执行血缘，不会立刻计算。
- **Action**：要求返回结果或产生外部输出，如 `count`、`show`、`collect`、`write`，会触发 Job。

Spark 的惰性执行使优化器能够合并多个转换、裁剪无用列、下推过滤并构建更优的执行计划。它也意味着一段看似只是“定义 DataFrame”的代码并没有真正验证数据，必须用受控 action 或写入测试来验证。

```python
filtered = (
    spark.table("main.sales.orders")
    .filter("order_status = 'COMPLETED'")
    .select("order_id", "customer_id", "amount")
)

# 到这里通常尚未执行计算
row_count = filtered.count()  # action，触发 Job
```

### Q8：什么是 shuffle？为什么它通常很昂贵？

Shuffle 指数据为了重新分组、重分区或 Join 而在 Executor 之间重新分发的过程。`groupBy`、`distinct`、`orderBy`、某些 Join 和 `repartition` 经常会产生 shuffle。

它昂贵是因为涉及网络传输、序列化、磁盘落盘、排序和等待最慢分区。优化方向不是“绝不 shuffle”，而是避免不必要且过大的 shuffle：

1. 尽早过滤和选择必要列，减少进入 shuffle 的数据量。
2. 使用正确的 Join 策略，例如小维度广播。
3. 控制文件大小和分区数量，避免过多小分区或超大分区。
4. 检查数据倾斜，让少数 Task 不成为长尾。
5. 用执行计划和 Spark UI 证实优化，而不是仅凭代码直觉。

### Q9：什么是窄转换和宽转换？

- **窄转换（Narrow transformation）**：父分区的数据只会流向有限的子分区，例如 `filter`、`map`、`union`（视具体情况）。通常不需要跨节点重分发。
- **宽转换（Wide transformation）**：多个父分区的数据需要重新分配给子分区，例如 `groupBy`、`join`、`distinct`。通常形成 shuffle 边界和新的 Stage。

这一概念用于预测性能和容错代价。窄转换的 lineage 可较直接地恢复；宽转换需要处理 shuffle 文件和跨节点依赖。

### Q10：DataFrame、Dataset 和 RDD 如何选择？

| 抽象 | 特点 | 一般建议 |
| --- | --- | --- |
| DataFrame | 带 schema 的分布式表，支持 Catalyst 优化和高效执行 | PySpark/SQL 中的默认首选 |
| Dataset | Scala/Java 的强类型 API，与 DataFrame 能力结合 | 需要 JVM 编译期类型约束时使用 |
| RDD | 低级分布式集合，灵活但优化信息少 | 仅在 DataFrame 难以表达的底层需求中谨慎使用 |

在 PySpark 中，大多数 ETL 应使用 DataFrame API 或 SQL。将数据过早转换成 RDD 或在 Python UDF 中逐行处理，可能失去 Catalyst 优化、列裁剪和代码生成等优势。

### Q11：如何阅读 Spark 执行计划？

使用 `df.explain()` 或 `df.explain("formatted")` 查看计划，重点关注：

- 是否有预期的 `Filter` 和 `Project`，且尽早发生；
- Join 类型是 Broadcast Hash Join、Sort Merge Join 还是其他；
- 是否出现大量 `Exchange`，它通常意味着 shuffle；
- 是否读取了不需要的列或分区；
- Adaptive Query Execution（AQE）是否对 Join 或分区做了优化。

示例：

```python
sales_by_region = (
    sales.filter("order_date >= DATE '2026-01-01'")
    .join(regions, "region_id")
    .groupBy("region_name")
    .sum("sales_amount")
)

sales_by_region.explain("formatted")
```

执行计划必须结合 Spark UI、数据大小和实际运行时间判断。单纯看到 `Exchange` 并不意味着错误，因为聚合和大表 Join 本身需要全局重分布。

### Q12：什么是 Catalyst Optimizer？什么是 AQE？

**Catalyst Optimizer** 是 Spark SQL/DataFrame 的查询优化器。它会解析 SQL/DataFrame 逻辑、分析 schema、执行规则优化（如谓词下推、列裁剪、常量折叠）并生成物理执行计划。

**Adaptive Query Execution（AQE）** 会利用运行时统计信息调整计划，例如：

- 合并过小的 shuffle 分区；
- 在适当情况下将 Sort Merge Join 转为 Broadcast Hash Join；
- 识别和缓解部分倾斜 Join 分区；
- 优化空关系传播。

AQE 能改善很多常见负载，但不是忽略数据建模、Join 条件和分区策略的理由。面对严重倾斜或源端读取瓶颈，仍需针对根因处理。

### Q13：什么是数据倾斜？如何发现和处理？

数据倾斜指某些 key 或分区的数据远多于其他分区，导致少数 Task 远慢于其他 Task。例如 `country = 'US'` 占 90% 数据，或匿名用户 ID 被统一填为 `UNKNOWN`。

**发现方式：**

- Spark UI 中少数 Task 的输入数据、shuffle read/write 或耗时显著更大；
- Job 总时长由最后几个 Task 决定；
- 对 Join/聚合 key 做频率分析；
- 检查空值、默认值和异常业务值。

**处理方式：**

1. 修复源端脏数据或缺失键，避免把大量记录归入单一默认 key。
2. 对小维表使用 broadcast join，减少大表两侧重分布。
3. 启用并验证 AQE 倾斜处理。
4. 对极端热点 key 采用 salting，将热点 key 拆散并在另一侧复制相应的小数据。
5. 将热点业务拆成单独路径，分别处理再合并。

Salting 会增加数据和代码复杂度，只应在确认倾斜是主要瓶颈后使用。

### Q14：`repartition` 和 `coalesce` 有什么区别？

- `repartition(n, ...)` 可以增加或减少分区，通常会产生全量 shuffle，分布更均匀；可按指定列重分区。
- `coalesce(n)` 通常用于减少分区，尽量避免 shuffle，但可能产生不均衡的分区。

常见做法是在写出前根据目标文件大小和并发度减少小文件；若需要按某列重新均匀分布以支持后续 Join 或写入，则使用 `repartition`。不要对每一个 DataFrame 机械调用它们，应先检查当前分区数和下游操作。

### Q15：为什么应避免在 Driver 上使用 `collect()`？

`collect()` 会将所有分区的数据发送到 Driver。若数据集较大，容易造成 Driver 内存溢出、网络压力和应用失败。

替代方式：

- 调试时用 `limit()`、`show()` 或只选择必要列；
- 汇总后再收集少量结果；
- 将结果写入 Delta 表或对象存储；
- 对少量配置值使用 `first()` 等受控操作，并验证数据量上限。

面试中应说明：`collect()` 并非绝对禁止，它适用于确认数据量很小且有明确上限的场景，例如读取几十条配置记录。

---

## 四、Delta Lake 高频题

### Q16：Delta Lake 的核心组成和能力是什么？

Delta Lake 在 Parquet 数据文件之上维护事务日志（通常位于表目录的 `_delta_log`），记录新增、删除、元数据和协议等变更。读者基于日志获得一致的表快照，而不是随意扫描目录里的文件。

核心能力包括：

- ACID 事务与乐观并发控制；
- Schema enforcement 和可控 Schema evolution；
- `MERGE`、`UPDATE`、`DELETE` 等可靠的数据修改；
- 表版本、`DESCRIBE HISTORY` 和时间旅行；
- 批流一体读写；
- 变更数据捕获等增量处理能力（需按平台和表配置确认）。

它不自动保证主键唯一、业务规则正确或来源数据可信。数据质量仍需在管道中显式校验。

### Q17：Delta Lake 如何实现 ACID 和并发控制？

Delta Lake 采用乐观并发控制（Optimistic Concurrency Control）：

1. 读取操作读取当前一致的表快照。
2. 写入操作准备要新增或移除的文件，并尝试提交一个新的事务日志版本。
3. 提交时检查自开始以来是否有冲突提交。
4. 无冲突则提交新版本；有冲突则失败或按操作类型重试。

这允许读写并发，同时避免读取到半完成的文件集合。并发 `MERGE`、分区覆盖和 Schema 变更仍可能冲突，因此生产任务应设计重试策略、清晰的写入边界和幂等逻辑。

### Q18：什么是 Schema enforcement 与 Schema evolution？

- **Schema enforcement**：写入时检查数据类型和字段是否符合目标表 schema，阻止意外的错误数据写入。
- **Schema evolution**：在明确允许时扩展或调整 schema，例如新增可空列。

示例：

```python
(
    incoming_df.write
    .format("delta")
    .mode("append")
    .option("mergeSchema", "true")
    .saveAsTable("main.bronze.customer_events")
)
```

`mergeSchema` 不是默认配置。生产环境必须谨慎使用：它可以处理预期的新字段，但也可能把源系统拼写错误的列永久写进表。对于 Silver/Gold 表，通常应采用契约校验和受控迁移，而不是无条件自动演进。

### Q19：如何使用 `MERGE` 实现 CDC Upsert？

`MERGE` 根据业务键匹配来源和目标，可对匹配记录更新、对新记录插入，也可处理删除标记。

```python
from delta.tables import DeltaTable

target = DeltaTable.forName(spark, "main.silver.customer")

(
    target.alias("t")
    .merge(
        source_df.alias("s"),
        "t.customer_id = s.customer_id"
    )
    .whenMatchedUpdate(
        condition="s.updated_at >= t.updated_at",
        set={
            "customer_name": "s.customer_name",
            "status": "s.status",
            "updated_at": "s.updated_at",
            "ingested_at": "current_timestamp()",
        },
    )
    .whenNotMatchedInsert(
        values={
            "customer_id": "s.customer_id",
            "customer_name": "s.customer_name",
            "status": "s.status",
            "updated_at": "s.updated_at",
            "ingested_at": "current_timestamp()",
        }
    )
    .execute()
)
```

**关键设计点：**

1. 来源数据在同一批内必须按业务键去重，否则一条目标记录匹配多条来源记录会导致失败或不确定结果。
2. 需要定义“更新较新”的比较字段和时区，防止迟到旧事件覆盖新状态。
3. 删除事件应选择物理删除、软删除或维护有效期，取决于审计、下游和合规要求。
4. 重跑同一批数据应得到相同最终结果，即实现幂等。

### Q20：什么是时间旅行（Time Travel）？有什么用途？

Delta 表保存多个版本。可以按版本号或时间戳读取历史快照：

```sql
SELECT *
FROM main.silver.customer VERSION AS OF 42;
```

```python
previous = (
    spark.read.format("delta")
    .option("versionAsOf", 42)
    .table("main.silver.customer")
)
```

用途包括：

- 排查某次写入后数据为何变化；
- 对账历史版本；
- 在错误写入后恢复或重建；
- 复现实验和训练数据快照。

时间旅行受历史文件和日志保留策略限制。执行 `VACUUM` 清理旧文件后，过早版本可能无法再读取，因此保留期要与审计、回滚和流式消费者需求协调。

### Q21：`OPTIMIZE`、`ZORDER`、聚簇和 `VACUUM` 分别做什么？

| 操作 | 目的 | 注意事项 |
| --- | --- | --- |
| `OPTIMIZE` | 合并过多小文件，改善读性能与元数据开销 | 需要评估运行窗口和写入并发 |
| `ZORDER BY` | 按常用过滤列对文件内数据进行多维局部聚集，提高数据跳过效果 | 只适合经过验证的高价值过滤列 |
| 聚簇（如平台支持的 liquid clustering） | 用声明式方式维护数据布局，减少手动分区/排序维护 | 需理解当前表类型、写入模式与成本 |
| `VACUUM` | 删除不再被当前表引用且超过保留期的旧文件 | 会影响时间旅行、回滚和落后的流式读取 |

不要把这些操作当作默认的“万能清理脚本”。应基于文件数量、查询模式、表更新频率、版本保留要求和作业窗口制定策略。

### Q22：分区（Partitioning）应该如何设计？

分区是目录级数据组织，会影响文件扫描和写入并发。适合分区的列通常：

- 经常用于等值或范围过滤；
- 基数较低到中等；
- 每个分区有足够数据量；
- 写入分布相对均衡。

日期常是合理候选，例如按 `event_date` 进行日或月分区。但对小表或高基数列（用户 ID、订单 ID）过度分区会制造海量小文件和元数据压力。

选择分区前应分析查询模式和每日数据量；对于许多现代 Delta 表，适当的文件布局或聚簇策略比大量静态分区更合适。

### Q23：什么是小文件问题？如何解决？

小文件问题指一个表包含大量很小的数据文件。每个文件都带来列举、打开、调度和元数据成本，使读取和写入变慢。

常见根因：

- 高频微批写入，每批数据很少；
- 过度分区；
- 写入时分区数过多；
- 多个并发作业频繁向同一表写入；
- 输出未做合理合并。

解决方法：

1. 调整微批频率或单批数据量。
2. 修正分区和写入并发度。
3. 对适合的表运行 `OPTIMIZE` 或使用平台的自动优化能力。
4. 评估目标文件大小，而不是追求单一固定数字。
5. 将小批事件先缓冲再合并，前提是不违反延迟 SLA。

---

## 五、数据摄取、ETL 与流式处理

### Q24：Auto Loader 是什么？与普通文件读取相比有什么价值？

Auto Loader 用于持续、可扩展地从云对象存储发现并摄取新文件。相较于每次递归列举整个目录，它可利用文件通知或增量发现机制，减少大目录扫描成本，并将已处理文件状态保存在 schema/checkpoint 位置。

典型模式：

```python
raw_events = (
    spark.readStream.format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/Volumes/main/ops/checkpoints/events-schema")
    .load("/Volumes/main/raw/events")
)

(
    raw_events.writeStream
    .option("checkpointLocation", "/Volumes/main/ops/checkpoints/events-bronze")
    .toTable("main.bronze.events")
)
```

实际路径应遵循组织的 Unity Catalog 外部位置或 Volume 管理规范。`schemaLocation` 与 `checkpointLocation` 是不同职责的持久状态：前者保存推断和演进相关 schema，后者保存流式处理进度。

### Q25：Structured Streaming 的 exactly-once 是什么意思？

Structured Streaming 通常通过 checkpoint、可重放的输入和支持事务的 sink 实现端到端“恰好一次的结果语义”或接近该语义的处理保证。它不意味着每条事件在物理层面只被读取一次：故障恢复时，微批可能被重新执行。

因此实际设计应追求**幂等结果**：

- 使用稳定的事件 ID 或业务键去重；
- 将 checkpoint 放在可靠、独占且持久的位置；
- 写入 Delta 表等支持事务的目标；
- 对外部副作用（发邮件、调用 API、写非事务数据库）实现幂等键、去重记录或补偿机制。

面试中如只说“开启 checkpoint 就能 exactly once”，答案是不完整的。

### Q26：Watermark 和流式去重如何工作？

事件可能延迟到达。若无限期保存所有已见事件 ID 进行去重，状态会无限增长。Watermark 为事件时间定义可接受的迟到边界，使系统在边界后清理旧状态。

```python
deduplicated = (
    events
    .withWatermark("event_time", "2 hours")
    .dropDuplicates(["event_id"])
)
```

含义不是“任何晚于 2 小时的事件都必然被处理”，而是系统可在 watermark 推进后丢弃旧状态；过晚事件是否保留取决于具体算子和数据到达情况。

设计时必须与业务确认：

- 可接受的最大延迟是多少；
- 是按事件时间还是摄取时间衡量；
- 超过阈值的数据是丢弃、进入隔离表，还是触发补数；
- 窗口聚合、去重和下游指标如何处理迟到修正。

### Q27：批处理与流处理如何选择？

| 判断维度 | 更适合批处理 | 更适合流处理 |
| --- | --- | --- |
| 业务时效 | 小时/天级即可 | 分钟/秒级有明确决策价值 |
| 数据来源 | 每日导出文件、周期性全量表 | 事件、日志、消息队列、CDC |
| 复杂度承受度 | 希望简单、便于对账与回填 | 有能力维护 checkpoint、延迟和状态 |
| 消费方式 | 日报、月报、离线训练 | 监控告警、实时风控、实时推荐 |

不要为了“技术先进”把日终财务报表改造成流式。先用业务 SLA 驱动技术选择；很多场景下高频微批或增量批处理已经足够。

### Q28：如何保证 ETL 管道可重跑和幂等？

可重跑是生产数据工程的基本要求。常见方法：

1. 保留 Bronze 原始数据与来源元数据，使下游表能从可信输入重建。
2. 用稳定业务键和事件版本实现 `MERGE`，而不是单纯 append。
3. 对每批次记录批次 ID、输入范围、读取水位、处理时间和输出版本。
4. 避免依赖当前时间、随机数或非确定性顺序；必须使用时将它们作为批次参数记录。
5. 对分区覆盖使用显式条件，避免误覆盖整张表。
6. 重跑时验证目标行数、金额、重复率和版本变化是否符合预期。

### Q29：如何设计数据质量检查？

数据质量不应只在报表出错后人工发现。可在每层定义可量化规则：

| 规则类别 | 示例 | 处理策略 |
| --- | --- | --- |
| 完整性 | `order_id`、事件时间不能为空 | 隔离无效记录，记录原因 |
| 唯一性 | 订单明细键在 Silver 层唯一 | 去重或终止任务并告警 |
| 有效性 | 金额非负、币种在允许集合内 | 隔离并通知数据源负责人 |
| 参照完整性 | 事实中的产品键能在产品维度找到 | 标记未知维度并跟踪 |
| 新鲜度 | 数据延迟不超过 SLA | 告警、阻止 Gold 发布或降级 |
| 对账 | 日销售额与源系统差异在阈值内 | 终止发布并启动排查 |

关键原则是：不要用广泛的异常捕获把坏数据替换为 `null` 后继续发布。应保留隔离数据、规则结果、批次信息和清晰告警。

### Q30：如何处理 CDC 中的乱序、重复和删除？

CDC 事件通常需要事件 ID、业务键、操作类型、源系统版本/LSN、事件时间和摄取时间。处理策略：

- **重复**：按事件 ID 或业务键 + 版本去重。
- **乱序/迟到**：只有来源版本更高或更新时间更晚时才覆盖目标；记录时间区域和时钟差异。
- **删除**：明确是物理删除、软删除，还是 SCD Type 2 结束有效期。
- **全量快照与增量并存**：定义快照基线、CDC 起始位置和切换过程，避免遗漏或重复。
- **回填**：以受控批次重新处理历史范围，避免与实时流并发写入同一逻辑范围产生冲突。

---

## 六、Unity Catalog、安全与数据治理

### Q31：Unity Catalog 的三层命名空间是什么？

Unity Catalog 通常使用：

```text
catalog.schema.object
```

例如：

```sql
SELECT * FROM main.sales.orders;
```

- **Catalog**：可对应业务域、环境或治理边界。
- **Schema**：Catalog 下的逻辑分组，例如 `bronze`、`silver`、`finance`。
- **Object**：表、视图、函数、Volume、模型等具体对象。

命名设计要服务于权限、所有权、环境隔离和可发现性。不要将所有对象堆入单一默认 schema，也不要仅为了目录美观而创建过深层级。

### Q32：Managed table、External table、External location 和 Volume 有什么区别？

| 概念 | 数据位置/用途 | 典型场景 |
| --- | --- | --- |
| Managed table | 平台管理表的数据生命周期和存储位置 | 新建的受治理 Delta 表，优先选择 |
| External table | 数据位于用户管理的外部云存储路径 | 迁移已有数据、共享受控外部数据 |
| External location | 将云存储 URL 与存储凭据绑定的受治理对象 | 安全访问外部路径，避免在代码中放密钥 |
| Volume | Unity Catalog 管理的非表文件访问对象 | 配置文件、模型文件、原始附件等文件型资产 |

重要原则是：不要让用户通过任意裸路径绕过权限治理。无论是表数据还是文件数据，都应经过组织批准的存储凭据和访问边界。

### Q33：如何设计最小权限访问？

建议以用户组而非个人账户授予权限，并将职责区分为：

- 数据生产者：对其负责的 Schema 有创建/修改表的必要权限；
- 数据消费者：对认证的 Gold 表或视图有 `SELECT` 权限；
- 平台运维：管理计算、作业和有限的基础设施对象；
- 数据管理员：管理 Catalog、权限策略和审计；
- 敏感数据管理员：审批或维护受限字段、动态视图和脱敏策略。

生产环境中，个人开发账号不应成为作业访问数据的长期身份。使用受控服务主体或组织认可的作业身份，并定期审查权限、所有权和离职人员访问。

### Q34：如何保护 PII 和敏感数据？

先识别并分类 PII、财务、健康、认证和受监管数据，再组合使用：

1. **最小化采集与暴露**：不为“以后可能有用”而复制敏感列。
2. **分层访问**：Bronze 原始敏感数据只给受控工程角色；Gold 只暴露必要的聚合或脱敏字段。
3. **动态视图/行列过滤**：根据用户组限制行或掩码敏感列。
4. **令牌化/哈希**：按用途和安全要求处理标识符；注意普通无盐哈希可能被字典攻击还原。
5. **密钥与凭据管理**：使用受控密钥管理和 secrets，绝不在 Notebook、Git 或表字段中硬编码。
6. **审计与保留策略**：记录访问和变更，满足数据保留、删除和合规要求。

掩码不是权限替代品。拥有底层原始表权限的用户仍可能绕过面向消费者的视图，因此源表权限必须同时收紧。

### Q35：什么是数据血缘？为什么重要？

数据血缘描述一个表、列、报表或模型从哪些上游数据产生、经过哪些转换、又被哪些下游消费。它用于：

- 评估修改字段、作业或表 schema 的影响范围；
- 在数据异常时快速定位源头；
- 满足审计和合规需求；
- 提高跨团队沟通和数据可信度。

自动血缘通常不能捕获所有 Notebook 中的动态 SQL、外部系统逻辑或人工文件处理。因此除使用平台血缘外，关键数据产品还应维护负责人、SLA、数据契约和变更流程。

---

## 七、性能优化与成本控制

### Q36：如何系统排查一个慢的 Databricks Job？

不要一开始就“加大集群”。应按以下顺序定位：

1. **确认症状**：是启动慢、读取慢、某个 Stage 慢、写入慢还是偶发失败？记录 Job run、输入规模和基线。
2. **查看 Spark UI**：检查 Stage、Task 耗时分布、shuffle、spill、GC、输入/输出和失败重试。
3. **看执行计划**：确认过滤、列裁剪、Join 策略、Exchange 和分区是否合理。
4. **检查数据布局**：文件数量、文件大小、分区、热点 key、表统计信息和历史维护。
5. **检查计算资源**：节点规格、autoscaling、并发作业、Driver 内存和网络/源端限制。
6. **单项验证优化**：每次改动后对比相同输入的耗时、成本和结果对账。

| 症状 | 常见根因 | 优先处理方式 |
| --- | --- | --- |
| 少数 Task 很慢 | 数据倾斜 | 分析热点 key、广播小表、AQE、必要时 salting |
| 大量 Task 很短 | 小文件/过度分区 | 合并文件、调整分区和写入并发 |
| Join 很慢 | 大表 shuffle 或错误 Join 策略 | 过滤列裁剪、广播小表、处理倾斜 |
| Driver OOM | `collect`、过大元数据或单分区结果 | 消除 Driver 聚合、限制结果、重新分区 |
| 频繁磁盘 spill | 分区过大或内存不足 | 调整分区、减少 shuffle、合理调整计算规格 |

### Q37：Broadcast Join 何时适用？有哪些风险？

当一侧表足够小、能够安全放入各 Executor 内存时，广播小表可避免大表两侧都按 Join key shuffle。

```python
from pyspark.sql.functions import broadcast

enriched_orders = orders.join(
    broadcast(product_dimension),
    "product_id",
    "left"
)
```

适用条件：

- 小表经过过滤和列裁剪后确实很小；
- 该表不会在每个 Executor 上导致内存压力；
- Join 条件正确，且不会因一对多关系造成意外行膨胀。

风险包括将“看起来小”的表广播到大量 Executor 后总内存消耗很大、数据增长后作业突然失败，以及强制广播掩盖更根本的模型或过滤问题。优先让优化器基于统计信息选择，必要时才明确提示。

### Q38：缓存（cache/persist）何时有用？何时有害？

缓存适合一个经过高成本计算的 DataFrame 在同一应用或作业内被多个 action 重复使用，且缓存的数据能放入可用内存/磁盘空间。例如同一清洗结果被多个下游聚合共享。

缓存可能有害的情况：

- DataFrame 只被使用一次，缓存只增加一次物化成本；
- 缓存数据太大，导致驱逐、GC 或其他任务内存不足；
- 代码未 `unpersist()`，长期交互集群积累无用缓存；
- 底层数据更新后错误使用陈旧缓存。

是否缓存应通过 Spark UI 中的复用次数、计算成本、内存占用和端到端时长验证，而不是习惯性添加 `cache()`。

### Q39：如何控制 Databricks 成本？

成本优化的目标是“单位业务价值的成本最低”，不是把集群配置到最小。常见手段：

1. 为交互、生产 ETL、SQL 和开发环境使用不同的计算策略。
2. 设置合理自动终止，避免空闲交互计算持续计费。
3. 使用 Job Compute 或受控的无服务器/按需计算来隔离生产任务与个人开发。
4. 通过查询、文件布局和数据模型优化减少不必要扫描，而不是只缩小节点。
5. 根据真实数据量和 SLA 选择节点规格与 autoscaling 上下限，避免长时间过度配置。
6. 监控按作业、团队、表和环境的成本，识别空跑、重复管道和异常增长。
7. 在不牺牲业务可用性的前提下，将低优先级历史回填安排在合适的运行窗口。

### Q40：为什么“更多分区”不一定更快？

分区数太少会导致并行度不足、单个 Task 太大；分区数太多则产生更多调度、序列化、网络和小文件开销。最优点取决于数据大小、操作类型、节点数、文件布局和下游写入方式。

正确方法是观察：

- 每个 Task 的输入数据和执行时长；
- CPU 利用率与 Task 排队情况；
- shuffle 分区大小；
- 目标文件数与文件大小；
- 端到端运行时间和成本。

不要把 `spark.sql.shuffle.partitions` 或 `repartition(200)` 当成所有任务的固定答案。

### Q41：如何优化 Delta 表的读取性能？

- 只读取必要列，让列裁剪生效。
- 在查询早期使用可下推的过滤条件。
- 根据实际过滤模式选择分区、数据布局或聚簇，利用数据跳过。
- 控制小文件，按需执行 `OPTIMIZE`。
- 为高频 Join/过滤列维护合适的 schema、统计信息和数据质量。
- 避免在过滤列上套使优化器难以下推的复杂表达式。
- 对重复的高成本聚合评估物化 Gold 表或增量聚合。

任何优化都应在代表性的查询和数据规模上验证，并且检查结果是否仍正确。

---

## 八、工程化、测试与部署

### Q42：如何将 Notebook 变成可生产运行的管道？

一个可生产运行的管道至少应具备：

1. **参数化**：环境、catalog/schema、输入范围、日期和运行模式由配置传入，而非硬编码。
2. **模块化**：将可复用读取、转换、质量检查和写入逻辑放入源码模块。
3. **明确输入输出**：定义表契约、主键、schema、分区/聚簇策略、SLA 和数据所有者。
4. **可观测性**：记录批次 ID、输入输出行数、延迟、质量结果、表版本和异常。
5. **故障处理**：只对可预期的暂时错误进行有限重试；对数据错误显式失败或隔离。
6. **测试**：单元测试转换逻辑、集成测试表读写、端到端对账。
7. **部署与回滚**：通过 Git、CI/CD 和受控环境提升发布，不依赖手工复制 Notebook。

### Q43：如何测试 PySpark 数据转换？

建议将业务转换写成接受和返回 DataFrame 的纯函数，使其可在小样本上测试：

```python
from pyspark.sql import DataFrame
from pyspark.sql import functions as F


def calculate_net_sales(orders: DataFrame) -> DataFrame:
    return (
        orders
        .filter(F.col("status") == "COMPLETED")
        .withColumn("net_sales", F.col("gross_sales") - F.col("discount_amount"))
    )
```

测试应覆盖：

- 正常行、空值、边界金额和无效状态；
- schema、列类型和行数预期；
- 去重、Join、CDC 乱序和迟到数据；
- 输入/输出对账和幂等重跑；
- 对关键 Delta 写入的集成测试或临时 schema 测试。

不要只测试“代码没有报错”；数据工程测试应验证数据值、记录数量、质量规则和副作用。

### Q44：如何管理环境配置和密钥？

- 将 dev/test/prod 的表名、存储位置、计算策略、阈值等放到受控配置中。
- 用变量、参数或部署配置注入环境差异；禁止在代码中散落生产路径。
- 密钥、Token、数据库密码和私钥放入组织批准的 secret 管理系统或密钥管理服务。
- 作业使用受控服务主体/身份，遵循最小权限。
- Git 仓库、Notebook 输出、日志和错误消息都不应包含敏感值。

如果密钥已经出现在代码或版本历史中，应立即撤销/轮换，并按组织安全流程处理；“删除当前行”通常不能消除已泄露风险。

### Q45：如何设计 Databricks Workflows/Jobs？

一个生产 Job 通常应明确：

- 任务 DAG 和依赖顺序；
- 每个 Task 的计算类型、参数、超时、重试次数和通知；
- 运行身份及输入/输出权限；
- 并发策略，避免多个运行同时写同一业务范围；
- 失败告警、运行日志、数据质量门禁和重跑入口；
- 关键表的 SLA 与下游依赖。

例如先运行 Bronze 摄取，再运行 Silver 清洗和质量检查；质量失败时不触发 Gold 发布和 BI 刷新。不要在失败后仍让下游报表使用部分更新的数据。

### Q46：什么是 CI/CD 在 Databricks 中的关键点？

CI/CD 的目标是让代码和基础设施变更可审查、可重复、可追溯地从开发进入生产：

1. 代码、SQL、配置和测试进入 Git。
2. Pull Request 触发格式化、静态检查和单元测试。
3. 在受控的测试环境部署并执行集成测试和数据质量验证。
4. 通过批准流程部署生产，并记录发布版本。
5. 若关键验证失败，能够停止发布或回滚到上一稳定版本。

数据表本身的 schema 和数据迁移需要单独计划。代码回滚不一定会自动恢复已写入的错误数据，因此关键写操作应可审计、可重建，并结合 Delta 版本策略处理。

### Q47：如何监控一个数据管道是否健康？

至少覆盖四类指标：

| 类别 | 示例 |
| --- | --- |
| 可用性 | Job 成功率、失败率、重试次数、连续失败次数 |
| 时效性 | 开始/结束时间、端到端延迟、数据新鲜度、SLA 违约 |
| 数据质量 | 输入输出行数、重复率、空值率、隔离记录数、对账差异 |
| 性能与成本 | 运行时长、节点使用、shuffle、文件数、计算成本 |

告警必须可执行：告警中应包含任务、环境、批次、失败步骤、影响范围、运行链接和初步处置建议。只有“任务失败”而没有上下文的告警会延长恢复时间。

---

## 九、综合场景与项目经历题

### Q48：请设计一个电商订单 Lakehouse 数据平台。

**需求假设：** 订单、支付和物流来自业务数据库 CDC；商品和客户来自每日主数据；运营需要近小时销售看板，财务需要次日可对账收入；区域团队只能访问所属区域数据。

**推荐回答：**

1. **澄清口径与 SLA**：确认订单、支付、发货、退款状态机；近小时看板和日终财务的不同口径、时区和截止规则。
2. **摄取层**：CDC/事件写入 Bronze Delta 表，保留来源位置、事件 ID、操作类型、源版本、事件时间和摄取时间；主数据文件/快照也保留批次元数据。
3. **Silver 层**：按业务键和来源版本去重，处理乱序和删除；标准化货币、时间、产品/客户键；输出质量隔离表与规则结果。
4. **Gold 层**：建立订单明细、支付、退款等事实以及日期、产品、客户、区域维度；为实时看板提供小时级聚合，为财务提供锁定口径的日结表。
5. **治理**：使用 Unity Catalog 的 Catalog/Schema 划分环境和数据域，用用户组/服务身份授权；通过受控视图或行过滤实现区域访问。
6. **编排与监控**：Workflows 按依赖运行摄取、清洗、对账和聚合；质量或对账失败时阻断下游发布并通知责任人。
7. **性能**：针对查询模式维护文件布局、常用聚合和 SQL Warehouse；用 Spark UI 处理热点 Join 和小文件，而不是默认扩大集群。

### Q49：源系统重复发送事件，且有些事件晚两天到达，如何处理？

先确认业务对历史修正的要求，再设计：

1. Bronze 表保留原始事件与稳定事件 ID，不因重复事件丢失审计能力。
2. Silver 层按照 `event_id` 或业务键 + 来源版本去重，并记录摄取时间。
3. 使用事件时间和 watermark 管理有限状态；两天迟到是否可接受必须由业务 SLA 决定。
4. 对超过常规延迟窗口的事件写入隔离/迟到表，触发定期回填或增量修正。
5. 下游 Gold 聚合须支持受影响日期的重算，而不是只能 append。
6. 用结果幂等的 `MERGE` 或可重复分区重建实现安全重跑。

### Q50：一个 Job 从 20 分钟变成 2 小时，你的排障步骤是什么？

1. 比较慢运行与正常运行的输入量、文件数、schema、数据分布、代码版本和计算配置。
2. 在 Spark UI 查看哪一个 Stage 增长；判断是读取、shuffle、Join、聚合、写入还是资源等待。
3. 检查 Task 耗时分布：少数长尾提示倾斜，所有 Task 均慢可能是源端、网络、集群或数据量变化。
4. 检查执行计划是否因统计信息或数据规模变化选择了不同 Join 策略。
5. 检查目标 Delta 表小文件、并发写入、OPTIMIZE/VACUUM 运行和存储限流。
6. 在隔离环境使用相同输入验证假设；每次只做一项修复并对账结果。
7. 修复后记录根因和预防措施，例如热点 key 监控、文件数量阈值、输入量告警或容量规划。

### Q51：如何将一个全量每日 ETL 改造成增量管道？

1. 先确认源系统能否提供可靠的更新时间、递增 ID、CDC log 或变更标记；没有可靠变更标记时不要假设增量正确。
2. 定义 watermark：上次成功处理到哪里、是否采用重叠窗口应对迟到更新。
3. 在 Bronze 保留原始增量批次；在 Silver 用业务键 + 版本进行幂等合并。
4. 对删除、回写、迟到和 schema 变化设计专门策略。
5. 初次全量加载后运行一段时间的全量/增量并行对账，确认行数、金额、哈希或关键指标一致。
6. 监控增量比例、延迟和漏数风险，并保留可回填的历史路径。

### Q52：如何回答“你做过什么 Databricks 优化”？

不要只回答“用了缓存和调大集群”。可以按以下模板组织真实经历：

> 一个订单明细管道每天处理约 X 数据量，运行时间从基线 Y 分钟开始增长。我先用 Spark UI 发现主要耗时在产品维度 Join 后的少数几个 Task，并通过 key 分布确认默认产品编码造成倾斜。同时发现表中有大量小文件，读取阶段也有明显开销。处理上，我先在 Join 前过滤无效记录并裁剪字段；对足够小的有效产品维度使用广播；修复默认编码数据并对异常记录隔离；再按查询和写入模式维护 Delta 文件布局。优化后，在相同输入规模和同一计算配置下，运行时间降至 Z 分钟，且通过订单数、金额和主键重复率对账。随后增加了数据倾斜和文件数量监控，防止问题复发。

用自己的真实 X、Y、Z 和业务限制替换模板中的占位符。

### Q53：如何处理“报表数字与源系统不一致”？

不要直接改 Gold 表或 BI 公式。应遵循从消费层到源头的可复现对账：

1. 明确比较的时间、时区、状态、币种、组织范围和刷新版本。
2. 在极小范围中选择订单或客户样本，逐笔核对。
3. 对比源系统、Bronze、Silver、Gold 和 BI 语义层的行数、金额和键集合。
4. 判断问题是源端迟到/重复、转换规则、维度关联、聚合口径还是报表过滤。
5. 记录根因、影响范围和修复方案；如已影响外部报表，明确数据更正和沟通方式。
6. 将对账规则自动化为质量门禁或持续监控，防止重复发生。

### Q54：如何回答“为什么不用单一大表，而要分层和建模”？

单一大表在原型阶段交付很快，但长期会造成：

- 原始数据、清洗规则和消费口径混杂，难以追溯；
- 重复维度属性导致存储膨胀和更新不一致；
- 多个团队反复复制同一逻辑；
- 质量问题难定位，回填和重跑风险高；
- 权限难以按数据层和消费需求管理。

分层并非为了增加表数量，而是将原始保留、标准化复用和业务消费分开，使每层有清晰责任、质量标准和所有者。Gold 层仍可为特定高性能分析提供经过治理的宽表或聚合表。

---

## 十、面试前自检清单

### 技术准备

- [ ] 能画出并解释 Lakehouse、Medallion Architecture 和端到端数据流。
- [ ] 能解释 Driver、Executor、Job、Stage、Task、shuffle、窄/宽转换。
- [ ] 能看懂基本执行计划，并说明 Broadcast Join、AQE、数据倾斜和小文件问题。
- [ ] 能解释 Delta Lake 的事务日志、ACID、`MERGE`、Schema enforcement/evolution、时间旅行和表维护。
- [ ] 能说明 Auto Loader、Structured Streaming、checkpoint、watermark、迟到数据和幂等。
- [ ] 能设计 Unity Catalog 的 Catalog/Schema、外部位置、最小权限和敏感数据保护。
- [ ] 能讲清如何在 Databricks 中进行测试、Job 编排、CI/CD、监控和生产发布。

### 项目准备

- [ ] 准备 2 至 3 个真实项目：端到端数据平台、性能/成本优化、数据质量或事故排查。
- [ ] 为每个项目准备数据来源、规模、时效、你的职责、技术取舍、困难和可验证结果。
- [ ] 准备一个 CDC、迟到数据或幂等重跑的具体实例。
- [ ] 准备一个 Spark UI 或执行计划驱动的性能问题定位案例。
- [ ] 准备一个权限、敏感数据、审计或跨团队治理案例。
- [ ] 对所有数字、架构和技术选型都能解释原因、限制和替代方案。

### 面试结束前可反问的问题

1. 团队目前最重要的数据域、数据源类型和时效 SLA 是什么？
2. 现有 Lakehouse 使用批处理、流处理还是 CDC 为主？最大的数据质量挑战是什么？
3. Unity Catalog、数据所有权和生产访问权限如何治理？
4. 团队的代码评审、测试、部署和回滚流程是怎样的？
5. 这个岗位前三到六个月最希望改善的是交付速度、数据可信度、性能成本，还是平台稳定性？

这些问题体现你关注的不只是“会运行 Notebook”，而是如何交付可靠、可治理且能持续创造业务价值的数据产品。


A **Databricks notebook** is an interactive, web-based workspace for writing and running data, analytics, and machine-learning code on Databricks compute.

It is similar to a Jupyter notebook, but tightly integrated with the Databricks Lakehouse platform and Apache Spark.

### What it contains
A notebook is divided into cells that can contain:

- **Python** (often PySpark)
- **SQL**
- **Scala**
- **R**
- **Markdown** for explanations and documentation

Example:

```python
# Read a Delta table with PySpark
sales = spark.read.table("main.analytics.sales")
display(sales.groupBy("region").sum("amount"))
```

```sql
-- Query the same data with SQL
SELECT region, SUM(amount) AS total_sales
FROM main.analytics.sales
GROUP BY region;
```

### What it is used for

- Data exploration and ad hoc analysis
- ETL / ELT data transformation
- Building Delta Lake tables
- Data quality checks
- Machine-learning experimentation and feature engineering
- Creating reusable pipelines and scheduled jobs
- Collaborative investigation of data issues

### Key characteristics

| Feature | Explanation |
| --- | --- |
| Spark execution | Code can run in a distributed manner across a cluster, so it can process large data volumes. |
| Lakehouse integration | Directly accesses data in Lakehouse, Delta tables, Unity Catalog, SQL Warehouses, and cloud storage. |
| Collaboration | Multiple team members can review, comment on, and share notebooks. |
| Reproducibility | Code, explanations, parameters, and results can live together. |
| Operationalization | A notebook can be attached to a Databricks Workflow/Job and scheduled or triggered by other pipelines. |
| Version control | Notebooks can be synchronized with Git repositories, although source-file-based projects are often preferred for larger production codebases. |

### Notebook vs. production pipeline

A notebook is excellent for development, exploration, demonstrations, and small-to-medium data workflows. For production systems, teams should usually also add:

- Parameterization instead of hard-coded paths and dates
- Error handling and logging
- Tests and data-quality validation
- Git-based review and deployment
- Databricks Jobs/Workflows orchestration
- Reusable Python/SQL modules for shared business logic

In short: **a Databricks notebook is an interactive coding document that runs data workloads on Databricks/Spark infrastructure.**



