# PostgreSQL 开发、调优、迁移与备份实战指南

本文以 PostgreSQL 日常开发与生产运维为主线，覆盖数据库与对象管理、SQL 编写、慢查询诊断、性能调优、版本/平台迁移、逻辑与物理备份恢复，以及可执行的演练清单。命令默认在 PostgreSQL 服务器或已安装客户端工具的运维主机执行。

> 适用版本：PostgreSQL 13 及以上。不同大版本的参数默认值和可用特性可能不同，生产变更前请以目标版本官方文档和预生产验证结果为准。

## 1. 基础概念与工具

### 1.1 核心对象

| 对象 | 作用 | 常用示例 |
| --- | --- | --- |
| Cluster（实例） | 由一个数据目录管理的一组数据库 | `PGDATA=/var/lib/postgresql/16/main` |
| Database | 逻辑隔离单元，连接时必须指定 | `appdb` |
| Schema | 数据库内的命名空间 | `app`、`audit` |
| Role | 用户和用户组统一模型，可登录、可授权 | `app_rw`、`app_ro` |
| Tablespace | 指定对象所在磁盘位置 | 大表或索引隔离到独立卷 |
| Extension | 扩展功能包 | `pg_stat_statements`、`pgcrypto` |

### 1.2 客户端工具

| 工具 | 典型用途 |
| --- | --- |
| `psql` | 交互式 SQL、脚本执行、元命令 |
| `pg_dump` / `pg_restore` | 单库逻辑备份与恢复 |
| `pg_dumpall` | 全局对象（角色、表空间）和全部数据库逻辑备份 |
| `pg_basebackup` | 物理基础备份、搭建备用库 |
| `pgbackrest` 或 Barman | 生产级物理备份、WAL 归档与时间点恢复 |
| `pg_upgrade` | 大版本原地升级 |
| `pg_repack` | 在线重组表/索引，降低锁表影响 |

建议将数据库版本和客户端版本纳入资产管理。`pg_dump` 可连接更早的服务器版本，但不要用明显旧于服务器的客户端做生产备份；恢复时，目标 PostgreSQL 主版本通常应不低于源版本。

### 1.3 连接与常用 psql 命令

```bash
psql "host=db.example.com port=5432 dbname=appdb user=app_admin sslmode=require"
```

```sql
\l                         -- 列出数据库
\c appdb                   -- 切换数据库
\dn                        -- 列出 schema
\dt app.*                  -- 列出表
\d+ app.orders             -- 查看表、索引、大小等定义
\du+                       -- 查看角色和属性
\x auto                    -- 宽结果自动展开
\timing on                 -- 显示每条语句耗时
\watch 2                   -- 每两秒重复上一条查询
```

避免把口令写在命令行、Shell 历史或脚本中。可使用权限为 `0600` 的 `~/.pgpass`：

```text
# hostname:port:database:username:password
db.example.com:5432:appdb:backup_user:replace-with-secret
```

## 2. 初始化、安全与权限模型

### 2.1 最小权限初始化

不要让应用使用超级用户，也不要让应用角色拥有建库、建角色或复制权限。推荐为所有者、读写应用、只读应用和迁移分别创建角色。

```sql
-- 由管理员执行
CREATE ROLE app_owner NOLOGIN;
CREATE ROLE app_rw LOGIN PASSWORD 'store-outside-sql' NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION;
CREATE ROLE app_ro LOGIN PASSWORD 'store-outside-sql' NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION;
CREATE ROLE app_migrator LOGIN PASSWORD 'store-outside-sql' NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION;

CREATE DATABASE appdb OWNER app_owner;
\c appdb

REVOKE CREATE ON SCHEMA public FROM PUBLIC;
CREATE SCHEMA app AUTHORIZATION app_owner;

GRANT CONNECT ON DATABASE appdb TO app_rw, app_ro, app_migrator;
GRANT USAGE ON SCHEMA app TO app_rw, app_ro, app_migrator;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_rw;
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_ro;
GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA app TO app_rw;

ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app
  GRANT SELECT ON TABLES TO app_ro;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app
  GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO app_rw;
```

