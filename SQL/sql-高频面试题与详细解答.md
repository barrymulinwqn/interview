# SQL 高频面试题与详细解答

本文面向 SQL 开发、数据工程和后端岗位面试，覆盖最常出现的查询、索引、事务和并发题。示例默认适用于 PostgreSQL 13+ 与 MySQL 8.0+；两者语法或行为不同处会单独注明。

> 面试回答建议：先说明语义和边界条件，再给 SQL，最后说明索引、并发或数据量放大时的风险。只背出一条能跑的 SQL 通常不够。

## 0. 示例表与约定

```sql
CREATE TABLE customers (
    id           BIGINT PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    email        VARCHAR(255) NOT NULL UNIQUE,
    created_at   TIMESTAMP NOT NULL
);

CREATE TABLE orders (
    id            BIGINT PRIMARY KEY,
    customer_id   BIGINT NOT NULL,
    amount        DECIMAL(12, 2) NOT NULL,
    status        VARCHAR(20) NOT NULL,
    created_at    TIMESTAMP NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE INDEX idx_orders_customer_created
    ON orders (customer_id, created_at);
```

- 金额使用 `DECIMAL` / `NUMERIC` 或以最小货币单位保存的整数，不能使用 `FLOAT` / `DOUBLE`。
- 示例中的时间均指“时间点”；生产系统应统一约定时区。PostgreSQL 优先使用 `timestamptz`，MySQL 需明确 `TIMESTAMP` 与 `DATETIME` 的时区语义。
- SQL 关键字不区分大小写，但对象名、字符串值和排序规则的行为由数据库及配置决定。

---

## 1. SQL 查询基础与聚合

### 1.1 SQL 的逻辑执行顺序是什么？

**答案：** 常见 `SELECT` 的逻辑处理顺序可以理解为：

```text
FROM / JOIN -> ON -> WHERE -> GROUP BY -> HAVING -> SELECT -> DISTINCT -> ORDER BY -> LIMIT / OFFSET
```

因此，`WHERE` 中通常不能直接引用本层 `SELECT` 中刚取的别名，而 `ORDER BY` 可以：

```sql
SELECT
    customer_id,
    SUM(amount) AS total_amount
FROM orders
WHERE status = 'paid'
GROUP BY customer_id
HAVING SUM(amount) >= 1000
ORDER BY total_amount DESC;
```

**易错/易混淆点：**

- 逻辑顺序不是优化器实际执行计划；优化器可能重排等价操作，但不能改变查询语义。
- `WHERE total_amount >= 1000` 是错误的，别名在 `WHERE` 阶段尚未产生。应使用 `HAVING`、重复聚合表达式，或包一层子查询/CTE。
- `LIMIT` 没有配合确定性的 `ORDER BY` 时，返回的“前 N 条”没有稳定业务含义。

### 1.2 `WHERE` 和 `HAVING` 有什么区别？

**答案：** `WHERE` 在分组前筛选行，不能直接使用聚合结果；`HAVING` 在分组后筛选组。

```sql
SELECT customer_id, COUNT(*) AS paid_order_count
FROM orders
WHERE status = 'paid'              -- 先减少参与聚合的明细行
GROUP BY customer_id
HAVING COUNT(*) >= 3;              -- 再筛选聚合后的客户组
```

**易错/易混淆点：**

- 将普通过滤条件写进 `HAVING` 往往仍能执行，但会让数据库先聚合更多数据，语义和性能都不如写在 `WHERE` 清楚。
- `HAVING` 不是 `WHERE` 的替代品。没有 `GROUP BY` 时，它也能用于整个结果集的聚合过滤。
- 不要假设所有数据库都允许在 `HAVING` 中使用 `SELECT` 别名；为可移植性，使用聚合表达式或外层查询。

### 1.3 `COUNT(*)`、`COUNT(1)`、`COUNT(column)` 的区别？

**答案：**

- `COUNT(*)`：统计行数，包含列值为 `NULL` 的行。
- `COUNT(1)`：表达式 `1` 对每一行都非 `NULL`，语义上与 `COUNT(*)` 相同；现代数据库通常性能也没有实质差异。
- `COUNT(column)`：只统计 `column IS NOT NULL` 的行。

