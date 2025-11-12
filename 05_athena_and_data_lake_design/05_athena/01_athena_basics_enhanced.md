# Athena Basics: Query Data in S3 with SQL

## TL;DR
Amazon Athena lets you run SQL queries directly on files stored in S3 without setting up databases or servers. You only pay for the data scanned (~$5 per TB). Perfect for analyzing logs, CSV exports, or data lake files without moving data around.

---

## The Big Picture: What is Athena and Why Does It Matter?

### What is Athena?
Athena is a **serverless query service** that lets you analyze data in S3 using standard SQL. No infrastructure to manage, no database servers to provision, no data loading required.

### Why it matters in data engineering
**Traditional approach pain points:**
- You have 10 GB of CSV files sitting in S3 from your application logs
- To analyze them, you'd need to: spin up a database (RDS/Redshift), create schemas, load the data (ETL), maintain the database, and pay for constant uptime
- That's overkill if you just want to run a few queries

**Athena solves this:**
- Point it at your S3 files
- Define the schema (column names and types)
- Query immediately with SQL
- Pay only when you query (~$5 per TB scanned)

### When to use Athena
✅ **Good fit:**
- Ad-hoc analysis of S3 data (logs, exports, data lake)
- Infrequent queries on large datasets
- Quick data exploration without infrastructure setup
- Cost-effective for sporadic querying patterns
- Building BI dashboards with tools like QuickSight or Tableau

❌ **Not ideal for:**
- High-frequency, low-latency queries (use RDS/DynamoDB instead)
- Real-time analytics (use Kinesis Analytics)
- When you need to update/delete individual rows (Athena is read-only)
- Sub-second response times for production apps

### Real-world use cases
1. **Log analysis**: Query CloudTrail logs, application logs, or access logs stored in S3
2. **Data lake analytics**: Run SQL on your data lake without loading into a warehouse
3. **Cost optimization**: Analyze AWS billing data exported to S3
4. **Ad-hoc reporting**: Business analysts querying sales data exports
5. **Data validation**: Check data quality after ETL jobs write to S3
6. **Compliance audits**: Query historical data without maintaining expensive databases

---

## Architecture: How Athena Works

```
┌─────────────────┐
│   Your Laptop   │
│  (Athena UI)    │
└────────┬────────┘
         │ SQL Query
         ↓
┌─────────────────────────────┐
│   Amazon Athena             │
│   (Query Engine)            │
│   - Parses SQL              │
│   - Plans execution         │
│   - Distributes work        │
└────────┬────────────────────┘
         │ Reads files
         ↓
┌─────────────────────────────┐
│   Amazon S3                 │
│   └─ my-bucket/             │
│      └─ trips/              │
│         ├─ trips.csv        │
│         └─ more_trips.csv   │
└─────────────────────────────┘
         │
         ↓ Writes results
┌─────────────────────────────┐
│   S3 Query Results Bucket   │
│   (Athena stores results)   │
└─────────────────────────────┘
```

**Key concept: Schema-on-Read**
- Traditional databases: Schema-on-Write (define schema before inserting data)
- Athena: Schema-on-Read (data stays as-is, schema applied only when querying)
- Files in S3 don't change; you just tell Athena how to interpret them

---

## Prerequisites

Before starting this tutorial, make sure you have:

### 1. IAM Permissions
Your IAM user needs these managed policies:
- `AmazonAthenaFullAccess` - to create databases/tables and run queries
- `AmazonS3FullAccess` (or read/write access to your specific buckets)

**How to verify:**
- AWS Console → IAM → Users → [your user] → Permissions tab
- Look for these policies attached directly or via a group

### 2. S3 Bucket
You need at least one S3 bucket:
- For storing data files (e.g., `my-athena-basics-bucket`)
- Athena will also need a location for query results (can be the same bucket)

**How to create if you don't have one:**
```bash
# Via AWS CLI
aws s3 mb s3://my-athena-basics-bucket

# Or use AWS Console → S3 → Create Bucket
```

### 3. Configure Athena Query Results Location
Athena stores query results in S3. You must configure this once:

**Steps:**
1. Open Athena Console
2. Before running your first query, you'll see: "Before you run your first query, you need to set up a query result location in Amazon S3"
3. Click **Settings** (or **Manage** in newer UI)
4. Set **Query result location** to: `s3://my-athena-basics-bucket/query-results/`
5. Click **Save**

**What this does:** Every query you run will save results as CSV files in this location. You can download them or use them in other queries.

---

## Hands-On: Query CSV Files in S3

### Step 1: Prepare Sample Data

We'll use a taxi trips dataset. The sample file is already available at `../05_athena/assets/trips.csv`:

```csv
trip_id,trip_date,vendor_id,passenger_count,trip_distance,fare_amount,payment_type
1,2023-01-01,VTS,2,3.2,12.50,CASH
2,2023-01-01,CMT,1,1.5,8.00,CARD
3,2023-01-02,VTS,3,5.0,18.75,CASH
4,2023-01-02,CMT,2,2.0,9.25,CARD
```

**Upload to S3:**

Option A - AWS Console:
1. Open S3 Console
2. Navigate to your bucket
3. Create folder: `trips/`
4. Upload `trips.csv` into that folder
5. Final location: `s3://my-athena-basics-bucket/trips/trips.csv`

Option B - AWS CLI:
```bash
# Upload the file
aws s3 cp ~/path/to/trips.csv s3://my-athena-basics-bucket/trips/trips.csv

# Verify upload
aws s3 ls s3://my-athena-basics-bucket/trips/
```

**Why we use a folder (`trips/`) not just the file:**
- Athena's `LOCATION` points to a folder, not individual files
- This lets you add more CSV files later (e.g., `trips_2024.csv`) and Athena will query all of them automatically
- This is how data lakes scale: just drop more files in the folder

---

### Step 2: Create a Database

In Athena, a **database** is just a namespace for organizing tables. It doesn't store data—data lives in S3.

**Open Athena Console:**
- AWS Console → Search "Athena" → Open Athena

**Run this SQL:**
```sql
CREATE DATABASE demo_trip_db;
```

**Expected result:** "Query successful" message