迁移工具执行 DDL 时，建议显式设置对象所有者或由 `app_owner` 执行，避免默认权限没有覆盖新对象。生产环境中，数据库管理员、迁移执行者与应用运行身份应相互分离。

### 2.2 网络、认证与加密

1. 在 `postgresql.conf` 中限制 `listen_addresses`，只监听需要的接口。
2. 在安全组、防火墙及 `pg_hba.conf` 三层限制来源网段。
3. 对远程连接使用 TLS，客户端指定 `sslmode=verify-full` 并校验证书。
4. `pg_hba.conf` 优先采用 `scram-sha-256`，避免继续新增 `md5` 认证。
5. 开启操作系统磁盘加密或云盘加密；备份也必须独立加密与访问控制。
6. 角色权限、`pg_hba.conf`、扩展安装和参数变更都应纳入变更审计。

示例 `pg_hba.conf` 规则应从精确规则开始：

```text
hostssl  appdb  app_rw  10.20.30.0/24  scram-sha-256
hostssl  appdb  app_ro  10.20.40.0/24  scram-sha-256
```

修改 `pg_hba.conf` 后可重新加载，无需重启：

```sql
SELECT pg_reload_conf();
```

## 3. 日常开发规范

### 3.1 建表与数据类型

推荐优先使用语义准确、可约束、可索引的数据类型：

| 场景 | 推荐 | 注意事项 |
| --- | --- | --- |
| 主键 | `bigint GENERATED ... AS IDENTITY` 或 UUID | 高并发写入 UUID 可考虑时间有序方案 |
| 金额 | `numeric(p,s)` 或最小货币单位 `bigint` | 不使用 `float` 存金额 |
| 时间点 | `timestamptz` | 统一存储时间点，展示时再转时区 |
| 日期 | `date` | 不以字符串保存日期 |
| 状态枚举 | 小而稳定时 `enum`；变化频繁时字典表/约束 | 枚举值删除和重排成本较高 |
| 半结构化数据 | `jsonb` | 热路径字段应抽出为列并建立索引 |
| 二进制文件 | 对象存储 + URL/元数据 | 不将大文件直接堆入业务表 |

```sql
CREATE TABLE app.orders (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES app.customers(id),
  status text NOT NULL CHECK (status IN ('pending', 'paid', 'cancelled')),
  amount_cents bigint NOT NULL CHECK (amount_cents >= 0),
  metadata jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX orders_customer_created_idx
  ON app.orders (customer_id, created_at DESC);
```

建模原则：

- 为每个表定义主键；关联字段应有外键或经过明确取舍后的完整性保障。
- `NOT NULL`、`CHECK`、`UNIQUE` 是数据质量的第一道防线，不能只依赖应用校验。
- 以业务访问路径设计索引。不要为每一列建索引，索引会增加写放大和维护成本。
- 始终在业务逻辑中明确 schema，或为连接设置固定 `search_path`，避免对象遮蔽风险。

### 3.2 事务、并发与锁

PostgreSQL 使用 MVCC：读通常不阻塞写，写通常不阻塞读；但 DDL、行级锁、长事务和某些维护操作仍会产生等待。

```sql
BEGIN;

-- 对需要串行修改的资源加行锁
SELECT balance_cents
FROM app.accounts
WHERE id = 42
FOR UPDATE;

UPDATE app.accounts
SET balance_cents = balance_cents - 500,
    updated_at = now()
WHERE id = 42 AND balance_cents >= 500;

-- 用受影响行数确认扣款是否成功，再插入流水
INSERT INTO app.account_ledger (account_id, delta_cents, created_at)
VALUES (42, -500, now());

COMMIT;
```

实践要点：