```sql
SELECT
    COUNT(*) AS all_rows,
    COUNT(shipping_address) AS rows_with_address,
    COUNT(DISTINCT customer_id) AS distinct_customers
FROM orders;
```

**易错/易混淆点：**

- `COUNT(DISTINCT column)` 会忽略 `NULL`；多列去重的语法与支持程度因数据库而异。
- 在 `LEFT JOIN` 后，`COUNT(right_table.id)` 可统计成功匹配的右表行；写成 `COUNT(*)` 则会把未匹配左表行也算进去。
- 不要以为 `COUNT(1)` 必然更快，这是一种过时的经验说法。

### 1.4 为什么 `NULL = NULL` 不是 `TRUE`？如何正确判断空值？

**答案：** SQL 使用三值逻辑：`TRUE`、`FALSE`、`UNKNOWN`。任何普通比较中只要一边是 `NULL`，结果通常是 `UNKNOWN`；`WHERE` 只保留结果为 `TRUE` 的行。

```sql
SELECT * FROM customers WHERE email IS NULL;
SELECT * FROM customers WHERE email IS NOT NULL;
```

**易错/易混淆点：**

- `email = NULL` 和 `email <> NULL` 都不会得到预期行，必须用 `IS NULL` / `IS NOT NULL`。
- `NULL` 不等于空字符串 `''`，也不等于数值 `0`。
- PostgreSQL 可用 `IS [NOT] DISTINCT FROM` 做空值安全比较；MySQL 可用 `<=>` 做空值安全等于比较。跨库写法通常是显式处理 `NULL`。

### 1.5 `INNER JOIN` 与 `LEFT JOIN` 的差异，以及最常见的陷阱？

**答案：** `INNER JOIN` 只保留两侧连接条件命中的行；`LEFT JOIN` 保留左表全部行，右表未命中时右侧列补为 `NULL`。

```sql
-- 返回所有客户；有已支付订单时才带出订单信息
SELECT c.id, c.name, o.id AS order_id, o.amount
FROM customers AS c
LEFT JOIN orders AS o
    ON o.customer_id = c.id
   AND o.status = 'paid';
```

**易错/易混淆点：**

- 将右表过滤条件放进 `WHERE` 会过滤掉右表为 `NULL` 的行，`LEFT JOIN` 会退化为 `INNER JOIN`：

  ```sql
  -- 这不会保留“没有 paid 订单”的客户
  WHERE o.status = 'paid'
  ```

- 想保留左表全部行时，把右表限定条件放在 `ON`；想只返回有匹配记录的左表行时，使用 `INNER JOIN` 或在 `WHERE` 中明确过滤。
- 一对多连接会放大行数。客户连接订单后，客户列会为每个订单重复；汇总前应先确认粒度。

### 1.6 如何找出“从未下过订单”的客户？`NOT IN`、`NOT EXISTS` 和 `LEFT JOIN` 怎么选？

**推荐答案：** 用 `NOT EXISTS`，语义直接且不受子查询中 `NULL` 的影响。

```sql
SELECT c.id, c.name
FROM customers AS c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.customer_id = c.id
);
```

等价的 `LEFT JOIN` 写法：

```sql
SELECT c.id, c.name
FROM customers AS c
LEFT JOIN orders AS o ON o.customer_id = c.id
WHERE o.id IS NULL;
```

**易错/易混淆点：**

- 避免直接写 `NOT IN (SELECT customer_id FROM orders)`。若子查询结果含一个 `NULL`，`x NOT IN (...)` 的结果可能全是 `UNKNOWN`，最终返回零行。
- `LEFT JOIN ... WHERE o.customer_id IS NULL` 只有在连接字段或被测字段能代表“是否匹配”时才可靠；通常检查右表主键 `o.id IS NULL` 最清晰。
- 大表上为 `orders(customer_id)` 建索引，才能让反连接高效执行。

### 1.7 `EXISTS` 与 `IN` 的区别是什么？

**答案：** 两者都可表达成员关系，现代优化器常能将简单查询优化为类似的半连接计划。选择时优先看语义和 `NULL` 行为。

