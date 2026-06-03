# RDBMS & MySQL — Interview Fundamentals

## Relational Database Concepts

### ACID Properties
| Property | Meaning |
|----------|---------|
| **Atomicity** | A transaction is all-or-nothing |
| **Consistency** | DB moves from one valid state to another |
| **Isolation** | Concurrent transactions behave as if serial |
| **Durability** | Committed transactions survive crashes |

### CAP Theorem
A distributed system can guarantee at most 2 of 3:
- **Consistency** — all nodes see the same data
- **Availability** — every request gets a response
- **Partition Tolerance** — system works despite network splits

Relational databases typically prioritize CP (consistency + partition tolerance).

---

## MySQL Architecture

```
Client
  └── Connection Layer (auth, thread pool)
        └── SQL Layer (parser, optimizer, query cache)
              └── Storage Engine API
                    ├── InnoDB (default — ACID, row-level locking)
                    ├── MyISAM (table-level locking, no transactions)
                    └── Memory / CSV / Archive / ...
```

### InnoDB vs MyISAM
| Feature | InnoDB | MyISAM |
|---------|--------|--------|
| Transactions | Yes | No |
| Foreign keys | Yes | No |
| Locking | Row-level | Table-level |
| Crash recovery | Yes | Manual |
| Full-text search | Yes (5.6+) | Yes |
| Use case | OLTP | Legacy read-heavy |

---

## Data Types

### Numeric
| Type | Range | Notes |
|------|-------|-------|
| `TINYINT` | -128 to 127 | 1 byte |
| `SMALLINT` | -32768 to 32767 | 2 bytes |
| `INT` | -2B to 2B | 4 bytes |
| `BIGINT` | ±9.2×10¹⁸ | 8 bytes |
| `DECIMAL(p,s)` | Exact | Finance/money |
| `FLOAT/DOUBLE` | Approximate | Avoid for money |

### String
| Type | Max Size | Notes |
|------|----------|-------|
| `CHAR(n)` | 255 chars | Fixed length, padded |
| `VARCHAR(n)` | 65535 bytes | Variable length |
| `TEXT` | 65KB | No default, no index (full) |
| `MEDIUMTEXT` | 16MB | |
| `LONGTEXT` | 4GB | |

### Date/Time
| Type | Format | Notes |
|------|--------|-------|
| `DATE` | YYYY-MM-DD | |
| `TIME` | HH:MM:SS | |
| `DATETIME` | YYYY-MM-DD HH:MM:SS | Not timezone-aware |
| `TIMESTAMP` | YYYY-MM-DD HH:MM:SS | UTC stored, local displayed |
| `YEAR` | YYYY | |

---

## SQL Fundamentals

### DDL (Data Definition Language)
```sql
CREATE TABLE orders (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id BIGINT UNSIGNED NOT NULL,
    amount      DECIMAL(15,2) NOT NULL,
    status      ENUM('pending','filled','cancelled') DEFAULT 'pending',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_customer (customer_id),
    INDEX idx_status_created (status, created_at),
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

ALTER TABLE orders ADD COLUMN filled_at TIMESTAMP NULL;
ALTER TABLE orders ADD INDEX idx_filled (filled_at);
ALTER TABLE orders DROP COLUMN filled_at;
DROP TABLE IF EXISTS orders;
TRUNCATE TABLE orders;     -- removes all rows, resets AUTO_INCREMENT
```

### DML (Data Manipulation Language)
```sql
-- INSERT
INSERT INTO orders (customer_id, amount, status) VALUES (1, 10000.00, 'pending');
INSERT INTO orders (customer_id, amount) VALUES (2, 5000.00), (3, 7500.00);

-- INSERT ... ON DUPLICATE KEY UPDATE (upsert)
INSERT INTO rates (pair, rate, updated_at)
VALUES ('GBPUSD', 1.2650, NOW())
ON DUPLICATE KEY UPDATE rate = VALUES(rate), updated_at = VALUES(updated_at);

-- UPDATE
UPDATE orders SET status = 'filled', filled_at = NOW()
WHERE id = 42 AND status = 'pending';

-- DELETE
DELETE FROM orders WHERE status = 'cancelled' AND created_at < NOW() - INTERVAL 90 DAY;

-- SELECT
SELECT
    c.name,
    COUNT(o.id)        AS order_count,
    SUM(o.amount)      AS total_amount,
    AVG(o.amount)      AS avg_amount,
    MAX(o.created_at)  AS last_order
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'filled'
  AND o.created_at >= '2025-01-01'
GROUP BY c.id, c.name
HAVING SUM(o.amount) > 100000
ORDER BY total_amount DESC
LIMIT 10 OFFSET 0;
```

