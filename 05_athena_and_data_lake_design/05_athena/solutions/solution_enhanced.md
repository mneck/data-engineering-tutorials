# Athena Exercises: Complete Solutions with Explanations

## How to Use This Solutions Guide

**Approach:**
1. Try exercises on your own first
2. If stuck, check hints in exercise file
3. Come here only for complete solutions
4. Read explanations to understand the "why" not just the "how"

**Note:** Replace `my_datalake_db` and `your-bucket-name` with your actual database and bucket names.

---

## Section A: SQL Queries in Athena

### Setup: Create Base Table

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS my_datalake_db.sample_sales (
  id INT,
  customer STRING,
  amount INT,
  date STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
  'separatorChar' = ',',
  'quoteChar' = '\"',
  'escapeChar' = '\\'
)
LOCATION 's3://your-bucket-name/data/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

**Verification:**
```sql
SELECT * FROM my_datalake_db.sample_sales LIMIT 5;
```

---

### Exercise 1: Filter by Multiple Conditions

**Solution:**
```sql
SELECT *
FROM my_datalake_db.sample_sales
WHERE customer = 'Alice' AND amount > 200
LIMIT 100;
```

**Explanation:**
- `WHERE` clause filters rows before returning results
- `AND` requires both conditions to be true
- String comparison (`customer = 'Alice'`) is case-sensitive
- Numeric comparison (`amount > 200`) doesn't need quotes
- `LIMIT 100` prevents accidentally returning millions of rows (good practice)

**Alternative approaches:**

Multiple customers:
```sql
WHERE customer IN ('Alice', 'Bob') AND amount > 200
```

Amount range:
```sql
WHERE customer = 'Alice' AND amount BETWEEN 200 AND 500
```

**Common mistakes:**
- Using `=` for strings without quotes: `customer = Alice` ❌ (should be `'Alice'`)
- Using `OR` when you meant `AND`: returns too many rows
- Forgetting LIMIT on large datasets: slow and expensive

---

### Exercise 2: Sort Results by Amount

**Solution:**
```sql
SELECT id, customer, amount, date
FROM my_datalake_db.sample_sales
ORDER BY amount DESC
LIMIT 5;
```

**Explanation:**
- `ORDER BY amount DESC` - sorts from highest to lowest
- `DESC` = descending, `ASC` = ascending (default if omitted)
- `LIMIT 5` - returns only top 5 results
- Selecting specific columns (not `SELECT *`) is a best practice for performance

**Variations:**

Bottom 5 (smallest):
```sql
ORDER BY amount ASC LIMIT 5;
```

Sort by multiple columns:
```sql
ORDER BY customer ASC, amount DESC;
-- First sorts by customer alphabetically, then by amount within each customer
```

Top 5 per customer:
```sql
SELECT customer, amount,
       ROW_NUMBER() OVER (PARTITION BY customer ORDER BY amount DESC) AS rank
FROM my_datalake_db.sample_sales
WHERE rank <= 5;  -- Note: This requires a subquery in practice
```

**Performance note:**
- Sorting scans all data before returning results
- With partitions, filter first: `WHERE year='2025' AND month='01'` before `ORDER BY`
- Athena sorts in distributed fashion but final merge can be slow for huge datasets

---

### Exercise 3: Find Distinct Customers

**Solution:**
```sql
SELECT DISTINCT customer
FROM my_datalake_db.sample_sales
ORDER BY customer;
```

**Explanation:**
- `DISTINCT` removes duplicate rows
- Works on all selected columns (if you select multiple, unique combinations are returned)
- `ORDER BY` makes output readable (alphabetical)

**Count distinct customers:**
```sql
SELECT COUNT(DISTINCT customer) AS unique_customers
FROM my_datalake_db.sample_sales;
```

**Expected result:** `10` (Alice, Bob, Charlie, David, Eve, Frank, Grace, Hannah, Ian, Judy)

**Distinct combinations:**
```sql
-- Unique customer-date pairs (how many days did each customer transact?)
SELECT DISTINCT customer, date
FROM my_datalake_db.sample_sales
ORDER BY customer, date;
```

**Performance note:**
- `DISTINCT` requires sorting/hashing all data
- On large datasets, consider sampling first: `WHERE RAND() < 0.1` for 10% sample

---

### Exercise 4: Group By and Average

**Solution:**
```sql
SELECT customer,
       COUNT(*) AS tx_count,
       ROUND(AVG(amount), 2) AS avg_amount,
       MIN(amount) AS min_amount,
       MAX(amount) AS max_amount
FROM my_datalake_db.sample_sales
GROUP BY customer
ORDER BY avg_amount DESC;
```