```sql
-- 是否存在至少一笔已支付订单
SELECT c.id, c.name
FROM customers AS c
WHERE EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.customer_id = c.id
      AND o.status = 'paid'
);
```

**易错/易混淆点：**

- `EXISTS` 只关心是否存在匹配，子查询中 `SELECT 1` 只是习惯写法，选择哪一列无关紧要。
- `IN` 适合小的常量集合或确定非空的单列表结果集；不要依据“外表大就一定 EXISTS 快”这类口诀下结论，应查看实际执行计划。
- `NOT EXISTS` 与 `NOT IN` 的空值行为不同，反向查询优先用 `NOT EXISTS`。

### 1.8 `UNION` 和 `UNION ALL` 有什么区别？

**答案：** `UNION` 会对合并结果去重，`UNION ALL` 会保留全部行。

```sql
SELECT customer_id FROM orders WHERE status = 'paid'
UNION ALL
SELECT customer_id FROM orders WHERE status = 'refunded';
```

**易错/易混淆点：**

- `UNION` 的去重通常需要排序或哈希，数据量大时成本显著。确定不需要去重时使用 `UNION ALL`。
- 两侧查询列数必须相同，且对应列类型必须可兼容；最终列名通常取自第一条查询。
- 单条分支的 `ORDER BY` / `LIMIT` 通常需要子查询包裹；合并结果的排序写在最后。

---

## 2. 窗口函数与经典手写 SQL

### 2.1 如何查询每个客户最近的一笔订单？

**答案：** 使用窗口函数给每个客户分区内的订单编号，取第一名。必须增加唯一列作为并列时间的稳定排序条件。

```sql
WITH ranked_orders AS (
    SELECT
        o.*,
        ROW_NUMBER() OVER (
            PARTITION BY o.customer_id
            ORDER BY o.created_at DESC, o.id DESC
        ) AS row_num
    FROM orders AS o
)
SELECT id, customer_id, amount, status, created_at
FROM ranked_orders
WHERE row_num = 1;
```

**易错/易混淆点：**

- `GROUP BY customer_id, MAX(created_at)` 只能拿到最大时间，不能安全地拿到对应订单的其他列；时间相同还会返回多条或出现错误匹配。
- PostgreSQL 可使用 `DISTINCT ON (customer_id)`，但它是方言特性；面试中用窗口函数可移植性更好。
- 推荐索引：`orders(customer_id, created_at DESC, id DESC)`。是否能完全利用排序方向取决于具体数据库和查询形态，应以 `EXPLAIN` 验证。

### 2.2 `ROW_NUMBER()`、`RANK()`、`DENSE_RANK()` 的区别？

**答案：** 假设分数为 `100, 90, 90, 80`：

| 函数 | 结果 | 适用场景 |
| --- | --- | --- |
| `ROW_NUMBER()` | `1, 2, 3, 4` | 每行唯一编号、去重保留一行 |
| `RANK()` | `1, 2, 2, 4` | 并列时名次跳跃的比赛排名 |
| `DENSE_RANK()` | `1, 2, 2, 3` | 并列时名次连续的业务排名 |

```sql
SELECT
    customer_id,
    amount,
    RANK() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS amount_rank
FROM orders;
```

**易错/易混淆点：**

- 要求“每组前 3 条记录”通常用 `ROW_NUMBER() <= 3`；要求“每组金额排名前 3 名，含并列”应使用 `RANK()` 或 `DENSE_RANK()`，两者是否跳号要先确认。
- 窗口函数在 `WHERE` 后、最终 `ORDER BY` 前计算，不能直接在同一层 `WHERE` 使用，需子查询或 CTE。

### 2.3 如何找出重复数据，并只保留每组最新的一条？

假设 `email` 理应唯一，但历史数据已经重复。先查询重复键：

```sql
SELECT email, COUNT(*) AS duplicate_count
FROM customers
GROUP BY email
HAVING COUNT(*) > 1;
```

再使用窗口函数标记需要删除的数据。以下仅展示选择删除目标，正式执行前必须备份并核对结果：

```sql
WITH marked AS (
    SELECT
        id,
        ROW_NUMBER() OVER (
            PARTITION BY email
            ORDER BY created_at DESC, id DESC
        ) AS row_num
    FROM customers
)
SELECT id
FROM marked
WHERE row_num > 1;
```