### JOIN Types
```sql
-- INNER JOIN: rows with matching in both tables
-- LEFT JOIN: all left rows + matching right (NULL if no match)
-- RIGHT JOIN: all right rows + matching left
-- FULL OUTER JOIN: all rows from both (MySQL: emulate with UNION)
-- CROSS JOIN: cartesian product
-- SELF JOIN: table joined to itself

-- Emulate FULL OUTER JOIN in MySQL
SELECT * FROM a LEFT JOIN b ON a.id = b.id
UNION ALL
SELECT * FROM a RIGHT JOIN b ON a.id = b.id
WHERE a.id IS NULL;
```

### Subqueries & CTEs
```sql
-- Correlated subquery
SELECT * FROM orders o
WHERE amount > (SELECT AVG(amount) FROM orders WHERE customer_id = o.customer_id);

-- CTE (Common Table Expression)
WITH monthly_totals AS (
    SELECT
        DATE_FORMAT(created_at, '%Y-%m') AS month,
        SUM(amount) AS total
    FROM orders
    WHERE status = 'filled'
    GROUP BY month
),
ranked AS (
    SELECT *, RANK() OVER (ORDER BY total DESC) AS rnk
    FROM monthly_totals
)
SELECT * FROM ranked WHERE rnk <= 3;
```

### Window Functions (MySQL 8.0+)
```sql
SELECT
    customer_id,
    amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at)  AS row_num,
    RANK()       OVER (PARTITION BY customer_id ORDER BY amount DESC) AS amount_rank,
    SUM(amount)  OVER (PARTITION BY customer_id)                      AS customer_total,
    LAG(amount)  OVER (PARTITION BY customer_id ORDER BY created_at)  AS prev_amount,
    LEAD(amount) OVER (PARTITION BY customer_id ORDER BY created_at)  AS next_amount
FROM orders;
```

---

## Indexes

### Types
| Type | Use Case |
|------|---------|
| `PRIMARY KEY` | Clustered index; InnoDB physically orders data by PK |
| `UNIQUE` | Enforce uniqueness; can be NULL (unlike PK) |
| `INDEX` (B-tree) | Range queries, equality, ORDER BY |
| `FULLTEXT` | Text search (`MATCH … AGAINST`) |
| `SPATIAL` | Geographic data |

### Index Design Rules
1. **Index selectivity**: High-cardinality columns benefit most
2. **Composite indexes**: Column order matters — leftmost prefix rule
3. **Covering index**: Index includes all columns needed by query (no table lookup)
4. **Avoid over-indexing**: Every index slows writes and uses space
5. **Index on FK columns**: MySQL does NOT auto-index foreign keys

```sql
-- Composite index: supports (a), (a,b), (a,b,c) but NOT (b) alone
CREATE INDEX idx_status_date ON orders (status, created_at);

-- Covering index example
CREATE INDEX idx_cover ON orders (customer_id, status, amount);
SELECT customer_id, status, SUM(amount) FROM orders
WHERE customer_id = 1 GROUP BY status;   -- uses index only
```

---

## Transactions & Locking

```sql
START TRANSACTION;
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
    UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
    -- Check for errors before committing
COMMIT;
-- or ROLLBACK;

-- Savepoints
SAVEPOINT sp1;
ROLLBACK TO SAVEPOINT sp1;

-- Explicit locking
SELECT * FROM orders WHERE id = 42 FOR UPDATE;          -- exclusive lock
SELECT * FROM orders WHERE id = 42 FOR SHARE;           -- shared lock
```

### Isolation Levels
| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| READ UNCOMMITTED | Yes | Yes | Yes |
| READ COMMITTED | No | Yes | Yes |
| REPEATABLE READ | No | No | Yes* |
| SERIALIZABLE | No | No | No |