- 事务只包含必要 SQL，不要在事务内调用远程 API、等待人工操作或长时间计算。
- 多表更新时保持统一的访问顺序，降低死锁概率。
- 对批处理采用小批量提交；长事务会阻止旧版本回收，导致表膨胀与复制延迟。
- 应用必须正确重试可恢复错误，例如 `40001`（序列化失败）和 `40P01`（死锁检测）。重试应有上限、退避，并保证操作幂等。
- 为交互型连接设置防护阈值：`statement_timeout`、`lock_timeout`、`idle_in_transaction_session_timeout`。

会话级示例：

```sql
SET statement_timeout = '15s';
SET lock_timeout = '3s';
SET idle_in_transaction_session_timeout = '60s';
```

### 3.3 查询与索引编写

```sql
-- 推荐：列出所需字段，使用绑定参数
SELECT id, status, amount_cents, created_at
FROM app.orders
WHERE customer_id = $1
  AND created_at >= $2
ORDER BY created_at DESC
LIMIT 50;
```

- 禁止在热路径中使用 `SELECT *`，它会扩大网络传输、阻碍覆盖索引并使接口随表结构漂移。
- 索引列顺序通常遵循：等值过滤列在前，范围过滤列在后，再考虑排序和覆盖列。
- 对低选择性字段单独建 B-tree 索引往往无效；应与其他过滤条件组成复合索引，或采用部分索引。
- 对 `jsonb` 中的包含查询可使用 GIN 索引；确认查询操作符与索引操作类相匹配。
- 对 `LIKE 'prefix%'`、函数表达式、大小写不敏感查询，应通过表达式索引或合适操作类验证执行计划。

```sql
-- 仅为活跃订单建立更小的索引
CREATE INDEX CONCURRENTLY orders_open_customer_idx
  ON app.orders (customer_id, created_at DESC)
  WHERE status IN ('pending', 'paid');

-- 大表新增索引请使用 CONCURRENTLY；不能放在显式事务块中
```

### 3.4 安全执行 DDL

常见生产迁移应拆分为可回滚、低锁定的阶段：

1. 先增加可空列或新表，不立刻写入大范围默认值。
2. 应用发布为“双写/兼容读”，开始填充新列。
3. 分批回填历史数据，每批短事务并限速。
4. 创建索引时使用 `CREATE INDEX CONCURRENTLY`。
5. 约束可先以 `NOT VALID` 加入，后续 `VALIDATE CONSTRAINT`，减小阻塞。
6. 观测稳定后再切换读取逻辑，并在下一个发布窗口删除旧字段。

```sql
ALTER TABLE app.orders
  ADD CONSTRAINT orders_amount_nonnegative
  CHECK (amount_cents >= 0) NOT VALID;

ALTER TABLE app.orders
  VALIDATE CONSTRAINT orders_amount_nonnegative;
```

注意：`CREATE INDEX CONCURRENTLY` 和部分维护命令不可在事务块中运行。所有迁移脚本必须说明锁级别、估计耗时、回滚策略与验证 SQL。

## 4. 运行监控与问题诊断

### 4.1 必要扩展与日志

在每个需要分析 SQL 的业务数据库启用 `pg_stat_statements`。它需要将扩展加入 `shared_preload_libraries` 后重启实例，再执行：

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

建议在 `postgresql.conf` 或参数组中设置并按业务调整：

```conf
shared_preload_libraries = 'pg_stat_statements'
track_io_timing = on
log_min_duration_statement = '500ms'
log_checkpoints = on
log_lock_waits = on
deadlock_timeout = '1s'
log_line_prefix = '%m [%p] %u@%d %r '
```

不要在高吞吐生产库长期启用无阈值 SQL 全量日志；应设置慢 SQL 阈值、脱敏管道、保留期限和访问权限。

### 4.2 日常巡检 SQL

```sql
-- 当前连接、事务与等待事件
SELECT pid, usename, datname, application_name, client_addr,
       state, wait_event_type, wait_event,
       now() - query_start AS query_age,
       now() - xact_start AS xact_age,
       left(query, 200) AS query
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
ORDER BY xact_start NULLS LAST, query_start NULLS LAST;
```