**易错/易混淆点：**

- “保留哪一条”必须由业务规则决定，不能武断地保留最小 `id`。可能应保留最新、最早、状态有效或关联数据最多的一条。
- 先处理外键引用、审计和归档需求，再删除；删除后应补上 `UNIQUE` 约束，避免重复继续进入。
- 并发写入时，清理与加唯一约束需要在维护窗口或合适的事务/锁策略下执行，否则可能再次产生重复数据。

### 2.4 如何计算连续登录（或连续下单）天数？

**答案：** 这是经典“岛屿问题”。先按用户和日期去重，再用日期减去行号构造连续段标识。以下为 PostgreSQL 写法：

```sql
WITH daily AS (
    SELECT DISTINCT customer_id, CAST(created_at AS DATE) AS active_date
    FROM orders
), grouped AS (
    SELECT
        customer_id,
        active_date,
        active_date - (ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY active_date
        ))::INTEGER AS group_key
    FROM daily
)
SELECT
    customer_id,
    MIN(active_date) AS start_date,
    MAX(active_date) AS end_date,
    COUNT(*) AS consecutive_days
FROM grouped
GROUP BY customer_id, group_key;
```

MySQL 可将 `active_date - row_number` 改写为 `DATE_SUB(active_date, INTERVAL row_num DAY)`，通常要再包一层 CTE 取得 `row_num`。

**易错/易混淆点：**

- 同一天多次登录必须先去重，否则行号会人为打断连续区间。
- 先明确“连续”的粒度：自然日、工作日、每 24 小时还是指定时区的业务日，结果会不同。
- 对时间列直接套 `DATE(created_at)` 会影响索引使用；大表应按时间范围过滤，或建立表达式索引/生成列。

### 2.5 如何实现安全且稳定的分页？

**答案：** 小页数可用 `LIMIT ... OFFSET ...`；深分页优先使用基于最后一条排序键的游标/键集分页（keyset pagination）。

```sql
-- 第一页
SELECT id, customer_id, amount, created_at
FROM orders
WHERE status = 'paid'
ORDER BY created_at DESC, id DESC
LIMIT 50;

-- 下一页：传入上一页最后一行的 created_at 与 id
SELECT id, customer_id, amount, created_at
FROM orders
WHERE status = 'paid'
  AND (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

**易错/易混淆点：**

- `OFFSET 1000000` 仍需要扫描并丢弃大量行，页码越深越慢；数据变化时还可能漏行或重复。
- 只按 `created_at` 排序不稳定，相同时间的行会在翻页间移动；应加唯一的次级排序键 `id`。
- 为该查询创建匹配访问路径的索引，例如 `orders(status, created_at DESC, id DESC)`；索引顺序应依据过滤条件和排序条件验证。

---

## 3. 索引、执行计划与性能

### 3.1 B+ 树索引为什么有“最左前缀”规则？

**答案：** 联合 B+ 树索引按列顺序排序。例如索引 `(customer_id, status, created_at)` 可高效支持以 `customer_id` 开头的等值/范围过滤，也可在条件合适时利用后续列。

```sql
-- 可有效利用联合索引的前缀
WHERE customer_id = 100 AND status = 'paid'

