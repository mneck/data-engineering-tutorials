# Partitioning & Schema-on-Read in Athena

## TL;DR
Partitioning = organizing S3 data into folders like `year=2023/month=01/` so Athena only scans relevant files when you filter by those columns. This cuts query costs by 10-100x. Schema-on-Read means data stays as-is in S3; you just define how to interpret it when querying.

---

## The Big Picture: Why Partitioning Matters

### The Problem Without Partitioning

Imagine you have 1 TB of sales data spanning 5 years:
```
s3://my-bucket/sales/all_sales_data.csv  (1 TB, 5 years of data)
```

When you query for January 2024 data:
```sql
SELECT * FROM sales WHERE sale_date = '2024-01-01';
```

**What happens:** Athena scans the ENTIRE 1 TB file to find rows matching that date.
- **Cost:** $5 (1 TB × $5/TB)
- **Time:** Slow—reading 1 TB takes minutes

### The Solution: Partitioning

Reorganize data into folders by date:
```
s3://my-bucket/sales/
    year=2020/
        month=01/ → sales_202001.csv
        month=02/ → sales_202002.csv
        ...
    year=2021/
        ...
    year=2024/
        month=01/ → sales_202401.csv  (only 200 MB)
```

Same query with partition filter:
```sql
SELECT * FROM sales WHERE year='2024' AND month='01';
```

**What happens:** Athena only scans `year=2024/month=01/` folder.
- **Cost:** $0.001 (200 MB × $5/TB)
- **Time:** Fast—reading 200 MB takes seconds

**Result:** 5000x cost reduction, 10-50x speed improvement.

---

## Real-World Use Cases for Partitioning

### 1. Log Analysis
**Scenario:** Analyzing CloudTrail logs or application logs
```
s3://logs/cloudtrail/
    year=2024/
        month=11/
            day=01/ → logs_20241101.json.gz
            day=02/ → logs_20241102.json.gz
```
**Benefit:** Query today's logs without scanning months of historical data.

### 2. Time-Series Data
**Scenario:** IoT sensor data, stock prices, website analytics
```
s3://iot-data/
    year=2024/
        month=11/
            day=10/
                hour=14/ → sensor_readings.parquet
```
**Benefit:** Analyze specific time windows (last hour, yesterday, last week) efficiently.

### 3. Multi-Region Data
**Scenario:** E-commerce sales across regions
```
s3://sales/
    region=us-east/
        year=2024/ → ...
    region=eu-west/
        year=2024/ → ...
    region=ap-south/
        year=2024/ → ...
```
**Benefit:** Regional reports only scan that region's data.

### 4. Multi-Tenant SaaS
**Scenario:** SaaS platform with multiple customers
```
s3://saas-data/
    customer_id=cust001/
        date=2024-11-10/ → ...
    customer_id=cust002/
        date=2024-11-10/ → ...
```
**Benefit:** Per-customer queries are isolated and cheap.

---

## Prerequisites

Before starting this section:

✓ **Completed previous section** - You should have `demo_trip_db.trips_csv` table working
✓ **Query results location configured** - Check Athena Settings
✓ **S3 bucket with write permissions** - For creating partitioned data
✓ **Basic SQL knowledge** - GROUP BY, WHERE clauses

---

## Understanding Partition Column Naming

### The S3 Folder Structure Convention

Athena recognizes partitions by folder names following this pattern:
```
column_name=value/
```

**Example:**
```
s3://my-bucket/trips/
    year=2023/           ← partition column: year, value: 2023
        month=01/        ← partition column: month, value: 01
            data.csv
        month=02/
            data.csv
    year=2024/
        month=01/
            data.csv
```

**Key rules:**
- Folder names MUST use format: `column_name=value`
- Equals sign `=` is required
- Values should not have special characters (use `_` or `-` for spaces)
- Multiple levels = multiple partition columns