```sql
-- 等待锁及阻塞者
SELECT blocked.pid AS blocked_pid,
       blocking.pid AS blocking_pid,
       blocked_activity.usename AS blocked_user,
       blocking_activity.usename AS blocking_user,
       now() - blocked_activity.query_start AS blocked_for,
       left(blocked_activity.query, 120) AS blocked_query,
       left(blocking_activity.query, 120) AS blocking_query
FROM pg_locks blocked
JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked.pid
JOIN pg_locks blocking ON blocking.locktype = blocked.locktype
  AND blocking.database IS NOT DISTINCT FROM blocked.database
  AND blocking.relation IS NOT DISTINCT FROM blocked.relation
  AND blocking.page IS NOT DISTINCT FROM blocked.page
  AND blocking.tuple IS NOT DISTINCT FROM blocked.tuple
  AND blocking.virtualxid IS NOT DISTINCT FROM blocked.virtualxid
  AND blocking.transactionid IS NOT DISTINCT FROM blocked.transactionid
  AND blocking.classid IS NOT DISTINCT FROM blocked.classid
  AND blocking.objid IS NOT DISTINCT FROM blocked.objid
  AND blocking.objsubid IS NOT DISTINCT FROM blocked.objsubid
  AND blocking.pid <> blocked.pid
JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking.pid
WHERE NOT blocked.granted AND blocking.granted;
```

```sql
-- 累计耗时最高的 SQL（按实际列名兼容不同版本）
SELECT queryid, calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       rows,
       left(query, 300) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

```sql
-- 表、索引大小和死元组估计
SELECT schemaname, relname,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       n_live_tup, n_dead_tup,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 30;
```

处理阻塞会话前，先确认业务影响与会话身份；优先终止查询，再谨慎终止连接：

```sql
SELECT pg_cancel_backend(12345);       -- 取消当前查询
SELECT pg_terminate_backend(12345);    -- 断开会话，事务会回滚
```

### 4.3 使用 EXPLAIN 分析查询

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT id, status, created_at
FROM app.orders
WHERE customer_id = 123
  AND created_at >= now() - interval '30 days'
ORDER BY created_at DESC
LIMIT 100;
```

重点比较预估行数与实际行数、扫描方式、循环次数、共享缓冲命中/读取以及排序/哈希是否溢出到磁盘。`EXPLAIN ANALYZE` 会实际执行 SQL；对写操作、锁敏感操作或代价极高的查询，应在副本或预生产环境分析。

常见症状与优先排查方向：

| 症状 | 先查什么 | 常见处理 |
| --- | --- | --- |
| 单条 SQL 很慢 | 执行计划、统计信息、索引、返回行数 | 改写 SQL、补合适索引、`ANALYZE` |
| CPU 高 | Top SQL、并发、函数计算、排序/哈希 | 降低扫描量、加索引、限制并发 |
| I/O 高 | `BUFFERS`、缓存命中、临时文件、检查点 | 优化查询、改善内存/磁盘、平滑检查点 |
| 连接耗尽 | `pg_stat_activity`、连接泄漏 | 使用连接池、设超时、修复泄漏 |
| 表持续变大 | 长事务、死元组、autovacuum | 结束长事务、调优 vacuum、按需重组 |
| 复制延迟 | WAL 生成量、网络、回放速度 | 限制大事务、优化副本资源、扩容 |

## 5. 性能调优方法

### 5.1 调优原则

先确定瓶颈，再改一个变量，再度量验证。不要把互联网参数模板直接复制到生产环境。调优优先级通常为：

1. 修正慢 SQL 和数据模型。
2. 处理锁等待、长事务、连接风暴与不合理批处理。
3. 保证 autovacuum、统计信息、检查点和 WAL 归档健康。
4. 最后根据硬件、并发及负载特征调整实例参数。

变更参数前记录基线：QPS、P95/P99 延迟、CPU、内存、磁盘延迟、缓存命中、WAL 量、复制延迟、活跃连接数。预生产压测后分批上线，并准备回滚参数。