-- 通常不能按 customer_id 精确定位
WHERE status = 'paid'
```

**易错/易混淆点：**

- 不是“查询条件必须从最左列开始写”，优化器不依赖书写顺序；而是索引键排序决定了能否有效缩小扫描范围。
- 当前导列出现范围条件（如 `customer_id > 100`）时，后续列通常不能继续用于精确缩小同一索引扫描范围，但仍可能参与过滤或覆盖查询。不能简单背成“范围后索引完全失效”。
- 不要为了每种组合都建立索引。每个索引都会增加插入、更新、删除、空间和统计维护成本。

### 3.2 什么是覆盖索引？为什么 `SELECT *` 往往不利于性能？

**答案：** 当查询所需列都可从索引中取得，数据库可能无需再访问表数据页（具体能力取决于引擎、可见性信息和计划），这常称为覆盖索引或索引仅扫描。

```sql
-- 索引 (customer_id, created_at, amount) 可覆盖此查询的可能性较高
SELECT created_at, amount
FROM orders
WHERE customer_id = :customer_id
ORDER BY created_at DESC;
```

**易错/易混淆点：**

- `SELECT *` 会读取不需要的列，增加 I/O、网络传输和应用对象构造成本，也常让索引无法覆盖查询。
- PostgreSQL 的 Index Only Scan 仍可能访问堆表检查行可见性；MySQL InnoDB 二级索引中本身包含主键，但不代表所有查询都无需回表。
- 不要为覆盖而无限追加宽列到索引。宽索引会显著增加写放大和缓存压力。

### 3.3 为什么对列做函数、隐式类型转换会导致索引失效？

**答案：** 传统 B+ 树索引按原始列值排序。若写成 `DATE(created_at) = '2026-01-01'`，数据库难以直接利用普通 `created_at` 索引定位范围。

```sql
-- 不推荐：对索引列施加函数
WHERE DATE(created_at) = DATE '2026-01-01'

-- 推荐：使用半开区间，且避免结束日 23:59:59 的精度问题
WHERE created_at >= TIMESTAMP '2026-01-01 00:00:00'
  AND created_at <  TIMESTAMP '2026-01-02 00:00:00'
```

**易错/易混淆点：**

- 不能绝对说“函数一定不走索引”：表达式索引、函数索引或生成列可以支持相同表达式，但必须与查询表达式匹配。
- 隐式转换也有风险，例如数字列与字符串参数比较、不同字符集/排序规则的比较。应用层参数类型应与列类型一致。
- `LIKE '%keyword'` 因前导通配符通常无法按普通 B+ 树前缀检索；全文搜索、倒排索引或专用搜索服务可能更合适。

### 3.4 如何读 `EXPLAIN` / `EXPLAIN ANALYZE`？

**答案：** 用 `EXPLAIN` 查看优化器计划，用实际执行型 `EXPLAIN ANALYZE` 对比估算与真实运行情况。

```sql
EXPLAIN ANALYZE
SELECT id, amount
FROM orders
WHERE customer_id = 42
  AND created_at >= CURRENT_TIMESTAMP - INTERVAL '30 days';
```

重点观察：

| 指标 | 含义与判断 |
| --- | --- |
| 扫描方式 | 全表扫描不必然错误；高选择性条件仍全扫时再考虑索引或统计信息 |
| `rows` 估算与实际 | 差异很大常提示统计信息过期、列相关性或数据倾斜 |
| `loops` | 内层节点被重复执行次数；嵌套循环在外层行数大时可能放大 |
| 排序/哈希 | 关注是否溢出到磁盘、内存使用及上游数据量 |
| 实际时间 | 找到耗时节点，而不是只看总代价 |

**易错/易混淆点：**

- `EXPLAIN ANALYZE` 会真实执行语句。对 `INSERT`、`UPDATE`、`DELETE` 不可直接在生产执行；应在事务中演练后回滚，或使用安全副本。
- `cost` 是优化器内部估算单位，不是毫秒。应对比 `actual time` 和真实行数。
- 新索引不等于一定被选用；若查询要返回表的大部分数据，顺序扫描可能更快。

### 3.5 为什么不应“给每一列建索引”？

**答案：** 索引加快部分读路径，却会带来额外成本：写入时维护索引、更多磁盘空间、更多缓存竞争、统计信息维护和更多供优化器选择的候选路径。

建立索引前应回答：

1. 哪条高频且慢的 SQL 需要优化？
2. 过滤、连接、排序、分组分别使用哪些列？
3. 条件的选择性如何，返回多少行？
4. 该表写入频率和索引维护成本是否可接受？
5. `EXPLAIN ANALYZE` 是否证明确实改善？

**易错/易混淆点：**

- 低基数列（如只有两种状态）单独建普通索引通常收益有限，但与高选择性列组成联合索引、使用部分索引或在特定访问模式下仍可能有价值。
- 外键列不是所有数据库都会自动建立索引；应检查并按父子表删除/连接路径评估。
- 冗余索引要定期清理，例如已有 `(a, b)` 后，单列 `(a)` 往往可能冗余，但不能不看真实查询就删除。

---

## 4. 事务、隔离级别与锁

### 4.1 ACID 分别是什么？

| 属性 | 含义 | 例子 |
| --- | --- | --- |
| 原子性（Atomicity） | 事务内操作要么全成功，要么全回滚 | 转账扣款和入账不能只做一半 |
| 一致性（Consistency） | 提交前后都满足约束和业务不变量 | 余额不能违反检查约束 |
| 隔离性（Isolation） | 并发事务彼此影响受控 | 一个事务不应读到另一个未提交修改 |
| 持久性（Durability） | 已提交数据在故障后可恢复 | 依赖 WAL/redo log、刷盘与恢复机制 |

**易错/易混淆点：**

- “一致性”主要是应用、约束和事务共同维持的业务正确性，不等同于分布式系统 CAP 中的线性一致性。
- 开启事务不自动保证业务正确；仍需要合理的条件更新、约束、隔离级别和重试机制。

### 4.2 如何避免库存或余额在并发下被扣成负数？

**推荐答案：** 将校验写入原子 `UPDATE` 的 `WHERE` 条件中，并检查受影响行数。

```sql
BEGIN;