**Explanation:**
- `GROUP BY customer` - creates one row per unique customer
- `COUNT(*)` - counts rows per group
- `AVG(amount)` - average amount per group
- `ROUND(..., 2)` - rounds to 2 decimal places for readability
- `ORDER BY avg_amount DESC` - shows highest-value customers first

**Real-world application:**
This query answers:
- Who are our VIP customers? (highest avg_amount)
- Who are frequent buyers? (highest tx_count)
- Who has most variable spending? (max - min)

**Filter aggregates with HAVING:**
```sql
SELECT customer, AVG(amount) AS avg_amount
FROM my_datalake_db.sample_sales
GROUP BY customer
HAVING AVG(amount) > 300  -- Filter groups after aggregation
ORDER BY avg_amount DESC;
```

**HAVING vs WHERE:**
- `WHERE` filters rows BEFORE grouping: `WHERE amount > 100`
- `HAVING` filters groups AFTER aggregation: `HAVING AVG(amount) > 300`

**Common mistakes:**
- Selecting non-aggregated columns not in GROUP BY: `SELECT customer, date, AVG(amount) GROUP BY customer` ❌
- Using WHERE instead of HAVING: `WHERE AVG(amount) > 300` ❌

---

### Exercise 5: Filter by Date Range

**Solution:**
```sql
SELECT *
FROM my_datalake_db.sample_sales
WHERE date BETWEEN '2025-01-01' AND '2025-01-10'
ORDER BY date;
```

**Explanation:**
- `BETWEEN` is inclusive: includes both start and end dates
- Works with strings if format is `YYYY-MM-DD` (lexicographic sorting matches chronological)
- Alternative: `WHERE date >= '2025-01-01' AND date <= '2025-01-10'`

**Get Q1 data (Jan-Mar):**
```sql
WHERE date BETWEEN '2025-01-01' AND '2025-03-31'
```

**Current month (if date is stored as DATE type):**
```sql
WHERE YEAR(date) = YEAR(CURRENT_DATE)
  AND MONTH(date) = MONTH(CURRENT_DATE)
```

**Last 30 days (with DATE type):**
```sql
WHERE date >= CURRENT_DATE - INTERVAL '30' DAY
```

**Performance optimization:**
If your table is partitioned by year/month, use partition columns instead:
```sql
WHERE year='2025' AND month='01'  -- Partition pruning!
```

**Common mistakes:**
- Date format mismatch: `'01/01/2025'` won't work with BETWEEN if stored as `'2025-01-01'`
- Using LIKE for dates: `WHERE date LIKE '2025-01%'` works but less readable than BETWEEN
- Forgetting time component: if timestamps include time, BETWEEN might miss late-day records

---

### Exercise 6: Use LIKE for Pattern Matching

**Solution:**
```sql
SELECT *
FROM my_datalake_db.sample_sales
WHERE customer LIKE 'C%';
```

**Explanation:**
- `LIKE` enables pattern matching with wildcards
- `%` = any number of characters (including zero)
- `_` = exactly one character
- Case-sensitive by default (use `LOWER()` for case-insensitive)

**Pattern examples:**

Starts with C:
```sql
WHERE customer LIKE 'C%'  -- Charlie
```

Ends with 'e':
```sql
WHERE customer LIKE '%e'  -- Alice, Eve, Grace
```

Contains 'ar':
```sql
WHERE customer LIKE '%ar%'  -- Charlie
```

Exactly 3 characters:
```sql
WHERE customer LIKE '___'  -- Bob, Eve, Ian (3-letter names)
```

Second letter is 'a':
```sql
WHERE customer LIKE '_a%'  -- David, Hannah, Ian, Zara
```

**Case-insensitive search:**
```sql
WHERE LOWER(customer) LIKE 'c%'  -- Matches charlie, Charlie, CHARLIE
```

**NOT LIKE (exclusion):**
```sql
WHERE customer NOT LIKE 'C%'  -- All except Charlie
```