### 5.2 关键参数说明

| 参数 | 作用与建议 |
| --- | --- |
| `shared_buffers` | PostgreSQL 自身缓存。常见起点约为专用数据库主机内存的 20%-25%，不是越大越好。 |
| `effective_cache_size` | 供优化器估计操作系统及 PostgreSQL 可用缓存，通常设置为可用于缓存内存的较大部分。 |
| `work_mem` | 每个排序/哈希节点可用内存，按并发会倍增，不能简单设成总内存的固定比例。 |
| `maintenance_work_mem` | VACUUM、建索引等维护内存；维护窗口可提高，但注意并发任务。 |
| `max_connections` | 连接不是越多越好；通常配合 PgBouncer 限制后端连接。 |
| `random_page_cost` | 与实际存储随机读性能有关；SSD/NVMe 可在压测后适度下调。 |
| `checkpoint_timeout`、`max_wal_size` | 影响检查点频率与写入平滑度；调整时观察崩溃恢复时间和磁盘空间。 |
| `autovacuum_*` | 写多的大表应按表设置更积极的阈值和成本限制。 |
| `wal_compression` | 可减少 WAL 网络/存储开销，代价是额外 CPU。 |

谨慎设置全局 `work_mem`。若有 $N$ 个并发会话、每个查询最多 $M$ 个可并行内存算子，则理论内存上限近似为：

$$
Memory_{upper} \approx N \times M \times work\_mem
$$

实际执行计划可能同时存在多个排序、哈希和并行 worker。更安全的方法是保持全局值保守，仅对已验证的报表会话执行 `SET LOCAL work_mem = '...'`。

### 5.3 Autovacuum 与表膨胀

UPDATE/DELETE 会留下死元组，autovacuum 负责回收可见性并更新统计信息。长事务或遗留复制槽会阻止回收，因此先找原因而不是盲目反复 `VACUUM FULL`。

```sql
-- 找到处于事务中时间过长的会话
SELECT pid, usename, application_name, xact_start,
       now() - xact_start AS transaction_age,
       left(query, 200) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;

-- 针对写入热点表设置更积极的 autovacuum 参数（示例需压测）
ALTER TABLE app.orders SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_analyze_scale_factor = 0.01,
  autovacuum_vacuum_cost_limit = 2000
);
```

维护命令选择：

- `VACUUM (ANALYZE)`：常规维护，通常可在线运行。
- `VACUUM FULL`：重写表并获取强锁，仅在维护窗口且有足够额外空间时使用。
- `REINDEX CONCURRENTLY`：在线重建索引，适用于疑似索引膨胀或损坏后的计划维护。
- `pg_repack`：在线重组大表/索引的常用工具，需要提前评估额外空间和权限。

### 5.4 连接池与高可用边界

应用应使用连接池，短事务后及时归还连接。大量短连接会消耗进程、内存和认证开销。PgBouncer 的 transaction pooling 很适合多数无状态 Web 工作负载，但依赖会话状态的功能（临时表、`SET`、预备语句、监听通知等）需要验证兼容性或使用 session pooling。

高可用不是备份：主备复制可以缩短故障切换时间，却可能复制误删和逻辑错误。必须同时拥有可验证的备份、WAL 归档和恢复流程。

## 6. 备份与恢复

### 6.1 制定恢复目标

先与业务确认：

- RPO（Recovery Point Objective）：最多能丢失多久的数据？
- RTO（Recovery Time Objective）：服务最多可中断多久？
- 恢复范围：整库、单库、单 schema、单表，还是误操作前某一时间点？
- 合规要求：保留周期、跨区域副本、加密、不可篡改存储及恢复审计。

备份成功不等于可恢复。应至少定期执行还原、校验数据量/关键表、运行冒烟查询，并记录实际恢复耗时。

### 6.2 逻辑备份：pg_dump

自定义格式支持并行恢复和按对象选择恢复，适合中小型数据库、对象级恢复和跨平台迁移。