UPDATE inventory
SET available_quantity = available_quantity - :quantity
WHERE product_id = :product_id
  AND available_quantity >= :quantity;

-- 应用必须确认受影响行数为 1；为 0 时表示库存不足或商品不存在

INSERT INTO inventory_reservations (product_id, quantity, created_at)
VALUES (:product_id, :quantity, CURRENT_TIMESTAMP);

COMMIT;
```

**易错/易混淆点：**

- 不能先 `SELECT available_quantity`，在应用中判断后再无条件 `UPDATE`；两个并发请求都可能读到同一库存，形成竞态。
- 若后续插入预留记录失败，必须回滚整个事务；只保证扣减语句原子并不够。
- 秒杀等极高冲突场景还要考虑热点、限流、队列、幂等键及分片，不能只依赖数据库锁。

### 4.3 脏读、不可重复读、幻读分别是什么？

| 现象 | 描述 |
| --- | --- |
| 脏读 | 读到了另一个尚未提交、之后可能回滚的数据 |
| 不可重复读 | 同一事务两次读同一行，另一事务提交更新后结果不同 |
| 幻读 | 同一条件两次查询，另一事务提交插入/删除符合条件的行，结果集行数不同 |

**易错/易混淆点：**

- 不同数据库、不同隔离实现对“幻读”的定义和防护方式并不完全相同；不要只背 ANSI 名词而忽略实际产品行为。
- PostgreSQL 的 `READ COMMITTED` 是默认级别，每条语句看到执行开始时已提交的数据；`REPEATABLE READ` 提供事务级一致快照；`SERIALIZABLE` 可能因序列化冲突报错，应用必须重试。
- MySQL InnoDB 默认常为 `REPEATABLE READ`，其 MVCC 和锁定读/next-key lock 行为与 PostgreSQL 不同。面试应主动说明“具体取决于引擎和语句类型”。

### 4.4 `SELECT ... FOR UPDATE` 是做什么的？何时使用？

**答案：** 它对选中的行施加写锁或等价锁定，避免其他事务在当前事务结束前并发修改这些行。适用于必须“读出状态后按状态做多步决策”的场景。

```sql
BEGIN;

SELECT status, amount
FROM orders
WHERE id = :order_id
FOR UPDATE;

-- 校验状态后执行状态迁移和记账
UPDATE orders
SET status = 'paid'
WHERE id = :order_id;