**Common patterns:**
- By time: `year=YYYY/month=MM/day=DD/`
- By location: `region=us-east/state=california/`
- By category: `product_type=electronics/brand=apple/`

---

## Hands-On: Create Partitioned Data

### Step 1: Prepare Partitioned CSV Files

We'll create a date-partitioned structure for trip data.

**Option A - Manual Upload (for learning):**

Create these files locally:

**File 1: `trips_jan.csv`** (for January 2023)
```csv
trip_id,trip_date,vendor_id,passenger_count,trip_distance,fare_amount,payment_type
1,2023-01-05,VTS,2,3.2,12.50,CASH
2,2023-01-12,CMT,1,1.5,8.00,CARD
3,2023-01-25,VTS,3,5.0,18.75,CASH
```

**File 2: `trips_feb.csv`** (for February 2023)
```csv
trip_id,trip_date,vendor_id,passenger_count,trip_distance,fare_amount,payment_type
10,2023-02-03,CMT,2,2.0,9.25,CARD
11,2023-02-14,VTS,1,4.5,15.50,CASH
12,2023-02-28,CMT,4,6.2,22.00,CARD
```

**Upload to S3 with partition structure:**

Using AWS Console:
1. Open S3 → your bucket
2. Create folders: `trips-partitioned/year=2023/month=01/`
3. Upload `trips_jan.csv` into `month=01/` folder
4. Create folder: `trips-partitioned/year=2023/month=02/`
5. Upload `trips_feb.csv` into `month=02/` folder

Using AWS CLI:
```bash
# Upload January data
aws s3 cp trips_jan.csv s3://my-athena-basics-bucket/trips-partitioned/year=2023/month=01/trips.csv

# Upload February data
aws s3 cp trips_feb.csv s3://my-athena-basics-bucket/trips-partitioned/year=2023/month=02/trips.csv

# Verify structure
aws s3 ls s3://my-athena-basics-bucket/trips-partitioned/ --recursive
```

**Expected S3 structure:**
```
s3://my-athena-basics-bucket/trips-partitioned/
    year=2023/
        month=01/
            trips.csv
        month=02/
            trips.csv
```

---

### Step 2: Create Partitioned Table

Now create an Athena table that recognizes these partitions:

```sql
CREATE EXTERNAL TABLE demo_trip_db.trips_csv_partitioned (
  trip_id INT,
  trip_date STRING,
  vendor_id STRING,
  passenger_count INT,
  trip_distance DOUBLE,
  fare_amount DOUBLE,
  payment_type STRING
)
PARTITIONED BY (year STRING, month STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://my-athena-basics-bucket/trips-partitioned/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

**Key differences from non-partitioned table:**
- `PARTITIONED BY (year STRING, month STRING)` - declares partition columns
- Partition columns are NOT in the main column list (they're inferred from folder names)
- `LOCATION` points to the root folder (not specific year/month)

---

### Step 3: Load Partitions

After creating the table, Athena doesn't automatically know about the partitions. You must tell it to discover them.

**Method 1: MSCK REPAIR TABLE (automatic discovery)**
```sql
MSCK REPAIR TABLE demo_trip_db.trips_csv_partitioned;
```

**What this does:**
- Scans the S3 location
- Finds all folders matching `column_name=value` pattern
- Registers them as partitions in Glue Data Catalog

**Expected output:**
```
Partitions not in metastore:
  trips_csv_partitioned:year=2023/month=01
  trips_csv_partitioned:year=2023/month=02
Repair: Added 2 partitions to metastore
```

**Method 2: Manual partition addition**
```sql
ALTER TABLE demo_trip_db.trips_csv_partitioned
ADD PARTITION (year='2023', month='01')
LOCATION 's3://my-athena-basics-bucket/trips-partitioned/year=2023/month=01/';