```bash
export PGHOST=db.example.com PGPORT=5432 PGUSER=backup_user PGDATABASE=appdb
pg_dump --format=custom --compress=zstd:6 --verbose \
  --file="/backup/appdb-$(date +%F).dump"
```

恢复到新库：

```bash
createdb -h restore.example.com -U app_owner appdb_restore
pg_restore -h restore.example.com -U app_owner -d appdb_restore \
  --jobs=4 --verbose --exit-on-error /backup/appdb-2026-09-30.dump
```

常用恢复方式：

```bash
# 先查看归档内容和对象名称
pg_restore --list /backup/appdb-2026-09-30.dump

# 仅恢复一个 schema
pg_restore -d appdb_restore --schema=app /backup/appdb-2026-09-30.dump

# 仅恢复数据或仅恢复结构
pg_restore -d appdb_restore --data-only /backup/appdb-2026-09-30.dump
pg_restore -d appdb_restore --schema-only /backup/appdb-2026-09-30.dump
```

角色、表空间等全局对象不包含在单个 `pg_dump` 中，应单独备份：

```bash
pg_dumpall --globals-only --file="/backup/globals-$(date +%F).sql"
```

恢复前应审阅备份中的 `OWNER`、`GRANT`、扩展、表空间路径和外部依赖。跨环境恢复时，通常使用受控角色和 `--no-owner`，然后重新应用目标环境权限。

### 6.3 物理备份、WAL 归档与 PITR

对于较大生产库，使用物理基础备份加连续 WAL 归档实现 PITR（Point-in-Time Recovery）。`pgBackRest` 或 Barman 提供压缩、校验、保留策略、并行、远程仓库和恢复编排，通常优于手工复制数据目录。

核心配置思路：

```conf
# postgresql.conf，示例命令须替换为组织认可的备份工具配置
archive_mode = on
archive_command = 'pgbackrest --stanza=prod archive-push %p'
wal_level = replica
max_wal_senders = 10
```

物理备份只能恢复到兼容的 PostgreSQL 主版本和系统架构环境。PITR 流程为：

1. 获取基础备份并确保其校验成功。
2. 确保从基础备份开始到目标恢复点的每个 WAL 段都可获取。
3. 在隔离目标目录恢复基础备份。
4. 创建 `recovery.signal`，设置 `restore_command` 和 `recovery_target_time`、`recovery_target_lsn` 或 `recovery_target_name`。
5. 启动实例，等待回放完成；验证目标时刻的数据与完整性。
6. 确认无误后再切换流量，保留原实例以便审计或回退。

每个恢复点前可创建命名还原点：

```sql
SELECT pg_create_restore_point('before_billing_batch_20260930');
```

切勿在唯一一份生产数据目录上直接尝试恢复。恢复演练必须在隔离主机、隔离端口和隔离网络中进行。

### 6.4 备份作业检查清单

- 基础备份和 WAL 归档均有独立成功告警，且监控最新成功时间。
- 备份仓库使用加密、最小权限、异地副本和生命周期策略。
- 监控 WAL 归档失败、归档积压、备份容量、校验失败和恢复演练超时。
- 保留周期满足业务与合规；删除策略考虑最长恢复窗口。
- 至少每季度做一次全流程恢复演练；重大架构变更后立即演练。
- 明确误删恢复责任人、审批路径、DNS/连接串切换方式和业务验证清单。

## 7. 数据库迁移

迁移分为逻辑迁移、物理迁移和大版本升级。选择依据是停机窗口、数据量、允许的数据丢失、源/目标版本兼容性和是否需要变更平台架构。

### 7.1 方式选择

| 场景 | 推荐方案 | 主要特点 |
| --- | --- | --- |
| 小中型库、可停机 | `pg_dump` + `pg_restore` | 简单可靠，可跨平台、跨大版本 |
| 大库、短停机 | 逻辑复制 + 最后切换 | 先全量同步，再追增量 |
| 同主版本迁移到新主机/存储 | 物理备份恢复或流复制切换 | 速度快，要求兼容性高 |
| 跨大版本原地升级 | `pg_upgrade` | 停机短，需要完整预检与回滚方案 |
| 云迁移 | 云厂商迁移服务或逻辑复制 | 注意扩展、网络、权限和参数差异 |