COMMIT;
```

**易错/易混淆点：**

- `FOR UPDATE` 必须处于显式事务中才有意义；自动提交模式下语句结束锁立即释放。
- 锁并非“越多越安全”。长事务、无索引条件和等待外部 RPC 都会扩大锁范围与等待时间。
- 能用单条条件 `UPDATE` 原子完成的操作，通常优于“先锁定查询，再更新”。

### 4.5 什么是死锁？如何降低死锁发生率？

**答案：** 两个或多个事务互相等待对方持有的锁，形成循环等待。数据库通常会检测并回滚其中一个事务，应用应捕获可重试错误并进行有限重试。

降低概率的实践：

1. 多行/多表更新按固定顺序访问资源，例如总是按主键升序锁定账户。
2. 事务只包含必要 SQL，避免在事务中等待用户输入、调用远程服务或做耗时计算。
3. 使用合适索引缩小扫描和加锁范围。
4. 正确处理死锁受害者错误，幂等地重试整个事务。

**易错/易混淆点：**

- 锁等待超时不一定是死锁；死锁是存在循环依赖，二者诊断和错误码可能不同。
- 降低隔离级别或禁用约束并不是通用解法，可能以数据正确性换取表面上的“无死锁”。

### 4.6 什么是乐观锁？版本号更新怎样写？

**答案：** 乐观锁假定冲突较少，不在读取时加锁；更新时用旧版本号作为条件，受影响行数为 0 则表示并发冲突。

```sql
UPDATE orders
SET status = :new_status,
    version = version + 1
WHERE id = :id
  AND version = :expected_version;
```

**易错/易混淆点：**

- 必须检查受影响行数；不检查就没有真正处理并发冲突。
- 受影响行数为 0 还可能是记录不存在、权限/过滤条件不满足，不应一概提示“版本冲突”。
- 乐观锁适合低冲突且可重试/可提示用户刷新的场景；高冲突的库存争抢未必合适。

---

## 5. 数据建模、安全与数据库差异

### 5.1 主键、唯一约束和外键分别解决什么问题？

**答案：**

- `PRIMARY KEY`：唯一标识一行，必须唯一且非空；一个表最多一个主键（可为复合主键）。
- `UNIQUE`：保证候选业务键不重复，空值的允许方式和多空值行为因数据库而异。
- `FOREIGN KEY`：保证引用完整性，避免子表引用不存在的父表记录。

```sql
ALTER TABLE orders
ADD CONSTRAINT orders_customer_fk
FOREIGN KEY (customer_id) REFERENCES customers(id);
```

**易错/易混淆点：**

- 应用层校验不能替代数据库约束；并发请求可轻易绕过“先查后插”的应用逻辑。
- 级联删除不是默认安全选项。`ON DELETE CASCADE` 会递归删除关联数据，必须与业务保留策略一致。
- 逻辑删除表上的唯一性需专门设计。例如“邮箱在未删除记录中唯一”可用 PostgreSQL 部分唯一索引，MySQL 可用生成列等方式实现，不能只在应用层约束。

### 5.2 为什么金额不能使用浮点数？

**答案：** 二进制浮点数不能精确表示多数十进制小数，会出现累计误差。金额应使用定点小数 `DECIMAL(p, s)` / `NUMERIC(p, s)`，或用 `BIGINT` 表示分、厘等最小单位。

```sql
-- 金额上限和小数位清晰的场景
amount DECIMAL(18, 2) NOT NULL