ALTER TABLE demo_trip_db.trips_csv_partitioned
ADD PARTITION (year='2023', month='02')
LOCATION 's3://my-athena-basics-bucket/trips-partitioned/year=2023/month=02/';
```

**When to use manual:**
- When adding new partitions after initial setup
- When partition folders don't follow standard naming (can specify custom locations)
- For programmatic partition management via scripts

---

### Step 4: Verify Partitions

Check that partitions were registered:

```sql
SHOW PARTITIONS demo_trip_db.trips_csv_partitioned;
```

**Expected output:**
```
year=2023/month=01
year=2023/month=02
```

If you don't see partitions, re-run `MSCK REPAIR TABLE`.

---

## Querying Partitioned Data

### The Power of Partition Pruning

**Query 1: All data (scans all partitions)**
```sql
SELECT COUNT(*) AS total_trips
FROM demo_trip_db.trips_csv_partitioned;
```

**What happens:** Athena scans both `month=01` and `month=02` folders.
**Data scanned:** ~Full dataset size
**Cost:** Higher

---

**Query 2: January only (partition pruning)**
```sql
SELECT COUNT(*) AS january_trips
FROM demo_trip_db.trips_csv_partitioned
WHERE year='2023' AND month='01';
```

**What happens:** Athena ONLY scans `year=2023/month=01/` folder. Ignores February.
**Data scanned:** ~50% of dataset (just January files)
**Cost:** Half of Query 1

**Look for this in query details:** "Data scanned: XX KB/MB" - compare both queries!

---

**Query 3: Combine partition filter with data filter**
```sql
SELECT
    trip_id,
    trip_date,
    fare_amount,
    vendor_id
FROM demo_trip_db.trips_csv_partitioned
WHERE year='2023'
  AND month='02'
  AND fare_amount > 15.00
ORDER BY fare_amount DESC;
```

**What happens:**
1. Partition pruning: scans only `month=02` folder
2. Then applies `fare_amount > 15.00` filter to rows

**Best practice:** Always include partition columns in WHERE clause when filtering by date/category/region.

---

**Query 4: Aggregations by partition**
```sql
SELECT
    year,
    month,
    COUNT(*) AS trips,
    ROUND(SUM(fare_amount), 2) AS total_revenue,
    ROUND(AVG(fare_amount), 2) AS avg_fare