### 7.2 pg_dump/pg_restore 离线迁移

1. 盘点版本、编码、locale、扩展、角色、表空间、数据库大小和依赖系统。
2. 冻结 DDL 并安排写入停机窗口。
3. 先导出角色与表空间定义，再导出数据库。
4. 在目标端创建兼容角色、扩展和数据库，执行恢复。
5. 校验对象数量、行数、关键汇总、序列值、权限与应用冒烟测试。
6. 切换连接串，观察后再解除旧库写入限制。

```bash
# 源端：先备份全局对象和业务库
pg_dumpall --globals-only > globals.sql
pg_dump -Fc -f appdb.dump appdb

# 目标端：先审阅 globals.sql，避免直接覆盖目标生产角色
psql -d postgres -f sanitized-globals.sql
createdb -O app_owner appdb
pg_restore -d appdb --jobs=4 --exit-on-error appdb.dump
```

大库导出可使用目录格式并行导出：

```bash
pg_dump --format=directory --jobs=4 --file=appdb-dir appdb
pg_restore --jobs=4 --dbname=appdb appdb-dir
```

### 7.3 逻辑复制低停机迁移

逻辑复制适合将表级数据持续同步到新环境。源端发布（publication）和目标端订阅（subscription）要覆盖所有需要的数据表；DDL、角色、序列状态、大对象及部分扩展对象不会自动完整同步，需单独管理。

```sql
-- 源库
CREATE PUBLICATION app_migration_pub FOR ALL TABLES;

-- 目标库（结构和必要扩展应先创建）
CREATE SUBSCRIPTION app_migration_sub
  CONNECTION 'host=source.example.com port=5432 dbname=appdb user=repl_user sslmode=verify-full'
  PUBLICATION app_migration_pub
  WITH (copy_data = true, create_slot = true);
```

切换步骤：

1. 目标库预先建好 schema、索引、约束、角色与扩展，并验证订阅连接。
2. 监控初始复制及延迟，修复复制冲突或不兼容对象。
3. 提前演练切换与回切，应用支持短暂只读或写入暂停。
4. 切换时停止源端写入，等待订阅追平，核对数据和序列值。
5. 提升目标库为唯一写入端，切换流量，再解除写入。
6. 保留源库只读一段观察期，确认后再清理复制槽和订阅。

复制槽可能因消费者停滞保留大量 WAL，导致源库磁盘耗尽。必须监控复制槽滞后和 `pg_wal` 容量。

### 7.4 大版本升级

大版本升级不能直接替换旧二进制启动旧数据目录。常见方案：

- `pg_upgrade`：停机较短，适合同机或可访问新旧数据目录的环境。
- 逻辑导出恢复：最通用，停机时间取决于数据量与恢复速度。
- 逻辑复制迁移：适合极短业务停机，但实施复杂度更高。

升级前必须在同等数据量或代表性数据的预生产环境完成：

1. 阅读跨版本 release notes 和废弃特性说明。
2. 盘点扩展是否支持新版本，确认二进制和 SQL 扩展升级路径。
3. 执行 `pg_upgrade --check`，清理不兼容对象和遗留预备事务。
4. 完成备份与可恢复性验证，明确回滚是恢复旧实例还是切回旧环境。
5. 升级后执行建议的统计信息刷新脚本，重新评估慢 SQL 与参数。

## 8. 生产变更与应急流程

### 8.1 变更前检查

- 明确变更目的、影响表、预计锁级别、耗时、磁盘空间和回滚步骤。
- 在生产数据规模相近的环境压测 SQL 和迁移脚本。
- 确认无长事务、无异常复制延迟、备份/WAL 归档健康。
- 设置变更会话的 `lock_timeout`、`statement_timeout`，防止无限等待。
- 准备观测面板和业务验收 SQL，并指定执行人和决策人。