**Performance note:**
- Wildcard at the start (`'%alice'`) is slow on large datasets (can't use index)
- Wildcard at the end (`'alice%'`) can be optimized
- For complex text search, consider AWS OpenSearch instead

---

### Exercise 7: Combine Aggregates

**Solution:**
```sql
SELECT customer,
       MIN(amount) AS min_amount,
       MAX(amount) AS max_amount,
       ROUND(AVG(amount), 2) AS avg_amount,
       COUNT(*) AS tx_count,
       SUM(amount) AS total_revenue,
       MAX(amount) - MIN(amount) AS spend_variance
FROM my_datalake_db.sample_sales
GROUP BY customer
ORDER BY total_revenue DESC;
```

**Explanation:**
- Multiple aggregate functions in one query provide comprehensive customer view
- Each aggregate operates on the group (per customer)
- `MAX(amount) - MIN(amount)` calculates variance (spread of spending)
- Useful for customer segmentation, RFM analysis, or executive dashboards

**Real-world dashboard:**
```sql
SELECT
  customer,
  COUNT(*) AS transactions,
  SUM(amount) AS lifetime_value,
  ROUND(AVG(amount), 2) AS avg_order_value,
  MIN(date) AS first_purchase,
  MAX(date) AS last_purchase,
  DATE_DIFF('day', CAST(MAX(date) AS DATE), CURRENT_DATE) AS days_since_last_purchase
FROM my_datalake_db.sample_sales
GROUP BY customer
ORDER BY lifetime_value DESC;
```

**Customer segmentation:**
```sql
SELECT
  CASE
    WHEN AVG(amount) >= 500 THEN 'VIP'
    WHEN AVG(amount) >= 200 THEN 'Premium'
    ELSE 'Standard'
  END AS segment,
  COUNT(DISTINCT customer) AS customer_count,
  ROUND(AVG(AVG(amount)), 2) AS avg_segment_value
FROM my_datalake_db.sample_sales
GROUP BY customer
GROUP BY segment;
```

**Performance tip:**
- Aggregates are computationally expensive on large datasets
- Consider pre-aggregating daily (CTAS materialized view pattern)
- Use partition filters to reduce data scanned

---

## Section B: Optimize with Partitions

### Setup: Upload Partitioned Data

**Upload structure:**
```bash
aws s3 cp s3_partitioned_sales/ s3://your-bucket-name/sales_partitioned/ --recursive
```

**Verify:**
```bash
aws s3 ls s3://your-bucket-name/sales_partitioned/ --recursive
```

Expected output:
```
sales_partitioned/year=2025/month=01/sales_202501.csv
sales_partitioned/year=2025/month=02/sales_202502.csv
...
```

---

### Exercise 8: Create Year-Month-Day Partitions

**Solution:**
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS my_datalake_db.sales_partitioned (
  id INT,
  customer STRING,
  amount INT
)
PARTITIONED BY (year STRING, month STRING, day STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
  'separatorChar' = ','
)
LOCATION 's3://your-bucket-name/sales_partitioned/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

**Explanation:**
- `PARTITIONED BY` declares partition columns - these are NOT in the main column list
- Partition columns must match folder names in S3: `year=2025/month=01/day=15/`
- Partition columns are typically STRING type for flexibility
- `LOCATION` points to root folder, not specific partitions

**Why three levels (year/month/day)?**
- **Year only:** Good for archival data queried annually
- **Year + Month:** Most common (daily/weekly queries on recent months)
- **Year + Month + Day:** For high-volume logs (query specific days)
- **Year + Month + Day + Hour:** For real-time streaming data

**Load partitions:**
```sql
MSCK REPAIR TABLE my_datalake_db.sales_partitioned;
```

**Verify:**
```sql
SHOW PARTITIONS my_datalake_db.sales_partitioned;
```

**Alternative partition strategies:**

By region:
```sql
PARTITIONED BY (region STRING, year STRING, month STRING)
-- S3: region=us-east/year=2025/month=01/
```

By product category:
```sql
PARTITIONED BY (category STRING, subcategory STRING)
-- S3: category=electronics/subcategory=laptops/
```

---

### Exercise 9: Manually Add a Partition

**Solution:**
```sql
ALTER TABLE my_datalake_db.sales_partitioned
ADD PARTITION (year='2025', month='02', day='01')
LOCATION 's3://your-bucket-name/sales_partitioned/year=2025/month=02/day=01/';
```

**Explanation:**
- `ALTER TABLE ... ADD PARTITION` registers partition in Glue Data Catalog
- Partition values must match exactly: `year='2025'` (with quotes because it's STRING)
- `LOCATION` must point to exact partition folder in S3
- Use when MSCK REPAIR is too slow or you need granular control

**Verify partition added:**
```sql
SHOW PARTITIONS my_datalake_db.sales_partitioned;
-- Should include: year=2025/month=02/day=01
```

**Bulk add partitions:**
```sql
ALTER TABLE my_datalake_db.sales_partitioned
ADD PARTITION (year='2025', month='02', day='01') LOCATION 's3://path/...'
    PARTITION (year='2025', month='02', day='02') LOCATION 's3://path/...'
    PARTITION (year='2025', month='02', day='03') LOCATION 's3://path/...';
```

**Automation with AWS Glue:**
Use Glue Python Shell job:
```python
import boto3
glue = boto3.client('glue')

glue.create_partition(
    DatabaseName='my_datalake_db',
    TableName='sales_partitioned',
    PartitionInput={
        'Values': ['2025', '02', '01'],
        'StorageDescriptor': {
            'Location': 's3://your-bucket-name/sales_partitioned/year=2025/month=02/day=01/',
            # ... copy from existing partition's storage descriptor
        }
    }
)
```

**When to use manual ADD PARTITION:**
- Real-time data pipelines (add as data arrives)
- Non-standard partition naming (custom LOCATION)
- Selective partition loading (not all S3 folders)
- Scripted automation (CI/CD, Lambda)

---

### Exercise 10: Drop a Partition

**Solution:**
```sql
ALTER TABLE my_datalake_db.sales_partitioned
DROP PARTITION (year='2025', month='01', day='01');
```

**Explanation:**
- Removes partition metadata from Glue Data Catalog
- ⚠️ **DOES NOT delete files from S3** (data still exists)
- Use when implementing data retention policies or cleaning up test data

**Verify partition dropped:**
```sql
SHOW PARTITIONS my_datalake_db.sales_partitioned;
-- year=2025/month=01/day=01 should be gone
```

**To delete S3 files too:**
```bash
aws s3 rm s3://your-bucket-name/sales_partitioned/year=2025/month=01/day=01/ --recursive
```

**Drop multiple partitions:**
```sql
ALTER TABLE my_datalake_db.sales_partitioned
DROP PARTITION (year='2025', month='01', day='01'),
     PARTITION (year='2025', month='01', day='02'),
     PARTITION (year='2025', month='01', day='03');
```

**Automated data retention:**

AWS Glue job (Python) to drop old partitions:
```python
import boto3
from datetime import datetime, timedelta

glue = boto3.client('glue')
cutoff_date = datetime.now() - timedelta(days=90)  # Keep 90 days

# Get all partitions
response = glue.get_partitions(DatabaseName='my_datalake_db', TableName='sales_partitioned')

for partition in response['Partitions']:
    values = partition['Values']  # ['2025', '01', '15']
    partition_date = datetime(int(values[0]), int(values[1]), int(values[2]))

    if partition_date < cutoff_date:
        glue.delete_partition(
            DatabaseName='my_datalake_db',
            TableName='sales_partitioned',
            PartitionValues=values
        )
        print(f"Dropped partition: {'/'.join(values)}")
```

**S3 Lifecycle policy (complement to DROP PARTITION):**
- Console: S3 → Bucket → Management → Lifecycle rules
- Automatically transition old data to Glacier or delete after N days
- More cost-effective than manual deletion

---

### Exercise 11: Compare Query Cost

**Solution:**

**Query 1: Without partition filter (scans all data)**
```sql
SELECT COUNT(*) AS total_transactions
FROM my_datalake_db.sales_partitioned;
```

**Check:** Note "Data scanned" in query results (e.g., 500 KB)

---

**Query 2: With partition filter (scans only January)**
```sql
SELECT COUNT(*) AS january_transactions
FROM my_datalake_db.sales_partitioned
WHERE year='2025' AND month='01';
```

**Check:** Note "Data scanned" (e.g., 80 KB)

---

**Comparison:**

| Metric | Without Filter | With Filter | Improvement |
|--------|---------------|-------------|-------------|
| Data scanned | 500 KB | 80 KB | 6.25x less |
| Query time | 2.5s | 0.4s | 6x faster |
| Cost | $0.0000025 | $0.0000004 | 83% savings |

**Cost calculation:**
- Athena pricing: $5 per TB scanned = $0.000005 per MB
- 500 KB = 0.5 MB → $0.000005 × 0.5 = $0.0000025
- With 1000 queries/month: $2.50 vs $0.40 → **saves $2.10/month**

**Scaled to production:**
- 1 TB dataset, 1000 queries/month, filtering 1 month of data (1/12th)
- Without partitions: 1 TB × 1000 × $5/TB = **$5,000/month**
- With partitions: 83 GB × 1000 × $5/TB = **$415/month**
- **Savings: $4,585/month** or **$55,000/year**!

**Real-world example:**
A data engineering team at a SaaS company reduced Athena costs from $12,000/month to $800/month by:
1. Partitioning by date (year/month/day)
2. Converting CSV to Parquet
3. Enforcing partition filters in BI tool queries

---

### Exercise 12: Convert to Parquet with Compression

**Solution:**
```sql
CREATE TABLE my_datalake_db.sales_parquet_partitioned
WITH (
  format = 'PARQUET',
  parquet_compression = 'SNAPPY',
  partitioned_by = ARRAY['year', 'month'],
  external_location = 's3://your-bucket-name/sales_parquet_partitioned/'
) AS
SELECT
  id,
  customer,
  amount,
  substr(date, 1, 4) AS year,
  substr(date, 6, 2) AS month
FROM my_datalake_db.sample_sales;
```

**Explanation:**
- `format = 'PARQUET'` - columnar storage format (fast + compressed)
- `parquet_compression = 'SNAPPY'` - balanced speed/compression
- `partitioned_by = ARRAY['year','month']` - creates folder structure
- `substr(date, 1, 4)` - extracts year from date string
- `CAST(...  AS VARCHAR)` - ensures partition columns are strings

**After CTAS completes:**
```sql
-- Load partitions
MSCK REPAIR TABLE my_datalake_db.sales_parquet_partitioned;

-- Verify count
SELECT COUNT(*) FROM my_datalake_db.sales_parquet_partitioned;

-- Check partitions
SHOW PARTITIONS my_datalake_db.sales_parquet_partitioned;
```

**Compare formats:**

```sql
-- CSV query (baseline)
SELECT AVG(amount) AS avg_amount
FROM my_datalake_db.sample_sales
WHERE substr(date, 1, 7) = '2025-01';
```
Note "Data scanned" → e.g., **120 KB**

```sql
-- Parquet query (optimized)
SELECT AVG(amount) AS avg_amount
FROM my_datalake_db.sales_parquet_partitioned
WHERE year='2025' AND month='01';
```
Note "Data scanned" → e.g., **15 KB** (**8x less!**)

**Compression options:**

| Compression | Speed | Size | Use Case |
|------------|-------|------|----------|
| NONE | Fastest | Largest | Temporary data, debugging |
| SNAPPY | Fast | Balanced | **Recommended default** |
| GZIP | Slower | Smaller | Archival, infrequent access |
| ZSTD | Balanced | Best | Best compression (newer) |

**Choose compression:**
```sql
-- GZIP (max compression)
WITH (format='PARQUET', parquet_compression='GZIP') AS ...

-- ZSTD (best overall)
WITH (format='PARQUET', parquet_compression='ZSTD') AS ...
```

**File size comparison (1 GB CSV dataset):**
- CSV: 1000 MB
- Parquet + SNAPPY: 150 MB (6.6x smaller)
- Parquet + GZIP: 100 MB (10x smaller)
- Parquet + ZSTD: 90 MB (11x smaller)

**Best practices:**
✓ Always use Parquet for production tables
✓ Use SNAPPY for frequent queries (speed)
✓ Use GZIP/ZSTD for cold storage (space)
✓ Run CTAS nightly to convert incoming CSV data
✓ Delete CSV files after conversion to save storage costs

---

### Exercise 13: Show All Partitions

**Solution:**
```sql
SHOW PARTITIONS my_datalake_db.sales_parquet_partitioned;
```

**Expected output:**
```
year=2025/month=01
year=2025/month=02
year=2025/month=03
year=2025/month=04
year=2025/month=05
year=2025/month=06
```

**Explanation:**
- Lists all registered partitions in Glue Data Catalog
- Useful for debugging "partition not found" errors
- Verify partitions were added after MSCK REPAIR or manual ADD

**Count partitions:**
```sql
SELECT COUNT(*) AS partition_count
FROM (SHOW PARTITIONS my_datalake_db.sales_parquet_partitioned);
-- Note: Subquery syntax may not work in all Athena versions
-- Alternative: Copy output to spreadsheet and count rows
```

**Alternative: Query system table (Athena v3):**
```sql
SELECT *
FROM "information_schema"."__internal_partitions__"
WHERE table_name = 'sales_parquet_partitioned';
```

**Check partition details (Glue API):**
```bash
aws glue get-partitions \
  --database-name my_datalake_db \
  --table-name sales_parquet_partitioned \
  --output json
```

Returns detailed metadata including:
- Partition values
- S3 location
- Row count (if available)
- Last update time

**Debugging workflow:**

1. **Issue:** Query returns no data
2. **Check partitions:**
   ```sql
   SHOW PARTITIONS my_datalake_db.sales_partitioned;
   ```
3. **If empty → Run:**
   ```sql
   MSCK REPAIR TABLE my_datalake_db.sales_partitioned;
   ```
4. **If still empty → Check S3:**
   ```bash
   aws s3 ls s3://your-bucket-name/sales_partitioned/ --recursive
   ```
5. **Verify folder naming:** Must be `year=2025/month=01/` format

---

## Section C: EC2 → S3 → Athena Lab

### Prerequisites: EC2 Setup

**Launch EC2 instance:**
- Type: t2.micro (Free Tier)
- AMI: Amazon Linux 2023
- IAM Role: Attach `AmazonS3FullAccess`
- Security Group: Allow SSH (port 22) from your IP

**Connect:**
```bash
ssh -i your-key.pem ec2-user@<EC2-Public-IP>
```

**Install dependencies:**
```bash
sudo yum update -y
sudo yum install -y python3 python3-pip
aws --version  # Pre-installed on Amazon Linux
```

**Verify S3 access:**
```bash
aws s3 ls  # Should list your buckets
```

---

### Exercise 14: Generate Data with EC2

**Solution:**

Create `generate_sales.py`:
```python
#!/usr/bin/env python3
import random
import csv
from datetime import datetime, timedelta

NUM_RECORDS = 200
OUTPUT_FILE = 'ec2_generated.csv'

customers = ["Alice", "Bob", "Charlie", "David", "Eve", "Frank",
             "Grace", "Hannah", "Ian", "Judy", "Zara", "Yasmin"]

start_date = datetime(2025, 1, 1)

print(f"Generating {NUM_RECORDS} records...")

with open(OUTPUT_FILE, 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(['id', 'customer', 'amount', 'date'])

    base_id = 100000
    for i in range(1, NUM_RECORDS + 1):
        record_id = base_id + i
        customer = random.choice(customers)
        amount = random.randint(5, 1500)
        days_offset = (i - 1) % 180  # Spread over 6 months
        record_date = (start_date + timedelta(days=days_offset)).strftime('%Y-%m-%d')

        writer.writerow([record_id, customer, amount, record_date])

print(f"✓ Generated {OUTPUT_FILE} with {NUM_RECORDS} records")
```

**Run:**
```bash
chmod +x generate_sales.py
python3 generate_sales.py
```

**Expected output:**
```
Generating 200 records...
✓ Generated ec2_generated.csv with 200 records
```

**Verify:**
```bash
ls -lh ec2_generated.csv
head -n 5 ec2_generated.csv
wc -l ec2_generated.csv  # Should be 201 (header + 200 rows)
```

**Real-world modifications:**

Generate larger datasets:
```python
NUM_RECORDS = 10_000_000  # 10 million rows
```

Add more realistic data (using `faker` library):
```bash
pip3 install faker
```

```python
from faker import Faker
fake = Faker()

customer_email = fake.email()
transaction_id = fake.uuid4()
ip_address = fake.ipv4()
```

Streaming data generation (continuous):
```python
while True:
    generate_row()
    time.sleep(1)  # One row per second
```

---

### Exercise 15: Upload Data from EC2 to S3

**Solution:**
```bash
# Upload file
aws s3 cp ec2_generated.csv s3://your-bucket-name/ec2-data/ec2_generated.csv

# Verify upload
aws s3 ls s3://your-bucket-name/ec2-data/
```

**Expected output:**
```
2025-11-10 10:30:45      10234 ec2_generated.csv
```

**Explanation:**
- `aws s3 cp` - copies file from local to S3
- Creates folder `ec2-data/` if it doesn't exist
- `ls` verifies file exists in S3

**Alternative upload methods:**

Using Python boto3:
```python
import boto3

s3 = boto3.client('s3')
s3.upload_file('ec2_generated.csv', 'your-bucket-name', 'ec2-data/ec2_generated.csv')
print("✓ Uploaded to S3")
```

Sync entire directory:
```bash
aws s3 sync ./local_data/ s3://your-bucket-name/ec2-data/
# Only uploads new/changed files
```

Multipart upload (for large files):
```bash
aws s3 cp large_file.csv s3://your-bucket-name/data/ \
  --metadata '{"source":"ec2","date":"2025-11-10"}'
```

**Automated daily upload (cron job):**
```bash
# Edit crontab
crontab -e

# Add line (runs daily at 2 AM):
0 2 * * * /home/ec2-user/generate_and_upload.sh
```

`generate_and_upload.sh`:
```bash
#!/bin/bash
cd /home/ec2-user
python3 generate_sales.py
aws s3 cp ec2_generated.csv s3://your-bucket-name/ec2-data/$(date +\%Y-\%m-\%d).csv
echo "✓ Uploaded $(date)"
```

---

### Exercise 16: Create Athena Table for Uploaded Data

**Solution:**
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS my_datalake_db.ec2_sales (
  id INT,
  customer STRING,
  amount INT,
  date STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES ('separatorChar'=',')
LOCATION 's3://your-bucket-name/ec2-data/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

**Verify:**
```sql
SELECT * FROM my_datalake_db.ec2_sales LIMIT 10;
```

**Expected result:** 10 rows with IDs 100001-100010

**Explanation:**
- `LOCATION` points to folder containing CSV files
- If you add more CSVs to `ec2-data/` folder, Athena automatically queries all of them
- No need to recreate table when adding new files

**Query file metadata:**
```sql
SELECT
  "$path" AS file_path,
  COUNT(*) AS rows_in_file
FROM my_datalake_db.ec2_sales
GROUP BY "$path";
```

Shows which S3 file each row came from (useful for debugging).

---

### Exercise 17: Run Query to Validate Data

**Solution:**
```sql
SELECT COUNT(*) AS total_rows
FROM my_datalake_db.ec2_sales;
```

**Expected result:** `200`

**Comprehensive validation:**
```sql
-- Date range
SELECT MIN(date) AS earliest, MAX(date) AS latest
FROM my_datalake_db.ec2_sales;
-- Expected: 2025-01-01 to 2025-06-29

-- Check for NULLs
SELECT
  SUM(CASE WHEN id IS NULL THEN 1 ELSE 0 END) AS null_ids,
  SUM(CASE WHEN customer IS NULL THEN 1 ELSE 0 END) AS null_customers,
  SUM(CASE WHEN amount IS NULL THEN 1 ELSE 0 END) AS null_amounts,
  SUM(CASE WHEN date IS NULL THEN 1 ELSE 0 END) AS null_dates
FROM my_datalake_db.ec2_sales;
-- Expected: All zeros

-- Amount distribution
SELECT
  MIN(amount) AS min_amount,
  MAX(amount) AS max_amount,
  ROUND(AVG(amount), 2) AS avg_amount,
  APPROX_PERCENTILE(amount, 0.5) AS median_amount
FROM my_datalake_db.ec2_sales;
-- Expected: min=5, max=1500

-- Duplicate check
SELECT id, COUNT(*) AS occurrences
FROM my_datalake_db.ec2_sales
GROUP BY id
HAVING COUNT(*) > 1;
-- Expected: Empty result (no duplicates)
```

**Data quality framework:**
```sql
-- Data quality report
SELECT
  'Total Rows' AS metric,
  CAST(COUNT(*) AS VARCHAR) AS value
FROM my_datalake_db.ec2_sales

UNION ALL

SELECT 'Null IDs', CAST(SUM(CASE WHEN id IS NULL THEN 1 ELSE 0 END) AS VARCHAR)
FROM my_datalake_db.ec2_sales

UNION ALL

SELECT 'Duplicate IDs', CAST(COUNT(*) AS VARCHAR)
FROM (
  SELECT id FROM my_datalake_db.ec2_sales GROUP BY id HAVING COUNT(*) > 1
)

UNION ALL

SELECT 'Invalid Amounts', CAST(COUNT(*) AS VARCHAR)
FROM my_datalake_db.ec2_sales
WHERE amount < 0 OR amount IS NULL;
```

---

### Exercise 18: Transform Data with CTAS

**Solution:**
```sql
CREATE TABLE my_datalake_db.ec2_sales_parquet
WITH (
  format = 'PARQUET',
  parquet_compression = 'SNAPPY',
  partitioned_by = ARRAY['year', 'month'],
  external_location = 's3://your-bucket-name/ec2_parquet/'
) AS
SELECT
  id,
  customer,
  amount,
  substr(date, 1, 4) AS year,
  substr(date, 6, 2) AS month
FROM my_datalake_db.ec2_sales;
```

**After CTAS:**
```sql
-- Load partitions
MSCK REPAIR TABLE my_datalake_db.ec2_sales_parquet;

-- Verify
SELECT COUNT(*) FROM my_datalake_db.ec2_sales_parquet;
-- Expected: 200

-- Check partitions
SHOW PARTITIONS my_datalake_db.ec2_sales_parquet;
-- Expected: year=2025/month=01 through year=2025/month=06
```

**Compare performance:**

CSV query:
```sql
SELECT customer, SUM(amount) AS total
FROM my_datalake_db.ec2_sales
WHERE substr(date, 1, 7) = '2025-01'
GROUP BY customer;
```
**Data scanned:** ~10 KB (must scan all to filter)

Parquet query:
```sql
SELECT customer, SUM(amount) AS total
FROM my_datalake_db.ec2_sales_parquet
WHERE year='2025' AND month='01'
GROUP BY customer;
```
**Data scanned:** ~2 KB (partition pruning + columnar format)

**5x cost reduction** on this small dataset; scales to 10-50x on larger datasets!

---

### Exercise 19: Partition Uploaded Data by Date

**Solution (split CSV by month and upload):**

Create `split_and_upload.py` on EC2:
```python
#!/usr/bin/env python3
import csv
import subprocess
from collections import defaultdict

data_by_month = defaultdict(list)

# Read and group by month
with open('ec2_generated.csv', 'r') as f:
    reader = csv.DictReader(f)
    header = reader.fieldnames

    for row in reader:
        year = row['date'][:4]
        month = row['date'][5:7]
        data_by_month[(year, month)].append(row)

# Write and upload partitioned files
bucket = 'your-bucket-name'

for (year, month), rows in data_by_month.items():
    filename = f'ec2_{year}{month}.csv'

    # Write CSV
    with open(filename, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=header)
        writer.writeheader()
        writer.writerows(rows)

    # Upload to S3 partition
    s3_path = f's3://{bucket}/ec2_partitioned/year={year}/month={month}/{filename}'
    subprocess.run(['aws', 's3', 'cp', filename, s3_path])
    print(f'✓ Uploaded {filename} to partition year={year}/month={month}')
```

**Run:**
```bash
python3 split_and_upload.py
```

**Create partitioned table:**
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS my_datalake_db.ec2_partitioned (
  id INT,
  customer STRING,
  amount INT,
  date STRING
)
PARTITIONED BY (year STRING, month STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://your-bucket-name/ec2_partitioned/'
TBLPROPERTIES ('skip.header.line.count'='1');

-- Load partitions
MSCK REPAIR TABLE my_datalake_db.ec2_partitioned;
```

**Verify:**
```sql
SHOW PARTITIONS my_datalake_db.ec2_partitioned;
```

---

### Exercise 20: Query Combined Datasets (Join)

**Solution 1: Simple join**
```sql
SELECT
  s.id AS sales_id,
  s.customer,
  s.amount AS sales_amount,
  e.id AS ec2_id,
  e.amount AS ec2_amount,
  s.date AS sales_date,
  e.date AS ec2_date
FROM my_datalake_db.sample_sales s
LEFT JOIN my_datalake_db.ec2_sales e
  ON s.customer = e.customer
LIMIT 100;
```

**Explanation:**
- `LEFT JOIN` - keeps all rows from `sample_sales` (left table)
- Matches on `customer` name
- Null values in `e.*` columns if no match found

**Solution 2: Aggregated join (customer summary)**
```sql
SELECT
  s.customer,
  COUNT(DISTINCT s.id) AS sales_tx_count,
  COALESCE(SUM(s.amount), 0) AS total_sales_amount,
  COUNT(DISTINCT e.id) AS ec2_tx_count,
  COALESCE(SUM(e.amount), 0) AS total_ec2_amount,
  COALESCE(SUM(s.amount), 0) + COALESCE(SUM(e.amount), 0) AS combined_revenue
FROM my_datalake_db.sample_sales s
LEFT JOIN my_datalake_db.ec2_sales e
  ON s.customer = e.customer
GROUP BY s.customer
ORDER BY combined_revenue DESC;
```

**Use case:** Executive dashboard showing total customer value across all data sources.

**Solution 3: Join with lookup table**

First, upload `customers_lookup.csv` (from assets):
```bash
aws s3 cp ../assets/customers_lookup.csv s3://your-bucket-name/lookup/customers_lookup.csv
```

Create lookup table:
```sql
CREATE EXTERNAL TABLE my_datalake_db.customers_lookup (
  customer STRING,
  email STRING,
  region STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://your-bucket-name/lookup/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

Enriched join:
```sql
SELECT
  s.id,
  s.customer,
  c.email,
  c.region,
  s.amount,
  s.date
FROM my_datalake_db.sample_sales s
LEFT JOIN my_datalake_db.customers_lookup c
  ON s.customer = c.customer
WHERE c.region = 'US-East'
LIMIT 50;
```

**Use case:** Filtered reports by region with customer contact info for follow-up.

**Performance tips for joins:**
- Filter early: Apply WHERE clauses before JOIN when possible
- Use partition filters: `WHERE s.year='2025' AND s.month='01'`
- Broadcast small tables: Athena automatically optimizes small lookup tables
- For large-large joins: Consider bucketing or pre-aggregation

---

## Summary

**Skills you mastered:**
✓ SQL fundamentals (filtering, aggregations, joins)
✓ Creating and managing partitioned Athena tables
✓ CTAS for format conversion and optimization
✓ Building end-to-end data pipelines (EC2 → S3 → Athena)
✓ Performance tuning and cost optimization
✓ Data quality validation

**Cost optimization achieved:**
- CSV to Parquet: **5-20x cost reduction**
- Adding partitions: **10-100x cost reduction**
- Combined: **50-1000x cost reduction** possible!

**Real-world application:**
These patterns are used by data engineers at companies processing terabytes to petabytes daily:
- Log analytics (CloudTrail, application logs)
- IoT data processing (sensor streams)
- Business intelligence (sales, marketing data)
- Data lake analytics (centralized data repositories)

**Next steps:**
1. Build a production data pipeline with automation
2. Integrate with BI tools (QuickSight, Tableau, Metabase)
3. Explore AWS Glue for ETL automation
4. Learn about Lake Formation for data governance
5. Study advanced Athena features (federated queries, ML functions)

**Congratulations on completing the exercises!** 🎉

You're now equipped to build cost-effective, scalable data analytics solutions with AWS Athena.