**What happened:**
- Created a logical database (metadata only)
- Stored in AWS Glue Data Catalog (Athena's metadata store)
- No files created in S3

---

### Step 3: Create an External Table for CSV

Now we tell Athena how to interpret the CSV files in S3.

**Important:** Select `demo_trip_db` from the database dropdown on the left before running this.

```sql
CREATE EXTERNAL TABLE demo_trip_db.trips_csv (
  trip_id INT,
  trip_date STRING,
  vendor_id STRING,
  passenger_count INT,
  trip_distance DOUBLE,
  fare_amount DOUBLE,
  payment_type STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://my-athena-basics-bucket/trips/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

**Replace:** `my-athena-basics-bucket` with your actual bucket name.

**Understanding each part:**

| Component | What it does |
|-----------|-------------|
| `CREATE EXTERNAL TABLE` | Creates table metadata without moving data (data stays in S3) |
| `trips_csv (...)` | Table name and column definitions (schema) |
| `ROW FORMAT SERDE` | SerDe = Serializer/Deserializer; tells Athena this is CSV format |
| `LOCATION` | S3 folder path where CSV files live (not individual file!) |
| `TBLPROPERTIES` | Skip first row because it's the header line |

**Common data types:**
- `INT` - whole numbers
- `DOUBLE` - decimal numbers
- `STRING` - text (dates as strings are fine for simple filtering)
- `DATE` - proper date type (requires format conversion)
- `TIMESTAMP` - datetime with time component

**After running:** Look in the left sidebar under `demo_trip_db` → Tables → you should see `trips_csv`

---

### Step 4: Query the Table

Now the fun part—run SQL queries just like any database!

#### Query 1: Count total trips
```sql
SELECT COUNT(*) AS total_trips
FROM demo_trip_db.trips_csv;
```

**Expected result:** `4` (or however many rows your CSV has)

**What happened:**
- Athena read the CSV from S3
- Skipped the header row
- Counted all rows
- Returned the result
- Saved result CSV to your query-results S3 location

---

#### Query 2: Total fare per day
```sql
SELECT
    trip_date,
    SUM(fare_amount) AS total_fare,
    COUNT(*) AS num_trips
FROM demo_trip_db.trips_csv
GROUP BY trip_date
ORDER BY trip_date;
```

**Use case:** See daily revenue trends. Typical in business analytics—group by time period, aggregate metrics.

**Expected result:**
```
trip_date    total_fare    num_trips
2023-01-01      20.50           2
2023-01-02      28.00           2
```

---

#### Query 3: Average fare
```sql
SELECT
    ROUND(AVG(fare_amount), 2) AS avg_fare
FROM demo_trip_db.trips_csv;
```

**Use case:** Understand typical transaction values. Key metric for pricing analysis.

---

#### Query 4: Cash trips only
```sql
SELECT *
FROM demo_trip_db.trips_csv
WHERE payment_type = 'CASH';
```

**Use case:** Filter analysis. Useful for understanding payment preferences, fraud detection, or reconciliation with cash registers.

---

#### Query 5: Trips by vendor
```sql
SELECT
    vendor_id,
    COUNT(*) AS trips,
    ROUND(SUM(fare_amount), 2) AS total_revenue
FROM demo_trip_db.trips_csv
GROUP BY vendor_id
ORDER BY trips DESC;
```

**Use case:** Vendor performance comparison. Common in multi-supplier analytics.

---

#### Query 6: High-value trips
```sql
SELECT
    trip_id,
    trip_date,
    fare_amount,
    passenger_count,
    trip_distance
FROM demo_trip_db.trips_csv
WHERE fare_amount > 15.00
ORDER BY fare_amount DESC;
```

**Use case:** Anomaly detection, premium trip analysis, or revenue optimization.

---

## Understanding Costs

**Athena pricing:** $5 per TB of data scanned

**What affects cost:**
1. **File format:** CSV scans entire file; Parquet scans only needed columns (10-100x cheaper)
2. **Partitioning:** Filter by partition = scan only relevant folders (10-100x cheaper)
3. **SELECT columns:** `SELECT *` scans everything; `SELECT column1, column2` can be cheaper with columnar formats
4. **Compression:** Gzip/Snappy reduces data scanned

**Cost example:**
- 1 GB CSV file queried 100 times/month = 100 GB scanned = $0.50/month
- Same data in Parquet + partitioned = 1 GB scanned = $0.005/month

**Check query cost:** After each query, look at the query details:
- "Data scanned: X MB/GB"
- This tells you how much data Athena read

---

## Parquet Format: Why You Should Care

### CSV vs Parquet

**CSV (Row format):**
```
trip_id,fare_amount,trip_date
1,12.50,2023-01-01
2,8.00,2023-01-01
```
- Reads entire row even if you only need `fare_amount`
- Larger file size (no compression)
- Slower queries

**Parquet (Columnar format):**
```
[Column 1: trip_id]    → [1,2,3,4...]
[Column 2: fare_amount]→ [12.50,8.00,18.75...]
[Column 3: trip_date]  → [2023-01-01,2023-01-01...]
```
- Reads only columns you SELECT
- Built-in compression
- 10-100x smaller and faster

### Convert CSV to Parquet

Use CTAS (Create Table As Select) to convert on the fly:

```sql
CREATE TABLE demo_trip_db.trips_parquet
WITH (
  format = 'PARQUET',
  external_location = 's3://my-athena-basics-bucket/trips-parquet/'
) AS
SELECT * FROM demo_trip_db.trips_csv;
```

**What this does:**
1. Reads data from `trips_csv` table (CSV format in S3)
2. Converts to Parquet format
3. Writes new Parquet files to `trips-parquet/` folder in S3
4. Creates a new table `trips_parquet` pointing to that location

**After conversion, compare:**

```sql
-- Query CSV version
SELECT COUNT(*) FROM demo_trip_db.trips_csv;
-- Check "Data scanned" in query details

-- Query Parquet version
SELECT COUNT(*) FROM demo_trip_db.trips_parquet;
-- Check "Data scanned" again - should be much less!
```

**Best practice:** Always convert to Parquet for production workloads. CSV is fine for initial uploads or one-time analysis.

---

## Troubleshooting Common Issues

### Error: "HIVE_CANNOT_OPEN_SPLIT: Error opening Hive split"
**Cause:** S3 path in `LOCATION` doesn't exist or has no files
**Fix:**
- Verify path: `aws s3 ls s3://your-bucket/trips/`
- Make sure files are in the folder (not subfolders)
- Check IAM permissions to read S3

### Error: "Access Denied"
**Cause:** IAM user/role lacks S3 read permissions
**Fix:** Attach `AmazonS3ReadOnlyAccess` or custom policy with `s3:GetObject` on your bucket

### Wrong data returned or empty results
**Causes:**
- Header row not skipped → Add `TBLPROPERTIES ('skip.header.line.count'='1')`
- Wrong delimiter → CSV uses commas; if your file uses tabs/semicolons, specify in SERDE properties
- Wrong data types → Check that column types match your data (e.g., don't use INT for text)

**Debug tip:** Run `SELECT * FROM table LIMIT 10` to see raw results and spot formatting issues.

### Query results not showing
**Cause:** Query results location not configured
**Fix:** Settings → Set query result location to an S3 path you control

---

## Practice Exercises

### Exercise 1: Upload your own CSV
1. Create a simple CSV with 3 columns (id, name, value)
2. Upload to S3
3. Create an Athena table
4. Query it

### Exercise 2: Multi-file queries
1. Upload 3 different CSV files to the same S3 folder (e.g., `trips_jan.csv`, `trips_feb.csv`)
2. Query the table—Athena automatically queries all files
3. Add a 4th file and re-query (no table changes needed!)

### Exercise 3: Data types exploration
1. Create a table with a proper `DATE` column instead of STRING
2. Use date functions: `SELECT YEAR(trip_date), MONTH(trip_date) FROM ...`

### Exercise 4: Cost comparison
1. Query your CSV table and note "Data scanned"
2. Convert to Parquet using CTAS
3. Query the Parquet table with same query
4. Compare data scanned (should be 5-50x less)

---

## Key Takeaways

✓ **Athena = SQL engine for S3** - No servers, no database setup
✓ **Schema-on-Read** - Data stays as-is; schema applied at query time
✓ **Supports multiple formats** - CSV, JSON, Parquet, ORC, Avro
✓ **Pay per query** - ~$5 per TB scanned
✓ **Parquet/ORC = cheaper & faster** - Use columnar formats for production
✓ **Partitioning reduces costs** - More on this in the next section
✓ **Great for ad-hoc analysis** - Not for high-frequency transactional queries

---

## What's Next?

In the next section ([02_Partitioning_&_Schema.md](./02_Partitioning_&_Schema_enhanced.md)), we'll cover:
- How to organize data in S3 folders by date/region/etc (partitioning)
- Reduce query costs by 10-100x with partition pruning
- CTAS (Create Table As Select) for data transformation
- Schema evolution and handling changing data formats

---

## Additional Resources

**AWS Documentation:**
- [Athena User Guide](https://docs.aws.amazon.com/athena/latest/ug/what-is.html)
- [Athena SQL Reference](https://docs.aws.amazon.com/athena/latest/ug/ddl-sql-reference.html)
- [SerDe Reference](https://docs.aws.amazon.com/athena/latest/ug/serde-reference.html)

**Sample Datasets to Practice:**
- [AWS Open Data Registry](https://registry.opendata.aws/)
- [NYC Taxi Trip Data](https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page)
- [Amazon Customer Reviews](https://s3.amazonaws.com/amazon-reviews-pds/readme.html)

**Cost optimization:**
- Use Parquet + compression
- Partition by frequently filtered columns
- Use `SELECT column1, column2` instead of `SELECT *`
- Limit data with WHERE clauses