### 8.2 变更后验证

```sql
-- 核对表和索引定义
\d+ app.orders

-- 核对数据量（大表可用近似统计或分段校验）
SELECT count(*) FROM app.orders;

-- 核对序列是否落后于最大主键
SELECT pg_get_serial_sequence('app.orders', 'id');
SELECT max(id) FROM app.orders;
```

同时验证应用错误率、连接数、慢 SQL、锁等待、CPU/I/O、复制延迟和业务关键指标。若需要回滚，优先执行预先验证过的兼容性回滚或切回流量；不要在故障压力下临时编写破坏性 SQL。

### 8.3 误删应急原则

1. 立即停止或隔离造成误操作的写入路径，记录精确时间、SQL 和受影响范围。
2. 不要直接在生产继续尝试恢复，先保护现有 WAL、备份及相关日志。
3. 在隔离环境执行 PITR 至误操作前，导出所需行或表。
4. 经数据校验后，以受控脚本将缺失数据导回生产。
5. 复盘根因：权限、审批、SQL 防护、软删除、审计、备份演练或发布流程。

## 9. 可执行的日常清单

### 每日

- 检查实例可用性、连接数、磁盘空间、CPU、内存、I/O 延迟与日志异常。
- 确认备份和 WAL 归档在预期窗口内成功完成。
- 检查复制延迟、复制槽积压、长事务、锁等待与失败任务。
- 查看新增的 Top SQL 和应用错误码趋势。

### 每周

- 审核最大表/索引增长、死元组、autovacuum 活动和临时文件使用。
- 复查慢 SQL，使用执行计划确认优化收益。
- 检查角色、权限、过期账号和高权限操作记录。
- 在非生产环境抽样恢复最近备份，验证数据和对象完整性。

### 每月或每季度

- 演练全库恢复和 PITR，记录实际 RTO/RPO 是否达标。
- 审查容量趋势、索引膨胀、分区策略、连接池配置和参数基线。
- 更新版本补丁计划，验证扩展兼容性和安全公告影响。
- 复盘迁移、故障与变更，更新运行手册和自动化脚本。

## 10. 常用参考命令速查

```bash
# 查看服务版本
psql -d postgres -c 'SHOW server_version;'

# 导出单张表的数据（可用于修复或核对）
pg_dump -d appdb --data-only --table=app.orders -f orders-data.sql

# 执行迁移脚本，遇错立即停止且只在一个事务中提交
psql -v ON_ERROR_STOP=1 --single-transaction -d appdb -f migration.sql

# 检查数据库大小
psql -d appdb -c "SELECT pg_size_pretty(pg_database_size(current_database()));"

# 手动刷新某张表统计信息
psql -d appdb -c 'ANALYZE VERBOSE app.orders;'

# 检查归档恢复所需的 WAL 配置
psql -d postgres -c 'SHOW archive_mode;'
psql -d postgres -c 'SHOW archive_command;'
```

## 11. 上线前最小验收标准

新建或迁移 PostgreSQL 服务至少应满足以下条件：

- 非超级用户应用账号、网络白名单、TLS、SCRAM 认证和密钥托管已经就位。
- 已启用慢 SQL 采集和 `pg_stat_statements`，并接入实例、备份、复制、容量告警。
- 所有核心表都有主键，关键约束、访问索引和迁移规范已审查。
- 已配置并验证逻辑备份或物理备份加 WAL 归档，满足经确认的 RPO/RTO。
- 已在隔离环境成功恢复至少一次，并保留恢复耗时、验证记录和责任人。
- 有针对误删、长事务、锁等待、磁盘满、主库故障和迁移回滚的可操作手册。

PostgreSQL 的稳定运行来自可观测的负载、受控的变更和经常验证的恢复能力。把备份恢复演练、SQL 性能回归和权限审查固化为日常流程，通常比一次性的参数“大调优”更能降低风险。