-- 多币种、精度固定且追求计算一致性时可存最小货币单位
amount_cents BIGINT NOT NULL
```

**易错/易混淆点：**

- `DECIMAL` 的精度与舍入策略仍需按货币和税务规则定义，不能只因为“精确”就忽略中间计算和展示舍入。
- `BIGINT` 避免小数误差，但不同币种的小数位数不同，需要额外货币元数据和换算规则。

### 5.3 如何防止 SQL 注入？

**答案：** 始终使用数据库驱动提供的参数化查询（prepared statement / bind variables），将 SQL 结构和用户数据分开。

```sql
-- 占位符形式由驱动决定；不要用字符串拼接用户输入
SELECT id, name
FROM customers
WHERE email = :email;
```

**易错/易混淆点：**

- 参数化只能绑定值，通常不能绑定表名、列名、排序方向等 SQL 标识符。这些动态结构必须通过严格白名单映射生成。
- “手动转义单引号”不是参数化的等价替代，容易遗漏编码、方言和边界情况。
- 最小权限是第二道防线：应用账户不应拥有 `DROP`、建用户或不必要的跨库访问权限。

### 5.4 `DELETE`、`TRUNCATE`、`DROP` 的区别？

| 命令 | 作用 | 常见特点 |
| --- | --- | --- |
| `DELETE` | 删除满足条件的行 | 可带 `WHERE`，通常逐行记录变更，事务与触发器行为因数据库而异 |
| `TRUNCATE` | 快速清空表数据 | 通常不能带 `WHERE`，往往重置自增/身份值，锁和事务行为需看数据库 |
| `DROP TABLE` | 删除表定义及数据 | 对象不再存在，依赖对象可能受影响 |

**易错/易混淆点：**

- 不要笼统断言 “`TRUNCATE` 不能回滚”。PostgreSQL 中它是事务性的；MySQL 中 `TRUNCATE` 通常隐式提交，行为不同。
- 生产删除要先确认范围：先用同条件 `SELECT COUNT(*)` 核对，再在事务和变更流程中执行；超大批量删除还应考虑分批、归档和锁/WAL 压力。

### 5.5 MySQL 和 PostgreSQL 常见语法差异有哪些？

| 需求 | PostgreSQL | MySQL 8.0+ |
| --- | --- | --- |
| 自增列 | `GENERATED ... AS IDENTITY` | `AUTO_INCREMENT` |
| UPSERT | `INSERT ... ON CONFLICT ... DO UPDATE` | `INSERT ... ON DUPLICATE KEY UPDATE` |
| 返回写入行 | `RETURNING` 广泛支持 | `RETURNING` 支持范围较有限，需按版本/语句确认 |
| 布尔类型 | 原生 `boolean` | `BOOLEAN` 是 `TINYINT(1)` 同义形式 |
| 时间函数 | `CURRENT_TIMESTAMP`、`date_trunc` 等 | `NOW()`、`DATE_FORMAT` 等 |
| 标识符引用 | 双引号 `"name"` | 反引号 `` `name` ``（默认模式） |

**易错/易混淆点：**

- 生产代码不要混用方言函数，例如把 `DATE_FORMAT` 原样带入 PostgreSQL。
- MySQL 应启用并遵守 `ONLY_FULL_GROUP_BY`；PostgreSQL 对未分组的非聚合列要求更严格。不要依赖 MySQL 历史宽松行为返回“某一条任意记录”。
- `UPSERT` 的冲突目标、锁语义和受影响行数各有差异；高并发场景要按目标数据库验证。

---

## 6. 面试追问与排查清单

当面试官给出“某条 SQL 很慢”或“线上数据不对”的模糊问题时，可按以下顺序展开：

1. **明确语义与数据量：** 返回多少行、正确结果的粒度、是否有 `NULL`、是否有重复、峰值 QPS 和表大小是多少？
2. **先看证据：** 获取带真实参数的 SQL、执行计划、耗时、扫描行数、锁等待与慢查询记录。
3. **确认访问路径：** 检查过滤、连接、排序和聚合列，判断现有索引是否与访问模式匹配。
4. **避免语义回归：** 优化前后对账，特别检查 `LEFT JOIN`、`NULL`、时区、并列排序和分页边界。
5. **评估写入代价：** 新索引、物化汇总、缓存和分区都会改变写放大、维护复杂度与故障恢复方式。
6. **灰度验证：** 在接近生产的数据分布上执行 `EXPLAIN ANALYZE`，观察长尾、并发和回滚方案，而非只看单次平均耗时。

## 7. 高频结论速记

- `WHERE` 过滤行，`HAVING` 过滤组；普通条件尽量放 `WHERE`。
- 判断空值只能用 `IS NULL` / `IS NOT NULL`；反向子查询优先 `NOT EXISTS`，避开 `NOT IN` 的 `NULL` 陷阱。
- `LEFT JOIN` 右表的保留条件放 `ON`；放进 `WHERE` 常会意外变成内连接。
- 聚合后取整行、每组 Top N、去重保留一条，优先使用窗口函数并明确并列排序规则。
- 深分页用键集分页，排序必须稳定且索引应匹配筛选和排序。
- 避免对索引列做函数和隐式类型转换；是否使用索引由执行计划和选择性决定。
- 并发扣减使用带条件的原子 `UPDATE` 并检查受影响行数；事务要短，死锁要可重试。
- 参数化查询防注入，约束防脏数据，最小权限降低误操作和攻击影响。