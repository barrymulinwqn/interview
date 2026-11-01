# HK Data Lakehouse（AWS Glue / S3 / Redshift / PySpark）面试高频题与详解

> **适用岗位**：AWS 数据开发、数据仓库 / BI 开发、Data Lakehouse 运维与交付工程师  
> **对应技术栈**：AWS Glue、Amazon S3、Amazon Redshift、SQL、Python、PySpark、Power BI、GitHub / CI/CD  
> **使用方式**：每题先用“简洁回答”在 60--90 秒内作答；面试官继续追问时，再展开“说明与追问”。回答真实经历时，请将示例中的数据域、规模、指标和工具替换为自己实际项目的内容，避免虚构。

---

## 目录

1. [岗位理解与项目表达](#一岗位理解与项目表达)
2. [Data Lakehouse 与总体架构](#二data-lakehouse-与总体架构)
3. [Amazon S3、数据格式与治理](#三amazon-s3数据格式与治理)
4. [AWS Glue 与 ETL 开发](#四aws-glue-与-etl-开发)
5. [Python 与 PySpark](#五python-与-pyspark)
6. [SQL 与 Amazon Redshift](#六sql-与-amazon-redshift)
7. [数据质量、调度与生产运维](#七数据质量调度与生产运维)
8. [Power BI 与语义模型](#八power-bi-与语义模型)
9. [GitHub、CI/CD 与发布](#九githubcicd-与发布)
10. [场景题与行为面试题](#十场景题与行为面试题)
11. [面试前速记清单](#十一面试前速记清单)

---

## 一、岗位理解与项目表达

### 1. 请介绍你对 HK Data Lakehouse 岗位的理解。

**简洁回答**

这是一个同时覆盖数据开发、测试、发布和生产支持的数据平台岗位。核心目标不是只写一个 Glue 脚本，而是让从源系统到 S3 / Redshift / Power BI 的数据链路在正确性、时效性、可观测性和可恢复性方面稳定运行。日常工作会包括 ETL 开发与 SQL 校验、数据模型或报表数据集调整、GitHub 协作与 CI/CD 发布，以及任务失败和数据异常的排查闭环。

**说明与追问**

可以用下图说明自己的全链路理解：

```text
业务数据库 / 文件 / API
          |
          v
S3 Raw（保留原始事实）
          |
          v
Glue / PySpark（清洗、标准化、去重、质量校验）
          |
          +--> Quarantine（拒绝记录与原因）
          |
          v
S3 Curated / Redshift（可分析数据集）
          |
          v
Power BI Semantic Model / Report

CloudWatch + 告警 + GitHub + CI/CD 贯穿全部环节
```

回答时建议强调四件事：

1. **开发正确性**：字段映射、业务口径、主键、增量逻辑和数据质量规则。
2. **生产稳定性**：重试、幂等、监控、告警、日志、故障升级与补数。
3. **交付可控性**：代码评审、自动化测试、环境隔离、发布审批和回滚。
4. **业务可用性**：报表数据刷新及时，指标口径可解释，问题有明确责任和处理记录。

---

### 2. 请用 STAR 方法介绍一个数据管道项目。

**推荐回答结构**

| STAR 要素 | 应表达的内容 | 示例句式 |
|---|---|---|
| Situation | 业务背景、来源、消费方和痛点 | “销售团队每天需要在 Power BI 查看订单与库存，但数据来自 CRM 和 ERP，手工合并导致 T+2 才能出报表。” |
| Task | 自己负责的边界和目标 | “我负责 S3 分层、Glue ETL、Redshift 数据集及上线后的监控，目标是实现每日 8:00 前可用。” |
| Action | 技术与协作动作 | “我以业务日期设计增量水位线，将原始数据保存在 Raw 层，通过 Glue Spark 清洗并写入 Parquet；对主键、金额和参照完整性设置质量规则。” |
| Result | 量化结果与沉淀 | “将时效从 T+2 改为 T+1 早晨；失败可在 15 分钟内告警；同时补充 runbook，减少同类问题的人工排查时间。” |

**高质量表达原则**

- 清楚区分“团队做了什么”和“我亲自负责什么”。
- 说清楚输入、处理、输出和验收标准，而不只罗列 AWS 服务。
- 不确定真实数值时，说“以项目实际 SLA 为准”，不要编造百分比或数据量。
- 结果不仅可以是性能，还可以是数据准确性、排障时长、交付周期和使用方满意度。

---

## 二、Data Lakehouse 与总体架构

### 3. 什么是 Data Lake、Data Warehouse 和 Data Lakehouse？三者有什么区别？

**简洁回答**

Data Lake 以低成本对象存储保存原始、多格式、大规模数据；Data Warehouse 以结构化建模和高性能 SQL 查询服务报表与分析；Lakehouse 尝试将两者结合，在数据湖存储上提供表格式、事务一致性、模式治理、版本回溯和 BI 可用的性能。对本岗位而言，S3 通常承担数据湖存储，Glue 负责目录和批处理，Redshift 或 Power BI 服务分析消费。

| 维度 | Data Lake | Data Warehouse | Data Lakehouse |
|---|---|---|---|
| 主要存储 | S3 等对象存储 | Redshift 等数仓引擎 | 对象存储 + 表格式 |
| 数据形态 | 原始、半结构化、非结构化均可 | 以结构化、建模后数据为主 | 原始和建模数据共存 |
| Schema | 多为 schema-on-read | 多为 schema-on-write | 兼顾两者，有表契约与演进 |
| 更新 / 事务 | 文件级，能力较弱 | 成熟 | Iceberg 等提供 ACID 能力 |
| 常见用途 | 留存、探索、机器学习 | 高并发 BI 与复杂分析 | 统一分析、可追溯数据产品 |

**追问：Lakehouse 是否意味着不再需要 Redshift？**

不一定。Lakehouse 不是“只用 S3”的同义词。对于高并发、复杂报表、低延迟聚合和成熟的工作负载管理，Redshift 仍常有优势。合理架构应根据数据量、更新频率、并发、成本和 SLA 决定数据留在 S3、外部表还是加载到 Redshift。

---

### 4. 请设计一个从业务系统到 Power BI 的 AWS Lakehouse 数据链路。

**推荐答案**

1. **采集层**：数据库全量或 CDC、文件、SaaS API 数据先写入 `S3 Raw`。记录来源、文件名、摄取时间、批次 ID 和原始负载。
2. **元数据层**：在 Glue Data Catalog 定义生产表。稳定数据集优先通过 IaC / DDL 管理，而不完全依赖 Crawler 自动推断。
3. **处理层**：Glue Spark Job 按业务日期读取增量，完成类型标准化、去重、关联、脱敏和质量检查。
4. **分层存储**：写入 `S3 Curated` 的 Parquet / Iceberg 表；不合格记录写入 Quarantine 并携带失败原因。
5. **服务层**：将高频、低延迟报表所需事实和维度加载或同步至 Redshift，或由 Redshift Spectrum / Athena 查询湖中表。
6. **BI 层**：Power BI 连接经治理的数据集，构建星型语义模型、度量值和 RLS；根据数据量选择 Import、Incremental Refresh 或 DirectQuery。
7. **运维与治理**：CloudWatch 记录日志和指标；EventBridge / Step Functions 编排依赖；IAM、KMS、Lake Formation 或 Bucket Policy 限制访问；GitHub Actions 执行测试和部署。

**关键设计原则**

- Raw 层尽量不可变，以便审计与重放。
- 每次运行有唯一 `run_id`，可追踪“哪批源数据生成了哪批目标数据”。
- 失败要有明确状态和告警，不能让“任务运行成功”掩盖数据质量失败。
- 业务日期（`business_date`）与摄取时间（`ingest_timestamp`）分开保存。

---

### 5. 什么是 Bronze / Silver / Gold 分层？每层该放什么？

**简洁回答**

Bronze（或 Raw）保存接近来源的原始事实；Silver（或 Curated）是完成清洗、标准化、去重和可关联处理的明细层；Gold（或 Serving）是面向指标、报表或应用的聚合层与数据产品。

| 层 | 常见内容 | 允许的操作 | 不应做的事 |
|---|---|---|---|
| Bronze | 原始 JSON、CSV、CDC、API 响应 | 增加摄取元数据、压缩、分目录 | 用修正值覆盖原始记录 |
| Silver | 标准字段、统一主键、可连接明细 | 类型转换、去重、质量校验、脱敏 | 只为单一报表任意堆叠指标 |
| Gold | 销售日报、客户 360、星型模型、指标表 | 聚合、业务口径、权限消费 | 把所有原始字段都复制进去 |

**易错点**

- 分层不是“复制三份文件”，而是每层都有明确消费者、数据契约与质量责任。
- Bronze 数据也需要安全治理，原始数据往往包含最敏感字段。
- 若要重跑 Silver，不应依赖只能保留一天的临时源数据，而应可从 Bronze 重放。

---

### 6. 如何设计增量加载？什么是水位线（watermark）？

**简洁回答**

增量加载只处理上次成功处理之后新增或变更的数据。水位线是记录该边界的状态，例如源表的 `updated_at`、递增 ID、CDC LSN 或文件到达时间。生产中应以“成功提交的批次”更新水位线，不能在任务一开始就推进它。

**推荐步骤**

1. 读取持久化的上次成功水位线，例如 `2026-10-27 00:00:00`。
2. 使用重叠窗口提取数据，例如从水位线向前回退两小时，以容忍迟到数据。
3. 将当前批次原始数据和 `run_id` 落入 Raw。
4. 在 Silver 层以业务主键和最新变更时间完成去重 / `MERGE`。
5. 质量校验和目标提交成功后，原子地更新新的水位线。
6. 保存输入行数、输出行数、最大源时间和版本，方便审计。

**迟到数据如何处理？**

不能简单按“今天的数据”过滤。应定义允许迟到窗口、按照业务主键做幂等合并，必要时重算受影响的 Gold 日期分区。若迟到超过 SLA，应产生数据质量事件并与数据所有者确认处理策略。

---

### 7. 什么是幂等（idempotency）？为什么对 ETL 很重要？

**简洁回答**

一个任务同一输入执行一次或多次，得到的目标状态相同，就称为幂等。ETL 任务可能因网络、基础设施或下游故障重试；若不幂等，重试可能产生重复事实、重复文件或重复指标。

**常用实现方式**

- 对每条数据使用稳定的业务主键，而不是每次运行生成新 ID。
- 写入分区前采用安全的覆盖 / 交换策略，避免直接追加相同业务日期。
- 使用 Iceberg / Redshift `MERGE` 按主键处理插入和更新。
- 在目标中记录 `source_batch_id` 或文件清单，已成功处理的输入不重复消费。
- 任务完成后再提交 checkpoint / watermark。

**反例**

```text
任务每次执行都 append 到 sales_daily，
但没有按 order_id 去重，也没有按 business_date 重写。
一次重试会让销售额翻倍。
```

---

## 三、Amazon S3、数据格式与治理

### 8. S3 在 Data Lakehouse 中承担什么角色？如何组织路径？

**简洁回答**

S3 是高可用、低成本的对象存储，通常用于保存原始数据、清洗后的列式数据、ETL 临时输出和数据质量隔离记录。路径应该表达数据域、层级、来源与分区，便于授权、生命周期管理和分区裁剪。

```text
s3://<bucket>/raw/crm/customer/ingest_date=2026-10-28/run_id=<uuid>/
s3://<bucket>/curated/sales/order/business_date=2026-10-28/
s3://<bucket>/serving/finance/daily_revenue/business_date=2026-10-28/
s3://<bucket>/quarantine/crm/customer/reason=invalid_email/
```

**设计说明**

- 将 bucket / prefix 的权限边界与数据域、环境相匹配，例如 `dev`、`test`、`prod` 严格隔离。
- `business_date` 适合业务数据查询；`ingest_date` 适合摄取审计。两者不应混为一谈。
- 不建议按 `customer_id`、UUID 或秒级时间建目录，会制造海量小分区。
- 配置版本控制、生命周期策略、SSE-KMS 加密、访问日志和 Block Public Access。

---

### 9. 为什么分析数据通常使用 Parquet，而不是 CSV 或 JSON？

**简洁回答**

Parquet 是列式格式，可以只读取查询需要的列，并具有较好的压缩和类型保存能力，通常能降低 S3 扫描量、加快 Athena / Spark / Redshift Spectrum 查询。CSV 和 JSON 适合交换或原始落地，但类型弱、体积更大且解析成本更高。

| 对比项 | CSV / JSON | Parquet |
|---|---|---|
| 存储方式 | 行式、文本 | 列式、二进制 |
| 类型信息 | 弱，常需推断 | 强，保留 schema |
| 列裁剪 | 较差 | 好 |
| 压缩 | 一般 | 通常较好 |
| 人工排查 | 易直接阅读 | 需要工具查看 |
| 适用位置 | Raw / 交换 | Curated / Serving |

**注意**

Parquet 不能自动解决数据质量和重复问题。错误类型、错误分区或重复记录写成 Parquet 后，仍然是错误数据。

---

### 10. 什么是分区裁剪（partition pruning）？如何选择分区键？

**简洁回答**

分区裁剪指查询条件命中分区字段时，查询引擎只扫描相关目录或文件，而不扫描全表。对按日期查询的事实表，`business_date` 常是合理分区键。

```sql
-- 可裁剪：如果 business_date 是分区字段
SELECT SUM(net_amount)
FROM curated.sales_order
WHERE business_date BETWEEN DATE '2026-10-01' AND DATE '2026-10-07';
```

**选择原则**

1. 先看真实查询中最常见且选择性合适的过滤条件。
2. 控制分区数量，日分区通常比分钟或用户级分区更稳妥。
3. 分区文件应有合理大小；大量几 KB / 几 MB 文件会让元数据和调度开销主导性能。
4. 高基数字段（如 `order_id`、用户 ID）通常不适合分区，可通过排序、Z-Order（视引擎支持）或适当的聚合来优化。

---

### 11. 如何保护 S3 中的敏感数据？

**推荐回答**

- **传输与静态加密**：强制 TLS；使用 SSE-KMS，并控制 KMS Key 的解密权限。
- **最小权限**：Glue Job Role、BI 访问角色、运维人员使用不同 IAM Role，只授权到所需 bucket / prefix / action。
- **数据分级和脱敏**：识别 PII、医疗、财务等敏感字段；在 Silver / Gold 使用掩码、哈希、令牌化或删除，避免给报表暴露原始个人信息。
- **细粒度授权**：需要表、列、行级控制时，使用 Lake Formation 或与数仓权限、RLS 协同。
- **审计与留存**：启用 CloudTrail data events（按成本评估）、S3 access logs / CloudTrail、版本控制和生命周期策略。

**面试加分点**

权限不仅是“能否读取 bucket”。用户拥有 S3 读取权限但没有 KMS 解密权限时仍无法读取加密对象；反过来，KMS 权限也不应替代 S3 的对象访问控制。

---

### 12. 什么是小文件问题？如何解决？

**简洁回答**

小文件问题是指一个数据集包含大量很小的文件。Spark、Athena、Redshift Spectrum 等需要为每个文件创建任务、读取元数据和建立连接，导致调度开销大、查询变慢、成本上升。

**处理方式**

- 控制写入分区数，避免无意义的高 `repartition()`。
- 定期执行 compaction，将小文件合并为合理大小的 Parquet 文件。
- 使用合适的日期 / 数据域分区，而不是高基数字段分区。
- 调整上游微批策略；对流式或频繁小批次写入，安排后续压实任务。
- 在 Iceberg 等表格式中使用其 compaction / rewrite data files 能力，并安排快照维护。

不要直接说“文件越大越好”。超大文件会降低并行度，并使单个任务重试成本过高；应通过真实查询和作业指标确定目标范围。

---

## 四、AWS Glue 与 ETL 开发

### 13. AWS Glue 的组成和职责是什么？

**简洁回答**

Glue 是 AWS 的托管数据集成服务。常用组成包括 Glue Data Catalog（元数据）、Crawler（发现 schema）、Connections（网络与数据源连接）、Jobs（Spark / Python Shell / Streaming 任务）、Triggers / Workflows（调度）以及 Data Quality 等能力。Glue 本身不替代 S3、Redshift 或权限体系，而是将数据发现、处理和编排连接起来。

**常见 Job 选择**

| 类型 | 适合场景 | 不适合场景 |
|---|---|---|
| Glue Spark Job | 大规模清洗、Join、聚合、分区写入 | 简单 SDK 调用或几十秒脚本 |
| Glue Python Shell | 轻量 API、文件控制、元数据操作 | 大量 DataFrame 计算 |
| Glue Streaming | Kinesis / Kafka 持续处理 | 每日批量任务 |

---

### 14. Glue DynamicFrame 和 Spark DataFrame 有什么区别？如何选择？

**简洁回答**

DynamicFrame 是 Glue 对半结构化和 schema 不一致数据的抽象，具有 `resolveChoice` 等处理类型歧义的能力，且方便从 Catalog 读取。DataFrame 是 Spark 的标准结构化 API，SQL 函数、优化能力、生态兼容性和可测试性通常更好。常见做法是从 Catalog 读入 DynamicFrame，处理完模式歧义后转成 DataFrame 执行主要转换，最终再按目标需要写出。

**选择建议**

- 输入 schema 经常漂移、嵌套 JSON 多、需使用 Glue 特有转换：先使用 DynamicFrame。
- 复杂 Join、窗口函数、聚合、明确 schema 的批处理：优先 DataFrame / Spark SQL。
- 不要因为 Glue 有 DynamicFrame 就放弃熟悉的 Spark 优化和测试手段。

---

### 15. 请描述一个生产级 Glue Job 的基本结构。

**参考代码**

```python
import sys
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from pyspark.sql import functions as F

args = getResolvedOptions(
    sys.argv,
    ["JOB_NAME", "RUN_DATE", "SOURCE_DATABASE", "SOURCE_TABLE", "TARGET_PATH"],
)

sc = SparkContext.getOrCreate()
glue_context = GlueContext(sc)
spark = glue_context.spark_session
job = Job(glue_context)
job.init(args["JOB_NAME"], args)

source = glue_context.create_dynamic_frame.from_catalog(
    database=args["SOURCE_DATABASE"],
    table_name=args["SOURCE_TABLE"],
    push_down_predicate=f"business_date = '{args['RUN_DATE']}'",
)

orders = source.toDF()
valid_orders = (
    orders
    .withColumn("order_id", F.col("order_id").cast("string"))
    .withColumn("net_amount", F.col("net_amount").cast("decimal(18,2)"))
    .filter(F.col("order_id").isNotNull() & (F.col("net_amount") >= 0))
)

valid_orders.write.mode("overwrite").partitionBy("business_date").parquet(
    args["TARGET_PATH"]
)

job.commit()
```

**代码说明**

- 参数化运行日期、输入表和输出路径，避免把环境或日期硬编码。
- 使用 `push_down_predicate` 尽可能在读取早期过滤分区。
- 显式转换关键字段类型，避免字符串金额或隐式类型造成错误。
- 真正的生产任务还需加入输入 / 输出计数、质量阈值、异常记录写入、结构化日志、重试策略和目标提交语义。
- `job.commit()` 对启用 Glue Job Bookmarks 的任务尤其重要；是否启用 Bookmark 要与自己的增量设计一致，不能同时使用两套互相冲突的水位线逻辑。

---

### 16. Glue Job 如何访问私有 RDS 或 Redshift？应排查哪些网络问题？

**简洁回答**

Glue Job 需通过 Glue Connection 或 Job 的 VPC 配置进入可访问目标数据库的子网与安全组。网络连通还不够：任务运行子网必须有到 S3、CloudWatch、Secrets Manager 等 AWS 服务的可达路径，通常是 NAT Gateway 或相应 VPC Endpoint。

**排查顺序**

1. 检查 Connection 的子网、安全组、数据库端口和 JDBC URL。
2. 检查数据库安全组入站规则是否允许来自 Glue 安全组，而不是只依赖固定 IP。
3. 检查路由表、NACL、DNS 解析和 VPC Endpoint / NAT 路径。
4. 检查用户名、密码或 Secrets Manager 读取权限。
5. 在 CloudWatch 日志确认是超时、认证失败、TLS 问题还是 JDBC 驱动问题。

**容易遗漏的点**

把 Glue 放进没有 NAT 或 S3 Endpoint 的私有子网后，任务可能能连接数据库，却无法下载依赖、写 S3 或发送日志。

---

### 17. Crawler 在生产环境中有哪些风险？如何管理 Schema 演进？

**简洁回答**

Crawler 适合快速发现文件和探索数据，但它会基于样本推断字段类型，异常文件可能让类型错误或表 schema 意外变化。对于生产数据集，应以 schema contract、DDL / IaC、版本控制和兼容性校验为主。

**Schema 演进策略**

| 变更类型 | 风险 | 推荐处理 |
|---|---|---|
| 新增可空列 | 一般可向后兼容 | 明确发布 schema 版本，消费端设默认逻辑 |
| 字段重命名 | 容易中断消费 | 过渡期保留旧字段或创建兼容视图 |
| 类型扩大（int 到 bigint） | 需检查下游 | 做兼容性测试，明确迁移时间 |
| 类型缩小 / 语义变化 | 高风险 | 新建字段或版本化数据集，不能静默替换 |
| 删除字段 | 高风险 | 先通知、观测使用情况、经过弃用期再删除 |

---

### 18. Glue Job 失败时，你如何快速定位问题？

**建议采用固定排查框架**

1. **确认影响范围**：哪个 Job Run、哪个环境、哪个业务日期、影响哪些表和 Power BI 刷新。
2. **查看 CloudWatch 日志**：找到第一条根因错误，而非最后的连锁异常。
3. **分类故障**：权限 / KMS、网络 / JDBC、源数据格式、schema、Spark 内存、代码、下游写入、配额或超时。
4. **检查输入与变更**：当天文件数量、大小、字段、源端延迟、近期代码和配置发布。
5. **采取安全恢复动作**：修复后从可重放的 Raw 数据重跑；确认幂等性；避免盲目重复触发。
6. **验证和闭环**：校验行数、金额、关键维度、报表刷新；记录时间线、根因、修复和预防项。

**常见症状与方向**

| 症状 | 常见原因 | 首先检查 |
|---|---|---|
| `AccessDenied` | S3 / KMS / Catalog / Secret 权限不足 | Job Role、Bucket Policy、KMS Key Policy |
| JDBC timeout | 安全组、路由、端口、数据库负载 | VPC Connection、SG、路由、数据库日志 |
| `AnalysisException` | 字段不存在、schema 漂移、SQL 错误 | 输入 schema、近期 DDL / 代码变更 |
| Executor OOM | 数据倾斜、大表 Join、缓存过多 | Spark UI / 日志、分区、Join 策略 |
| 输出重复 | 非幂等重试、覆盖范围错误 | run_id、分区写入策略、主键去重 |

---

## 五、Python 与 PySpark

### 19. PySpark 的 lazy evaluation 是什么？有什么实际意义？

**简洁回答**

DataFrame 的多数转换，例如 `select`、`filter`、`join`，只是构建执行计划，不会立刻计算；直到 `count()`、`write()`、`collect()` 等 action 才真正执行。这称为惰性求值。它让 Spark 能合并和优化转换，但也意味着调试时一个看似简单的 `count()` 可能触发完整作业。

**面试可补充**

- 不要在生产 ETL 中为调试频繁调用 `count()` 或 `collect()`，它们会增加计算成本。
- `collect()` 会把数据拉到 Driver，大数据集可能导致 Driver 内存溢出；调试时使用 `limit()`、`show()` 或针对性采样。
- 缓存只适合被重复使用且成本高的中间结果；缓存后要监控内存并在不再需要时 `unpersist()`。

---

### 20. 如何优化一个慢的 PySpark Job？

**推荐回答框架：先测量，再优化**

1. **减少读入数据**：使用列裁剪、分区过滤、`push_down_predicate`，不要先读全表再过滤。
2. **检查 Join**：小表可使用 broadcast join；大表 Join 前过滤和选择必要列；关注 key 倾斜。
3. **减少 Shuffle**：避免不必要的 `distinct`、全局 `orderBy` 和连续 `repartition`；预聚合后再 Join。
4. **改善文件布局**：Parquet、合理分区、压实小文件。
5. **调节资源**：根据 executor OOM、shuffle spill、并发和数据量调整 worker / DPU、shuffle 分区，而不是盲目加资源。
6. **使用执行证据**：根据 Spark UI、CloudWatch 指标、stage 时间、输入文件数和 spill 数据判断瓶颈。

**示例：减少不必要的数据移动**

```python
from pyspark.sql import functions as F

recent_orders = (
    orders
    .filter(F.col("business_date") >= F.lit("2026-10-01"))
    .select("customer_id", "order_id", "net_amount")
)

active_customers = customers.filter(F.col("is_active")).select("customer_id", "segment")

result = recent_orders.join(F.broadcast(active_customers), "customer_id", "left")
```

只有在维表确实足够小、能安全放进 executor 内存时才能 broadcast。不可把它当作所有 Join 的默认答案。

---

### 21. 什么是数据倾斜（data skew）？如何处理？

**简洁回答**

当某些分区或某个 Join Key 的数据量远大于其他分区时，少数 task 会远慢于其他 task，甚至内存溢出，这就是数据倾斜。例如“未知客户”或某个大客户占据大量订单。

**识别方式**

- Spark UI 中大部分 task 已完成，但少数 task 运行很久。
- 某些 task 的 shuffle read / write 或输入行数远大于平均值。
- 失败总集中在个别 executor。

**处理方式**

- 检查并修正异常 key，如空值、默认值集中到一个 key。
- 对小表使用 broadcast，减少大规模 shuffle。
- 对极端热点 key 进行 salting，并在另一侧复制对应维度记录；此方案要谨慎，且验证重复与聚合正确性。
- 先对事实数据预聚合，降低 Join 前数据量。
- 开启并验证 Spark Adaptive Query Execution（能力取决于运行时与配置）。

---

### 22. 如何在 PySpark 中做去重并保留最新记录？

**参考代码**

```python
from pyspark.sql import Window
from pyspark.sql import functions as F

latest_per_order = (
    source
    .withColumn(
        "row_number",
        F.row_number().over(
            Window.partitionBy("order_id").orderBy(
                F.col("updated_at").desc(),
                F.col("ingest_timestamp").desc(),
            )
        ),
    )
    .filter(F.col("row_number") == 1)
    .drop("row_number")
)
```

**说明**

- `dropDuplicates(["order_id"])` 对“保留哪一条”没有清晰业务语义，不适合需要保留最新版本的场景。
- 排序规则必须稳定。若 `updated_at` 相同，需要 `ingest_timestamp`、源序列号或 CDC LSN 等第二排序字段。
- 在数据进入 Gold 前，要明确业务上“更新”“删除”“撤销”分别如何表达，不能只做技术去重。

---

### 23. Python 脚本中如何处理配置、日志和异常？

**推荐回答**

- **配置**：环境、日期、路径、数据库连接名通过 Job 参数、环境变量或受控配置文件注入；密码来自 Secrets Manager，而非 Git 仓库或日志。
- **日志**：用结构化、可搜索的日志记录 `run_id`、数据集、业务日期、输入 / 输出行数、最大水位线与错误类别。
- **异常**：只捕获可预期且能处理的异常；记录上下文后重新抛出，让 Glue Job 正确失败并触发告警。不要写广泛的 `except Exception: pass`。
- **测试**：将纯业务转换抽成普通 Python 函数，使用小型 DataFrame fixture 做单元测试；端到端测试使用隔离 bucket / schema。

```python
try:
    output = transform_orders(source, run_date)
    validate_orders(output)
    write_output(output)
except ValueError as error:
    logger.exception("Validation failed for run_date=%s", run_date)
    raise
```

上例是“记录后失败”，不是吞掉异常。实际代码还应按项目日志框架实现。

---

## 六、SQL 与 Amazon Redshift

### 24. 如何用 SQL 找出重复记录？

**答案**

先定义“重复”的业务主键是什么，例如订单应以 `order_id` 唯一，而不是整行字段完全相同。

```sql
SELECT
    order_id,
    COUNT(*) AS record_count
FROM curated.sales_order
GROUP BY order_id
HAVING COUNT(*) > 1;
```

如果要保留最新记录、识别待清理记录，可使用窗口函数：

```sql
SELECT *
FROM (
    SELECT
        order_id,
        updated_at,
        ROW_NUMBER() OVER (
            PARTITION BY order_id
            ORDER BY updated_at DESC, ingest_timestamp DESC
        ) AS row_number
    FROM curated.sales_order
) ranked
WHERE row_number > 1;
```

面试中应指出：SQL 能查出重复，但修复策略取决于业务语义，是保留最新、合并字段，还是标记为异常。

---

### 25. `WHERE` 和 `HAVING` 有什么区别？

**简洁回答**

`WHERE` 在分组聚合前过滤行；`HAVING` 在 `GROUP BY` 聚合后过滤分组。能在 `WHERE` 过滤的条件应尽量放在 `WHERE`，可减少参与聚合的数据量。

```sql
SELECT
    customer_id,
    SUM(net_amount) AS total_amount
FROM sales_order
WHERE business_date >= DATE '2026-10-01'
GROUP BY customer_id
HAVING SUM(net_amount) > 10000;
```

---

### 26. 窗口函数有什么用？请举一个排名和累计值的例子。

**答案**

窗口函数在不折叠明细行的前提下，按指定分区计算排名、累计值、上期值等分析指标。

```sql
SELECT
    business_date,
    customer_id,
    net_amount,
    SUM(net_amount) OVER (
        PARTITION BY customer_id
        ORDER BY business_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS customer_running_amount,
    ROW_NUMBER() OVER (
        PARTITION BY business_date
        ORDER BY net_amount DESC
    ) AS daily_sales_rank
FROM sales_order;
```

**常见易错点**

- `ROW_NUMBER()`、`RANK()`、`DENSE_RANK()` 对并列值的处理不同，必须按业务要求选择。
- 累计值需要明确窗口 frame；不明确时，不同数据库的默认行为可能不符合预期。
- 窗口函数通常在 `WHERE` 之后计算，若要过滤其结果，一般使用 CTE / 子查询或支持 `QUALIFY` 的数据库语法。

---

### 27. 什么是 SCD（缓慢变化维）？Type 1 与 Type 2 如何选择？

**简洁回答**

SCD 用于处理维度属性随时间变化，例如客户地址、产品分类或员工部门。Type 1 直接覆盖旧值，不保留历史；Type 2 为每个变化保留版本，通常包含生效开始 / 结束时间和当前标志。

| 类型 | 做法 | 适合场景 |
|---|---|---|
| Type 1 | 覆盖当前值 | 纠错、历史不重要的属性 |
| Type 2 | 新增版本行，维护有效期 | 需要按历史属性还原事实分析 |

**Type 2 示例字段**

```text
customer_sk, customer_id, customer_segment,
effective_from, effective_to, is_current
```

若订单事实发生在客户升级为 VIP 前，报表是按“下单时的客户等级”还是“当前等级”统计？这个业务问题决定是否使用 Type 2，而不是只由技术偏好决定。

---

### 28. Redshift 的分布（distribution）和排序（sort）概念是什么？

**简洁回答**

Redshift 是分布式数仓。分布方式决定行如何分布到计算节点，影响 Join 时是否需要跨节点搬移数据；排序键决定数据在磁盘上的物理有序性，影响范围过滤和扫描效率。

**回答要点**

- 经常与大事实表 Join 的同一高基数键，可能适合作为相关表的 `DISTKEY`，以减少数据重新分布。
- 经常按日期范围过滤的大表，日期可能适合作为 `SORTKEY` 的前导列。
- 小维表可使用 `DISTSTYLE ALL`，以空间换取 Join 性能；但不适用于大表。
- 不能只凭经验设置，要看实际 Query Plan、表大小、过滤条件、Join 模式、数据倾斜和 Redshift Advisor 建议。

**追问：为什么错误的分布键会慢？**

如果两个大表按不同键分布，Join 时大量数据需要在节点间重新分发（data redistribution），网络移动可能主导查询时间。若某个键值过于集中，又会造成节点倾斜。

---

### 29. 如何将数据安全、高效地加载进 Redshift？

**推荐回答**

对于大量文件，优先使用 Redshift `COPY` 从 S3 并行加载，而不是逐行 `INSERT`。文件应按适当大小切分、使用 Parquet 等高效格式，并通过 IAM Role 授权 Redshift 读取 S3。

```sql
COPY analytics.sales_order
FROM 's3://company-prod-curated/sales_order/'
IAM_ROLE 'arn:aws:iam::<account-id>:role/RedshiftCopyRole'
FORMAT AS PARQUET;
```

**生产注意事项**

- 先加载 staging 表，再进行校验与合并，避免半成功状态直接暴露给报表。
- 对加载批次记录文件清单、行数、校验结果和 `run_id`。
- 明确重复加载策略：同一批文件重跑是否覆盖、去重或拒绝。
- 使用最小权限 IAM Role；不要在脚本中嵌入 Access Key。
- 加载后根据表设计和平台建议维护统计信息 / 排序，具体运维动作应遵从当前 Redshift 类型与团队标准。

---

### 30. Redshift Spectrum、Athena 和 Redshift 本地表应如何选择？

| 方案 | 优点 | 适合场景 | 关注点 |
|---|---|---|---|
| Athena | 无服务器、按扫描量计费、直接查 S3 | 临时分析、探索、较低频湖查询 | Parquet、分区、扫描成本 |
| Redshift Spectrum | 在 Redshift SQL 中访问 S3 外部表 | 数仓表与湖表联合查询 | 外部表布局、并发与扫描量 |
| Redshift 本地表 | 性能与并发更可控 | 高频 BI、复杂 Join、低延迟需求 | 数据加载、建模、存储和集群成本 |

不是所有数据都需要复制到 Redshift。应根据消费频率、数据新鲜度、并发、延迟、成本与治理要求决定；Gold 层高频报表数据常更适合以优化后的本地表或物化视图服务。

---

## 七、数据质量、调度与生产运维

### 31. 数据质量检查应覆盖哪些维度？

**简洁回答**

数据质量不只是“任务没报错”。至少包括完整性、唯一性、有效性、一致性、及时性和准确性；每项都应有可执行规则、阈值、责任人和失败后的处理路径。

| 维度 | 示例规则 | 失败处理示例 |
|---|---|---|
| 完整性 | `order_id` 非空率为 100% | 写入 Quarantine，阻断下游 |
| 唯一性 | 订单主键无重复 | 阻断 Gold 发布并发出告警 |
| 有效性 | 金额 >= 0，币种在允许集合内 | 隔离无效记录，生成原因统计 |
| 参照完整性 | 订单客户可匹配客户维度 | 标记未匹配并等待维表补齐 |
| 一致性 | 明细金额汇总与源端控制总额一致 | 超阈值时人工确认 |
| 及时性 | 每日 08:00 前有当日应到数据 | 触发 SLA 告警 |

**关键原则**

质量结果本身也是数据，应保存规则版本、检测时间、批次、样本记录和结果。对于关键财务或合规报表，质量失败必须阻止不可信数据继续发布。

---

### 32. 如何设计 ETL 任务监控和告警？

**推荐答案**

应同时监控**技术健康度**和**数据健康度**：

- **技术健康度**：任务状态、运行时长、重试次数、DPU / executor 错误、读取 / 写入失败、并发和配额。
- **数据健康度**：输入文件是否到达、行数波动、空值率、重复率、控制总额、最大业务时间、水位线推进、质量规则结果。
- **业务健康度**：Power BI 数据集刷新是否完成、关键报表是否在 SLA 前可用。

**告警设计**

1. 告警消息包含环境、Job 名称、运行 ID、业务日期、失败阶段、影响数据集和日志链接。
2. 区分 Warning 与 Critical，避免所有小波动都唤醒值班人员。
3. 为重复告警做聚合和抑制，防止告警风暴。
4. 每个告警关联 Runbook、责任人和升级路径。

---

### 33. 上游文件没有按时到达，你如何处理？

**建议回答**

1. 不应让下游以“空文件”或旧数据静默成功。先检查预期文件清单、文件大小、到达时间和控制表。
2. 确认是源端未生成、传输失败、权限问题、命名变化还是调度依赖未触发。
3. 按 SLA 和影响范围通知数据所有者、报表使用方与值班团队，说明当前状态和下一次更新时间。
4. 根据业务批准决定等待、使用已认证的最近快照并显著标注，或暂停下游发布。
5. 数据补到后执行幂等回补，验证控制总额、分区和报表结果。
6. 记录根因和预防措施，例如到达检测、事件触发、文件契约和提前预警。

核心原则：**没有数据不等于零数据**，更不能被当成成功结果。

---

### 34. 如何安全地做历史回补（backfill）？

**简洁回答**

回补应是独立、可审计、限范围、可恢复的操作，而不是修改生产任务默认日期后直接运行。

**推荐流程**

1. 明确回补时间窗、原因、影响数据集、是否影响已发布报表。
2. 为回补使用显式参数，例如 `--start-date`、`--end-date`、`--mode=backfill`，并确认不会自动推进日常 watermark。
3. 从可信 Raw 数据重放，使用与日常一致的转换和质量规则。
4. 以分区交换、暂存表加校验或支持事务的表格式安全发布结果。
5. 对比回补前后行数、金额、主键、关键指标与下游报表。
6. 记录审批、执行人、代码版本、运行 ID 和结果。

---

### 35. 如何制定 Runbook（运维手册）？

**Runbook 最小内容**

| 内容 | 说明 |
|---|---|
| 服务 / 数据集概述 | 业务用途、上游、下游、数据所有者、SLA |
| 调度和依赖 | 触发时间、前置任务、输入到达条件 |
| 正常运行标准 | 预期时长、输入 / 输出量、质量阈值 |
| 监控入口 | CloudWatch Dashboard、日志、告警频道、Job Run 链接 |
| 常见故障 | 症状、检查命令 / 页面、根因候选、修复步骤 |
| 重跑与回补 | 幂等前提、允许参数、风险、验证步骤 |
| 升级路径 | 一线、数据所有者、平台团队和业务联系人 |
| 变更记录 | 最近版本、发布日期、已知限制 |

好的 Runbook 让不熟悉该任务的值班同事能安全完成第一轮诊断与恢复，而不是只写“联系开发人员”。

---

## 八、Power BI 与语义模型

### 36. 什么是 Power BI Semantic Model？为什么不能让报表直接连接原始表？

**简洁回答**

Semantic Model 是报表之上的业务语义层，包含事实与维度关系、度量值（DAX）、字段含义、数据类型、层级、刷新策略和安全规则。它将“表字段”转化为统一、可复用的业务指标。

直接连接原始表的问题包括：

- 每份报表自行定义“销售额”“活跃客户”等口径，结果不一致。
- 表关系复杂且性能差，报表作者易误用字段。
- 原始字段可能含敏感信息或未清洗数据。
- 缺少统一的 RLS、刷新和变更治理。

---

### 37. Import、DirectQuery 和 Incremental Refresh 如何选择？

| 模式 | 特点 | 适用场景 | 风险 / 注意点 |
|---|---|---|---|
| Import | 数据导入 Power BI 内存模型，查询快 | 数据量适中、刷新窗口可接受 | 刷新时间和容量限制 |
| DirectQuery | 查询时实时访问 Redshift 等数据源 | 需要较新数据或数据过大 | 查询延迟、源端并发与模型限制 |
| Incremental Refresh | 只刷新近期分区，历史保留 | 大事实表按时间持续增长 | 必须设计时间字段、查询折叠和分区策略 |

**面试回答重点**

先问业务需要的刷新频率、数据量、用户并发、容量和延迟，而不是绝对地说某种模式“最好”。即使用 DirectQuery，也需要优化数仓模型、聚合表和 SQL，不能把所有性能问题转移给源端。

---

### 38. 什么是星型模型？为什么它适合 BI？

**简洁回答**

星型模型以事实表记录可加总的业务事件，以维度表描述时间、客户、产品、地区等分析角度。事实表通过外键关联维度表，结构直观，适合报表工具建立关系和度量值。

```text
                dim_date
                   |
dim_customer -- fact_sales -- dim_product
                   |
              dim_region
```

**设计要点**

- 先定义事实表的粒度，例如“一行代表一个订单明细”。
- 维度使用一致的业务键或代理键；处理慢变维历史需求。
- 度量值应避免重复计数；不同粒度的事实表不要随意直接 Join。
- 维度关系应尽量清晰、单向，减少多对多和双向过滤带来的歧义。

---

### 39. 如何排查 Power BI 报表数字与源系统不一致？

**排查路径**

1. 明确比较条件：同一业务日期、时区、刷新版本、筛选条件、币种和口径。
2. 检查源到 Raw 的输入是否完整：文件 / CDC 是否漏到或重复到。
3. 检查 Silver / Gold：行数、主键去重、Join 丢失、维度匹配、金额转换和回补情况。
4. 检查 Redshift / SQL 数据集：视图版本、过滤条件、聚合粒度。
5. 检查 Semantic Model：关系、DAX 度量、RLS、刷新时间和缓存。
6. 将差异量化为可定位的集合，例如按日期、区域、订单状态逐层对账，而不是只比较一个总数。

最终输出应包含：差异根因、影响范围、纠正方案、历史数据是否需要重算，以及防复发的质量规则。

---

## 九、GitHub、CI/CD 与发布

### 40. 数据 ETL 项目中 Git 分支和 Pull Request 应如何使用？

**推荐回答**

- 主分支应始终保持可发布状态；开发从受控分支创建 feature / bugfix 分支。
- 一个 PR 聚焦一个可审查的目的，例如“新增客户清洗规则”，不要混入无关格式化或大范围重构。
- PR 应包含业务背景、受影响的数据集、schema / 回填影响、测试证据、部署与回滚说明。
- 通过保护规则要求代码审查和 CI 成功；避免直接向生产分支推送。
- 不提交密码、Access Key、导出的生产数据或包含 PII 的调试文件。

**数据项目的额外审查点**

SQL 和 PySpark 代码审查不应只看语法，还要看重复写入风险、分区覆盖范围、schema 兼容性、数据质量规则、权限变化和成本影响。

---

### 41. 一个适合 Glue 项目的 CI/CD 流水线应包括什么？

**推荐流程**

```text
Pull Request
  -> 格式化 / Lint / 静态检查
  -> Python 单元测试与 SQL 校验
  -> 打包脚本和依赖
  -> 部署到隔离开发 / 测试环境
  -> 小数据集集成测试与质量断言
  -> 审批
  -> 部署生产
  -> Smoke Test、监控与发布记录
```

**部署对象不应只有 Glue 脚本**

还可能包括 Glue Job 配置、IAM Role / Policy、S3 路径策略、Catalog 表、Redshift DDL / View、EventBridge 调度、CloudWatch 告警和 Power BI 数据集相关配置。建议使用 CDK、CloudFormation 或 Terraform 等 IaC 保持环境一致，减少控制台手工配置漂移。

---

### 42. 如何进行安全发布和回滚？

**回答要点**

1. 使用不可变的版本标识，例如 Git commit SHA 或构建产物版本。
2. 生产前在代表性样本上完成转换、质量和兼容性验证。
3. 优先采用向后兼容 schema 变更；对破坏性变更使用版本化表 / 视图和明确的迁移窗口。
4. 发布时保留上一个可运行脚本、Job 配置和基础设施版本。
5. 若发现问题，先停止扩大影响，再回滚代码 / 配置或切换至上个稳定数据集；必要时从 Raw 重算受影响分区。
6. 回滚后验证业务指标、告警状态和 Power BI 刷新，而不是只验证部署命令成功。

---

## 十、场景题与行为面试题

### 43. 场景题：Glue 任务显示成功，但报表中的销售额为 0。你如何排查？

**高分答案**

我会先把“作业成功”和“数据正确”分开处理：

1. 检查报表使用的业务日期、刷新时间、筛选器、RLS 和度量值，确认不是展示层条件问题。
2. 检查 Gold / Redshift 服务表该日期是否有数据，核对表名、环境、视图版本和分区。
3. 回溯 Glue Job 的输入 / 输出行数、质量结果、目标路径和写入模式。重点查看过滤条件是否误把当天数据过滤掉、分区是否写错、是否写到了测试路径。
4. 从 Raw 层抽样验证输入是否真的到达且金额字段可解析；检查金额是否在类型转换时变为 null。
5. 对比 Silver、Gold、Power BI 每一层的行数与金额，定位第一处出现零值 / 差异的位置。
6. 修复后只回补受影响分区，并验证报表、质量检查与告警恢复。

这道题的关键是避免回答“重跑一次”。盲目重跑既可能无效，也可能在非幂等任务中制造重复。

---

### 44. 场景题：每天的 Job 从 20 分钟变成 2 小时，你会怎么处理？

**回答框架**

1. **确定变化点**：是从哪天、哪个 stage 开始变慢？近期是否有代码、schema、数据量、分区、Glue 版本或资源配置变更？
2. **查看证据**：CloudWatch / Spark UI 的输入数据量、文件数、shuffle、spill、慢 task、executor 失败重试。
3. **分类定位**：
   - 文件数激增：小文件问题；
   - 单个 task 特别慢：数据倾斜；
   - Join stage 慢：Join 策略、分布和 shuffle；
   - JDBC 读取慢：源数据库负载、查询计划、网络；
   - 写出慢：分区过多、S3 写入、小文件或下游限流。
4. **安全优化**：先做低风险改变，如读取过滤、列裁剪、修正分区和文件压实；再基于瓶颈调整 Join 或资源。
5. **验证**：对同一代表性数据比较运行时长、成本、行数和指标结果，确认未因优化丢失数据。

---

### 45. 场景题：源系统新增了一个字段并将金额从字符串改成 decimal，如何上线？

**推荐回答**

- 先确认字段的业务定义、精度 / 小数位、是否可为空、历史数据处理方式及下游影响。
- 在开发 / 测试环境用真实代表样本验证 schema 解析和数值精度，禁止使用 `double` 处理货币金额。
- 将新字段作为兼容性扩展加入 Bronze / Silver；对旧格式字符串进行显式转换，对无法转换的记录进入 Quarantine。
- 对 Gold、Redshift 表、视图、Power BI Semantic Model 和质量规则逐一评估影响。
- 发布前设定转换失败率阈值、金额控制总额对账和回滚方案。
- 如果是破坏性变更，不直接替换旧字段；可采用版本化列 / 表、兼容视图和消费端迁移期。

---

### 46. 遇到紧急生产故障时，你如何与业务方沟通？

**推荐回答**

沟通要及时、事实驱动且避免过度承诺。首条通知至少包含：

```text
事件：销售日报数据管道延迟
影响：2026-10-28 的 Power BI 销售数据尚未刷新；历史数据不受影响
当前状态：已定位为上游文件未到达，正在与源系统团队确认
临时措施：暂停下游发布，避免展示空数据
下一次更新时间：16:00 HKT
负责人 / 跟踪编号：<on-call owner> / <incident id>
```

恢复后应发送根因、影响窗口、修复、数据验证结论和预防措施。不要在没有证据时说“没有影响”；若仍在调查，应明确写“影响范围正在确认”。

---

### 47. 请举例说明你如何处理与业务方、数据工程师或 BI 开发人员的意见分歧。

**回答结构**

1. 先确认分歧是需求定义、数据口径、实现成本还是上线优先级，而非把问题归因于个人。
2. 用可验证证据对齐，例如字段定义、样本数据、现有报表、数据字典、SLA 和合规要求。
3. 提出选项及其代价：快速临时修复、正确的长期模型、是否需要历史回算。
4. 明确最终决策人和记录方式，例如需求文档 / ticket / PR。
5. 将结论落实到数据契约、测试和文档，避免同一争议重复发生。

在跨地区团队中，英文书面更新应短、清楚、可执行：说明 **issue、impact、action、owner、ETA**。

---

## 十一、面试前速记清单

### 48. 60 秒自我介绍模板

> 我有 ___ 年数据开发 / 数据仓库 / BI 相关经验，主要使用 SQL、Python 和 PySpark 构建和维护数据管道。我的项目经验覆盖从数据源接入、S3 数据分层、AWS Glue ETL、Redshift 数据集，到 Power BI 报表支持的完整链路。  
>  
> 在日常工作中，我不仅关注开发，也关注数据质量、可观测性、幂等重跑和生产问题闭环。例如，我曾负责 ___，通过 ___ 解决了 ___，最终使 ___ 达到 ___。我习惯使用 GitHub PR 和 CI/CD 保障发布质量，也能阅读英文技术文档、与业务和跨团队同事对齐数据口径。这个岗位的 Lakehouse、Glue、SQL / PySpark 和 BI 支持职责与我的经验和兴趣高度匹配。

将空白处替换为真实经历。若经验不足，不要夸大为“主导架构”，可准确表述为“参与开发”“负责某个数据域”“协助排障与测试”。

---

### 49. 常见英文术语与表达

| 中文 | 英文 | 示例表达 |
|---|---|---|
| 数据延迟 | data latency / delay | “The data refresh is delayed because the source file has not arrived.” |
| 数据质量检查 | data quality check | “We validate completeness, uniqueness, and reconciliation totals.” |
| 水位线 | watermark | “The watermark is updated only after the target commit succeeds.” |
| 幂等 | idempotent | “The pipeline is idempotent, so a retry does not duplicate records.” |
| 回补 | backfill | “We will backfill the affected business dates from the raw layer.” |
| 根因分析 | root cause analysis (RCA) | “We will share the RCA and preventive actions after recovery.” |
| 语义模型 | semantic model | “The semantic model centralizes metric definitions and row-level security.” |
| 数据倾斜 | data skew | “A skewed join key caused a few Spark tasks to run much longer.” |
| 变更管理 | change management | “The schema change requires impact assessment and a rollback plan.” |

---

### 50. 面试前最后检查

- [ ] 能在 90 秒内讲清一个真实数据管道：来源、处理、存储、消费、质量、监控。
- [ ] 能解释 Lake、Warehouse、Lakehouse 的差异，不把它们说成互相替代。
- [ ] 能说明 S3 分层、Parquet、分区裁剪、小文件和权限治理。
- [ ] 能解释 Glue Catalog、Crawler、Connection、Job、DynamicFrame / DataFrame。
- [ ] 能用自己的话说明增量、水位线、迟到数据、幂等和回补。
- [ ] 能写出或讲清 `GROUP BY`、`HAVING`、窗口函数、去重和 SCD Type 2。
- [ ] 能说明 Redshift 的 `COPY`、分布与排序的基本取舍。
- [ ] 能给出数据质量、监控、故障排查和 Runbook 的具体做法。
- [ ] 能说明 Power BI Semantic Model、星型模型和刷新模式。
- [ ] 能讲清一次 GitHub PR / CI/CD / 回滚的实际流程。
- [ ] 准备 2--3 个 STAR 案例：开发交付、生产故障、跨团队协作。

---

## 附录：回答技术题的通用公式

面对没有准备过的问题，可按以下顺序组织回答：

1. **先定义目标**：这个设计要保障什么业务结果或 SLA？
2. **说明方案**：选择哪些 AWS 服务、数据模型和处理逻辑？
3. **说明可靠性**：如何做到幂等、可重试、可观测和可回滚？
4. **说明数据正确性**：有哪些质量规则、对账和验收标准？
5. **说明安全和成本**：如何处理权限、敏感数据、扫描量和资源使用？
6. **说明取舍**：什么条件下改用另一种方案？

这样回答能体现数据工程师的系统性思维：不仅会写 SQL 或 PySpark，也能对生产数据产品的正确性、稳定性和可维护性负责。
