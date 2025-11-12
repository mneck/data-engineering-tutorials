# Athena Quick Reference Cheat Sheet

## Essential Commands

### Database Operations
```sql
-- Create database
CREATE DATABASE my_db;

-- Use database
USE my_db;

-- Show databases
SHOW DATABASES;

-- Drop database
DROP DATABASE IF EXISTS my_db CASCADE;
```

---

## Table Operations

### Create External Table (CSV)
```sql
CREATE EXTERNAL TABLE table_name (
  col1 INT,
  col2 STRING,
  col3 DOUBLE
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://bucket/path/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

### Create Partitioned Table
```sql
CREATE EXTERNAL TABLE table_name (
  col1 INT,
  col2 STRING
)
PARTITIONED BY (year STRING, month STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://bucket/path/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

### Create Table As Select (CTAS)
```sql
CREATE TABLE new_table
WITH (
  format = 'PARQUET',
  parquet_compression = 'SNAPPY',
  partitioned_by = ARRAY['year', 'month'],
  external_location = 's3://bucket/path/'
) AS
SELECT col1, col2,
       substr(date_col, 1, 4) AS year,
       substr(date_col, 6, 2) AS month
FROM source_table;
```

### Table Management
```sql
-- Show tables
SHOW TABLES;

-- Describe table
DESCRIBE table_name;

-- Show create statement
SHOW CREATE TABLE table_name;

-- Drop table
DROP TABLE IF EXISTS table_name;
```

---

## Partition Management

### Load Partitions (Auto-discover)
```sql
MSCK REPAIR TABLE table_name;
```

### Add Partition Manually
```sql
ALTER TABLE table_name
ADD PARTITION (year='2025', month='01')
LOCATION 's3://bucket/path/year=2025/month=01/';
```

### Drop Partition
```sql
ALTER TABLE table_name
DROP PARTITION (year='2025', month='01');
```

### Show Partitions
```sql
SHOW PARTITIONS table_name;
```

---

## Query Patterns

### Basic Query
```sql
SELECT col1, col2, col3
FROM table_name
WHERE condition
ORDER BY col1 DESC
LIMIT 100;
```

### Aggregations
```sql
SELECT
  customer,
  COUNT(*) AS tx_count,
  SUM(amount) AS total,
  AVG(amount) AS average,
  MIN(amount) AS min_val,
  MAX(amount) AS max_val
FROM table_name
GROUP BY customer
HAVING COUNT(*) > 5
ORDER BY total DESC;
```

### Joins
```sql
-- Left join
SELECT a.*, b.col
FROM table_a a
LEFT JOIN table_b b ON a.id = b.id;

-- Inner join
SELECT a.*, b.col
FROM table_a a
INNER JOIN table_b b ON a.id = b.id;
```

### Partition Filtering
```sql
-- ALWAYS include partition columns in WHERE
SELECT *
FROM partitioned_table
WHERE year='2025' AND month='01'  -- Partition pruning!
  AND amount > 100;               -- Data filter
```

---

## Common SerDes

### CSV
```sql
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
  'separatorChar' = ',',
  'quoteChar' = '\"',
  'escapeChar' = '\\'
)
```

### JSON
```sql
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
```

### Parquet (via CTAS)
```sql
WITH (format = 'PARQUET', parquet_compression = 'SNAPPY')
```

---

## Date/Time Functions

```sql
-- Extract parts
YEAR(date_col)
MONTH(date_col)
DAY(date_col)

-- Parse string to date
CAST('2025-01-01' AS DATE)

-- Date arithmetic
date_col + INTERVAL '7' DAY
CURRENT_DATE - INTERVAL '30' DAY

-- Date difference
DATE_DIFF('day', start_date, end_date)

-- Format date
DATE_FORMAT(date_col, '%Y-%m-%d')
```

---

## String Functions

```sql
-- Substring
SUBSTR(str, start, length)  -- substr('2025-01-01', 1, 4) → '2025'

-- Concatenate
CONCAT(str1, str2, str3)
str1 || str2  -- Alternative

-- Case conversion
UPPER(str)
LOWER(str)

-- Trim whitespace
TRIM(str)

-- Pattern matching
str LIKE 'pattern%'  -- % = any chars, _ = one char
```

---

## Conditional Logic

```sql
-- CASE statement
CASE
  WHEN condition1 THEN value1
  WHEN condition2 THEN value2
  ELSE default_value
END AS new_column

-- COALESCE (first non-null value)
COALESCE(col1, col2, 'default')

-- NULLIF (null if equal)
NULLIF(col1, col2)
```

---

## Window Functions

```sql
-- Row number
ROW_NUMBER() OVER (PARTITION BY customer ORDER BY date DESC)

-- Rank
RANK() OVER (ORDER BY amount DESC)

-- Running total
SUM(amount) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

---

## Cost Optimization

### Query Cost Formula
```
Cost = (Data Scanned in TB) × $5
```

### Optimization Checklist
✓ Use Parquet/ORC (5-20x less data scanned)
✓ Partition by date/region (10-100x less with filters)
✓ Use compression (SNAPPY, GZIP, ZSTD)
✓ SELECT specific columns, not `SELECT *`
✓ Filter on partition columns
✓ Avoid small files (<128 MB each)

### Compression Options
```sql
-- CTAS with compression
WITH (
  format = 'PARQUET',
  parquet_compression = 'SNAPPY'  -- Options: SNAPPY, GZIP, ZSTD, NONE
)
```

---

## Data Types

| Type | Example | Use Case |
|------|---------|----------|
| INT | 42 | Whole numbers |
| BIGINT | 9223372036854775807 | Large integers |
| DOUBLE | 3.14159 | Decimals |
| STRING | 'text' | Text data |
| DATE | DATE '2025-01-01' | Dates only |
| TIMESTAMP | TIMESTAMP '2025-01-01 10:30:00' | Date + time |
| BOOLEAN | TRUE, FALSE | Binary flags |
| ARRAY | ARRAY[1,2,3] | Lists |
| MAP | MAP('key', 'value') | Key-value pairs |

---

## File Formats

| Format | Type | Compression | Speed | Cost | Use Case |
|--------|------|-------------|-------|------|----------|
| CSV | Row | Poor | Slow | High | Raw ingestion |
| JSON | Row | Poor | Slow | High | Semi-structured |
| Parquet | Columnar | Excellent | Fast | Low | **Production** |
| ORC | Columnar | Excellent | Fast | Low | Production |

---

## Common Errors & Fixes

### "HIVE_CANNOT_OPEN_SPLIT"
**Cause:** S3 path doesn't exist or is empty
**Fix:** Verify path with `aws s3 ls s3://bucket/path/`

### "Access Denied"
**Cause:** Missing IAM permissions
**Fix:** Attach `AmazonS3ReadOnlyAccess` policy

### "Query exhausted resources"
**Cause:** Query result location not set
**Fix:** Settings → Set query result location

### No partitions found
**Cause:** Partition metadata not loaded
**Fix:** Run `MSCK REPAIR TABLE table_name;`

### Wrong data in results
**Cause:** Header row not skipped
**Fix:** Add `TBLPROPERTIES ('skip.header.line.count'='1')`

---

## S3 Path Patterns

### Non-Partitioned
```
s3://bucket/data/
  ├── file1.csv
  ├── file2.csv
  └── file3.csv
```

### Partitioned (Date)
```
s3://bucket/data/
  └── year=2025/
      ├── month=01/
      │   ├── day=01/
      │   │   └── data.csv
      │   └── day=02/
      │       └── data.csv
      └── month=02/
          └── ...
```

### Partitioned (Region)
```
s3://bucket/data/
  ├── region=us-east/
  │   └── year=2025/
  │       └── data.csv
  └── region=eu-west/
      └── year=2025/
          └── data.csv
```

---

## AWS CLI Commands

```bash
# List S3 bucket
aws s3 ls s3://bucket/path/

# Upload file
aws s3 cp local_file.csv s3://bucket/path/

# Upload recursively
aws s3 cp local_dir/ s3://bucket/path/ --recursive

# Download file
aws s3 cp s3://bucket/path/file.csv ./

# Sync directory
aws s3 sync local_dir/ s3://bucket/path/

# Remove file
aws s3 rm s3://bucket/path/file.csv

# Remove recursively
aws s3 rm s3://bucket/path/ --recursive
```

---

## Athena Limits

| Resource | Limit |
|----------|-------|
| Query timeout | 30 minutes |
| Query result size | 1 GB (can increase) |
| DDL query timeout | 600 seconds |
| Concurrent queries | 20 (per workgroup) |
| Max partitions | 1,000,000 |
| Bytes scanned per query | No limit ($ cost increases) |

---

## Best Practices Summary

### Data Organization
1. Partition by frequently filtered columns (date, region)
2. Keep partition sizes 128 MB - 1 GB
3. Use Parquet for production, CSV for raw ingestion
4. Enable compression (SNAPPY default)
5. Avoid small files (compact with Glue)

### Query Writing
1. Always filter on partition columns
2. Use specific column names, not `SELECT *`
3. Limit results during development: `LIMIT 100`
4. Use EXPLAIN to understand query plan
5. Monitor "Data scanned" metric

### Cost Management
1. Set up billing alerts (CloudWatch)
2. Use workgroups to track team costs
3. Archive old data to Glacier
4. Use Glue Crawlers for schema changes
5. Regularly review query patterns

### Security
1. Use IAM policies for access control
2. Enable S3 bucket encryption
3. Use Lake Formation for column-level security
4. Audit queries with CloudTrail
5. Use workgroups to isolate teams

---

## Quick Decision Tree

**When to use Athena:**
- Ad-hoc analysis of S3 data ✓
- Infrequent queries (daily/weekly) ✓
- Data lake analytics ✓
- Serverless requirement ✓

**When NOT to use Athena:**
- Real-time queries (<1s latency) ✗
- Frequent updates/deletes ✗
- Transactional workloads ✗
- Sub-second response times ✗

---

## Resource Links

- [AWS Athena Console](https://console.aws.amazon.com/athena/)
- [Athena Pricing](https://aws.amazon.com/athena/pricing/)
- [SQL Reference](https://docs.aws.amazon.com/athena/latest/ug/ddl-sql-reference.html)
- [SerDe Reference](https://docs.aws.amazon.com/athena/latest/ug/serde-reference.html)

---

**Print this page for quick reference while coding!**