FROM demo_trip_db.trips_csv_partitioned
GROUP BY year, month
ORDER BY year, month;
```

**Use case:** Monthly summary reports. Partition columns can be used like regular columns in SELECT.

---

## Schema-on-Read Deep Dive

### What is Schema-on-Read?

**Traditional databases (Schema-on-Write):**
1. Define schema first (CREATE TABLE with constraints)
2. Insert data (data validated and transformed to match schema)
3. Query data (schema already enforced)

**Athena (Schema-on-Read):**
1. Write data to S3 (any format, no validation)
2. Define schema when creating table (just metadata)
3. Query data (schema applied on-the-fly when reading)

### Why It Matters

**Flexibility:**
- Data producers don't need to coordinate with data consumers
- Can have multiple schemas for the same data
- Easy to experiment with different interpretations

**Speed:**
- No ETL loading time
- Data available for querying immediately after upload to S3

**Cost:**
- No compute cost for "loading" data
- Only pay when querying

**Trade-off:**
- No validation at write time (can query invalid data)
- Query-time parsing overhead (minimal with Parquet)

### Example: Multiple Schemas for Same Data

Suppose you have JSON files in S3:
```json
{"user_id": 123, "timestamp": "2023-01-01T10:30:00", "event": "login"}
```

**Schema 1: Simple logging**
```sql
CREATE EXTERNAL TABLE events (
  user_id INT,
  timestamp STRING,
  event STRING
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://my-bucket/events/';
```

**Schema 2: Parse timestamp as proper datetime**
```sql
CREATE EXTERNAL TABLE events_typed (
  user_id INT,
  timestamp TIMESTAMP,
  event STRING
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://my-bucket/events/';
```

Same data, different interpretations—no data movement needed!

---

## CTAS: Create Table As Select

### What is CTAS?

CTAS = **Create Table As Select**

One SQL statement that:
1. Creates a new table
2. Defines its storage format/location/partitions
3. Populates it with query results
4. All in a single atomic operation

### Why Use CTAS?

**1. Format conversion**
Convert CSV → Parquet for better performance:
```sql
CREATE TABLE my_table_parquet
WITH (format='PARQUET') AS
SELECT * FROM my_table_csv;
```

**2. Data transformation**
Clean/filter/aggregate data into optimized tables:
```sql
CREATE TABLE sales_clean
WITH (format='PARQUET') AS
SELECT
  order_id,
  UPPER(TRIM(customer_name)) AS customer_name,  -- clean names
  amount
FROM sales_raw
WHERE amount > 0;  -- filter invalid data
```

**3. Create partitioned tables**
Reorganize non-partitioned data into partitions:
```sql
CREATE TABLE sales_partitioned
WITH (
  format='PARQUET',
  partitioned_by=ARRAY['year','month']
) AS
SELECT
  order_id,
  customer_name,
  amount,
  CAST(YEAR(order_date) AS VARCHAR) AS year,
  CAST(MONTH(order_date) AS VARCHAR) AS month
FROM sales_raw;
```

**4. Deduplication**
Remove duplicates and save cleaned data:
```sql
CREATE TABLE users_deduplicated
WITH (format='PARQUET') AS
SELECT DISTINCT user_id, email, signup_date
FROM users_with_duplicates;
```

---

## Hands-On: Convert to Parquet with Partitions

Let's take our non-partitioned CSV table and create an optimized Parquet partitioned version.

### Step 1: Create Parquet Partitioned Table via CTAS

```sql
CREATE TABLE demo_trip_db.trips_parquet_partitioned
WITH (
  format = 'PARQUET',
  parquet_compression = 'SNAPPY',
  partitioned_by = ARRAY['year', 'month'],
  external_location = 's3://my-athena-basics-bucket/trips-parquet-partitioned/'
) AS
SELECT
  trip_id,
  trip_date,
  vendor_id,
  passenger_count,
  trip_distance,
  fare_amount,
  payment_type,
  CAST(substr(trip_date,1,4) AS VARCHAR) AS year,
  CAST(substr(trip_date,6,2) AS VARCHAR) AS month
FROM demo_trip_db.trips_csv;
```

**Understanding the query:**

| Part | Explanation |
|------|-------------|
| `WITH (...)` | Table properties (storage format, compression, partitions, location) |
| `format = 'PARQUET'` | Use columnar Parquet format |
| `parquet_compression = 'SNAPPY'` | Compression algorithm (options: SNAPPY, GZIP, ZSTD, none) |
| `partitioned_by` | List of columns that will become partition folders |
| `external_location` | S3 path where Parquet files will be written |
| `substr(trip_date,1,4)` | Extract year from date string (first 4 chars) |
| `substr(trip_date,6,2)` | Extract month from date string (chars 6-7) |
| `CAST(... AS VARCHAR)` | Convert to string type (partition columns must be strings) |

**What happens:**
1. Athena reads from `trips_csv` (CSV format)
2. Converts to Parquet format
3. Extracts year/month from trip_date
4. Writes Parquet files to S3 in partition folders:
   ```
   s3://.../trips-parquet-partitioned/
       year=2023/
           month=01/
               <random-file-name>.parquet
           month=02/
               <random-file-name>.parquet
   ```
5. Creates table metadata in Glue Data Catalog

---

### Step 2: Load Partitions

```sql
MSCK REPAIR TABLE demo_trip_db.trips_parquet_partitioned;
```

---

### Step 3: Compare Performance

Run the same query on both tables and compare costs:

**CSV version:**
```sql
SELECT COUNT(*)
FROM demo_trip_db.trips_csv
WHERE substr(trip_date,1,4) = '2023' AND substr(trip_date,6,2) = '01';
```
**Data scanned:** ~Full CSV size (must read all rows to check date)

**Parquet partitioned version:**
```sql
SELECT COUNT(*)
FROM demo_trip_db.trips_parquet_partitioned
WHERE year='2023' AND month='01';
```
**Data scanned:** ~Only January partition, in compressed columnar format

**Expected improvement:** 5-50x less data scanned!

---

## Partition Management Best Practices

### 1. Choose Partition Columns Wisely

**Good partition columns:**
- ✅ Date/time (year, month, day) - most common filter
- ✅ Region/geography (country, state, city)
- ✅ Category (product_type, department)
- ✅ Tenant ID (customer_id in multi-tenant SaaS)

**Bad partition columns:**
- ❌ High-cardinality columns (user_id, transaction_id) - creates millions of tiny folders
- ❌ Columns you never filter by
- ❌ Continuous numeric values (price, age) - partition by ranges instead

**Rule of thumb:**
- Each partition should have at least 128 MB of data (ideally 1+ GB)
- Total partitions < 100,000 per table (Athena limit)
- 1-3 partition levels is ideal (e.g., year/month/day)

---

### 2. Adding New Partitions

When new data arrives, add partitions:

**Option 1: Automatic (MSCK REPAIR)**
- Upload new files to S3 with correct folder structure
- Run `MSCK REPAIR TABLE table_name;`
- Good for batch loads

**Option 2: Manual (ALTER TABLE ADD PARTITION)**
```sql
ALTER TABLE demo_trip_db.trips_parquet_partitioned
ADD PARTITION (year='2024', month='03')
LOCATION 's3://my-athena-basics-bucket/trips-parquet-partitioned/year=2024/month=03/';
```
- Good for real-time/streaming scenarios
- Can be automated via AWS Glue, Lambda, or Step Functions

---

### 3. Dropping Old Partitions

Remove old data you no longer query:

```sql
ALTER TABLE demo_trip_db.trips_parquet_partitioned
DROP PARTITION (year='2020', month='01');
```

**Important:** This only removes the partition metadata (from Glue Catalog), NOT the S3 files.

**To delete files too:**
```bash
aws s3 rm s3://my-athena-basics-bucket/trips-parquet-partitioned/year=2020/month=01/ --recursive
```

---

### 4. Partition Projection (Advanced)

For time-series data, you can have Athena automatically infer partitions without MSCK REPAIR.

**Example: Daily partitions**
```sql
CREATE EXTERNAL TABLE logs (
  message STRING,
  level STRING
)
PARTITIONED BY (dt STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
LOCATION 's3://my-bucket/logs/'
TBLPROPERTIES (
  'projection.enabled' = 'true',
  'projection.dt.type' = 'date',
  'projection.dt.range' = '2023-01-01,NOW',
  'projection.dt.format' = 'yyyy-MM-dd',
  'projection.dt.interval' = '1',
  'projection.dt.interval.unit' = 'DAYS',
  'storage.location.template' = 's3://my-bucket/logs/dt=${dt}'
);
```

**Benefits:**
- No MSCK REPAIR needed
- Automatically queries new partitions as dates progress
- Perfect for logs or time-series data with predictable folder structure

**Limitations:**
- Only works for time-based or integer-range partitions
- Requires consistent naming convention

---

## Troubleshooting Partitioned Tables

### Issue: MSCK REPAIR finds no partitions

**Possible causes:**
1. Folder names don't match `column_name=value` format
   - Wrong: `2023/01/` ❌
   - Right: `year=2023/month=01/` ✅

2. LOCATION path in CREATE TABLE is wrong
   - Run: `aws s3 ls s3://your-bucket/path/` to verify

3. Files are in wrong place (e.g., in year folder instead of month subfolder)

**Fix:** Check S3 structure matches partition schema exactly.

---

### Issue: Query returns no data for specific partition

**Possible causes:**
1. Partition not loaded - run `SHOW PARTITIONS table_name` to check
2. WHERE clause uses wrong data type
   - If partition column is STRING, use: `WHERE year='2023'` (with quotes)
   - Wrong: `WHERE year=2023` (without quotes may not match)

---

### Issue: Query still scans all data despite partition filter

**Cause:** Not filtering on partition columns, or using functions on them.

**Bad (no pruning):**
```sql
WHERE trip_date LIKE '2023-01%'  -- filtering on data column, not partition column
```

**Good (pruning works):**
```sql
WHERE year='2023' AND month='01'  -- filtering on partition columns
```

---

## Cost Optimization Checklist

✅ **Partition by time** (year/month/day) if you have time-series data
✅ **Use Parquet or ORC format** instead of CSV/JSON
✅ **Enable compression** (SNAPPY for balance, GZIP for max compression)
✅ **Always filter on partition columns** in WHERE clause
✅ **Use SELECT column_list** instead of SELECT *
✅ **Avoid small files** - aim for 128 MB+ per file (use Glue ETL to compact)
✅ **Use CTAS** to create optimized tables from raw data
✅ **Drop old partitions** you don't query anymore

**Cost comparison example (1 TB dataset, 1000 queries/month):**

| Setup | Data Scanned/Query | Cost/Month |
|-------|-------------------|-----------|
| CSV, no partitions | 1 TB | $5,000 |
| CSV, partitioned (filter 1 month) | 83 GB | $415 |
| Parquet, partitioned | 8.3 GB | $41.50 |
| Parquet, partitioned, compressed | 2 GB | $10 |

---

## Practice Exercises

### Exercise 1: Create 3-level partitions
Create a table partitioned by year/month/day. Upload sample data to that structure.

### Exercise 2: Partition projection
Set up partition projection for a logs table with daily partitions.

### Exercise 3: CTAS transformation
Create a CTAS query that:
- Converts CSV to Parquet
- Partitions by year and month
- Filters out rows where amount < 0
- Adds a new column: `fare_per_mile = fare_amount / trip_distance`

### Exercise 4: Cost comparison
Run the same query on partitioned vs non-partitioned table. Calculate cost savings based on "Data scanned" metrics.

---

## Key Takeaways

✓ **Partitioning = organizing data in S3 folders** by column values (date, region, etc.)
✓ **Partition pruning = scanning only relevant folders** when filtering by partition columns
✓ **Schema-on-Read = data stays in S3, schema applied at query time** (no ETL loading)
✓ **CTAS = powerful tool for format conversion, transformation, and partitioning**
✓ **Parquet + partitioning = 10-100x cost reduction** compared to CSV
✓ **Always filter on partition columns** to get cost benefits
✓ **MSCK REPAIR TABLE** to discover partitions automatically
✓ **Choose partition columns wisely** - use low-cardinality columns you filter on

---

## What's Next?

In the exercises section ([exercises/exercise.md](./exercises/exercise_enhanced.md)), you'll practice:
- Complex SQL queries on partitioned data
- Joining multiple datasets
- Using Athena with EC2-generated data
- Real-world partition management scenarios

---

## Additional Resources

**AWS Documentation:**
- [Partitioning Data](https://docs.aws.amazon.com/athena/latest/ug/partitions.html)
- [Partition Projection](https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html)
- [CTAS Queries](https://docs.aws.amazon.com/athena/latest/ug/ctas.html)
- [Columnar Storage Formats](https://docs.aws.amazon.com/athena/latest/ug/columnar-storage.html)

**Best Practices:**
- [Top 10 Performance Tuning Tips](https://aws.amazon.com/blogs/big-data/top-10-performance-tuning-tips-for-amazon-athena/)
- [Cost Optimization](https://docs.aws.amazon.com/athena/latest/ug/cost-optimization.html)