*InnoDB uses gap locks to prevent phantoms at REPEATABLE READ.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SHOW VARIABLES LIKE 'transaction_isolation';
```

### Deadlocks
Occur when two transactions each hold a lock the other needs. MySQL detects and rolls back one transaction.
```sql
SHOW ENGINE INNODB STATUS;  -- view latest deadlock
-- Prevention: always lock rows in same order; keep transactions short
```

---

## Performance & Query Optimization

### EXPLAIN
```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 1 AND status = 'filled';
EXPLAIN FORMAT=JSON SELECT ...;
EXPLAIN ANALYZE SELECT ...;   -- MySQL 8.0.18+
```

Key EXPLAIN columns:
| Column | Good | Bad |
|--------|------|-----|
| `type` | `const`, `ref`, `range` | `ALL` (full scan) |
| `key` | shows index used | NULL (no index) |
| `rows` | small number | large number |
| `Extra` | `Using index` (covering) | `Using filesort`, `Using temporary` |

### Slow Query Log
```sql
-- Enable slow query log
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;        -- seconds
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- Analyze with pt-query-digest (Percona Toolkit)
pt-query-digest /var/log/mysql/slow.log
```

### Common Optimization Tips
- Use `LIMIT` to avoid full result scans
- Avoid `SELECT *` — select only needed columns
- Avoid functions on indexed columns in WHERE: `WHERE YEAR(created_at) = 2025` → use range
- Avoid `OR` on different indexed columns — use `UNION`
- Use `COUNT(1)` or `COUNT(id)` instead of `COUNT(*)` if needed (usually equivalent)
- Partition large tables by date range for archival queries

---

## Replication & High Availability

### Replication Modes
| Mode | Description |
|------|-------------|
| Async (default) | Primary commits, replica lags |
| Semi-sync | Primary waits for at least one replica ACK |
| GTID-based | Global Transaction IDs simplify failover |
| Group Replication | Multi-primary, built-in HA |

```sql
-- On replica
SHOW REPLICA STATUS\G   -- (MySQL 8.0+) / SHOW SLAVE STATUS\G
-- Key fields: Seconds_Behind_Source, Last_Error
```

### HA Topologies
- **Primary + Read Replicas**: scale reads, manual failover
- **MySQL InnoDB Cluster**: Group Replication + MySQL Router + MySQL Shell
- **Percona XtraDB Cluster**: synchronous multi-primary (Galera)
- **ProxySQL**: connection pooling + query routing

---

## Administration

### Backup & Restore
```bash
# Logical backup (mysqldump)
mysqldump -u root -p \
  --single-transaction \       # consistent snapshot for InnoDB
  --routines \
  --triggers \
  --set-gtid-purged=OFF \
  mydb > mydb_backup.sql

# Restore
mysql -u root -p mydb < mydb_backup.sql

# Physical backup (Percona XtraBackup — hot backup)
xtrabackup --backup --target-dir=/backup/full
xtrabackup --prepare --target-dir=/backup/full

# Binary log backup (point-in-time recovery)
mysqlbinlog --start-datetime="2025-06-01 00:00:00" \
            --stop-datetime="2025-06-01 12:00:00" \
            /var/lib/mysql/binlog.000001 | mysql -u root -p
```

### Key Configuration (`my.cnf`)
```ini
[mysqld]
innodb_buffer_pool_size     = 8G      # 70-80% of RAM for dedicated MySQL
innodb_log_file_size        = 512M
innodb_flush_log_at_trx_commit = 1    # ACID (2 = better perf, slight risk)
sync_binlog                 = 1       # ACID
max_connections             = 500
slow_query_log              = ON
long_query_time             = 1
```

---

## Common Interview Questions

### Q1: What is the difference between DELETE, TRUNCATE, and DROP?
| | DELETE | TRUNCATE | DROP |
|--|--------|----------|------|
| DML/DDL | DML | DDL | DDL |
| WHERE clause | Yes | No | No |
| Rollback | Yes | No* | No |
| Triggers fired | Yes | No | No |
| Resets AUTO_INCREMENT | No | Yes | N/A |
| Removes structure | No | No | Yes |

### Q2: What is a covering index?
An index that contains all columns referenced by a query, allowing MySQL to satisfy the query entirely from the index without reading the actual table rows.

### Q3: What is the N+1 query problem?
Running 1 query to fetch N records, then N additional queries for related data. Solve with JOINs or batch fetching.

### Q4: Explain MVCC (Multi-Version Concurrency Control)
InnoDB uses MVCC to provide consistent reads without locking. Each row stores version information; readers see a snapshot of data at transaction start time, not blocked by writers.

### Q5: What is the difference between clustered and non-clustered index?
- **Clustered (PK)**: Data rows physically stored in index order; one per table
- **Non-clustered (secondary)**: Separate structure; leaf nodes contain PK value used to look up actual row

### Q6: How would you find the top 3 customers by revenue per month?
```sql
WITH ranked AS (
    SELECT
        DATE_FORMAT(created_at, '%Y-%m') AS month,
        customer_id,
        SUM(amount) AS revenue,
        RANK() OVER (PARTITION BY DATE_FORMAT(created_at,'%Y-%m') ORDER BY SUM(amount) DESC) AS rnk
    FROM orders
    WHERE status = 'filled'
    GROUP BY month, customer_id
)
SELECT * FROM ranked WHERE rnk <= 3;
```

### Q7: What is a foreign key and should you always use them?
FK enforces referential integrity at the DB level. In high-throughput OLTP or sharded systems, FK checks add overhead; teams sometimes enforce integrity at the application layer instead.

### Q8: What is connection pooling?
Maintaining a pool of reusable database connections to avoid the overhead of creating/closing connections per request. Tools: ProxySQL, PgBouncer, HikariCP, DBCP.
