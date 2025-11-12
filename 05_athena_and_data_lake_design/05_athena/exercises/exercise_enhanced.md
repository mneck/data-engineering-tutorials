# Athena Hands-On Exercises

## TL;DR
20 progressive exercises covering SQL queries, partitioning, CTAS transformations, and EC2-to-S3-to-Athena data pipelines. Each includes context, hints, and expected outcomes. Datasets provided in assets folder.

---

## How to Use This Exercise Guide

**Structure:**
- **Section A:** SQL fundamentals (7 exercises)
- **Section B:** Partitioning and optimization (6 exercises)
- **Section C:** End-to-end data pipeline (7 exercises)

**Approach:**
1. Read the exercise description
2. Try to solve it yourself first
3. Use hints if stuck
4. Check your results against expected outcomes
5. Review solutions only after attempting

**Time estimate:**
- Section A: 30-45 minutes
- Section B: 45-60 minutes
- Section C: 60-90 minutes

---

## Prerequisites

### Before Starting Section A:
✓ AWS account with Athena access
✓ IAM permissions: `AmazonAthenaFullAccess`, `AmazonS3FullAccess`
✓ Athena query results location configured
✓ Database created (e.g., `my_datalake_db`)

### Dataset: sales_data.csv
Located at: `../assets/sales_data.csv`

**What it contains:**
- Customer transaction data with ~1000 rows
- Columns: `id`, `customer`, `amount`, `date`
- Date range: 2025-01-01 to 2025-06-30
- Customers: Alice, Bob, Charlie, David, Eve, Frank, Grace, Hannah, Ian, Judy

**Upload to S3:**
```bash
aws s3 cp ../assets/sales_data.csv s3://your-bucket-name/data/sales_data.csv
```

---

## Section A: SQL Queries in Athena (Exercises 1-7)

### Setup: Create the Base Table

Before starting, create the Athena table:

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

**Verify setup:**
```sql
SELECT * FROM my_datalake_db.sample_sales LIMIT 5;
```

Expected: 5 rows with customer names and transaction amounts.

---

### Exercise 1: Filter by Multiple Conditions

**Real-world context:**
You're analyzing high-value transactions for a specific customer. Marketing wants to know which customers made purchases over $200.

**Task:**
Query all sales where:
- `customer = 'Alice'` AND
- `amount > 200`

**Learning objective:** Combining multiple WHERE conditions with AND/OR

**Hints:**
- Use `AND` to combine conditions
- String values need single quotes: `'Alice'`
- Numeric values don't need quotes: `200`

**Expected result:**
- Only rows with Alice's name
- All amounts greater than 200
- Likely 10-30 rows depending on data

**Questions to consider:**
- What if you wanted Alice OR Bob? (use `OR`)
- What if you wanted amount between 200 and 500? (use `BETWEEN`)

---

### Exercise 2: Sort Results by Amount

**Real-world context:**
Finance team needs to review the largest transactions for potential fraud detection.

**Task:**
List the top 5 largest transactions by amount.

**Learning objective:** Ordering and limiting results

**Hints:**
- Use `ORDER BY column_name DESC` for descending order
- Use `LIMIT 5` to get only first 5 rows
- `DESC` = descending (largest first), `ASC` = ascending (smallest first)

**Expected result:**
- 5 rows
- Sorted from highest to lowest amount
- Should see the biggest transactions first

**Questions to consider:**
- How would you get the 5 smallest transactions?
- How would you get transactions ranked 6-10?

---

### Exercise 3: Find Distinct Customers

**Real-world context:**
You need to count how many unique customers made purchases (for customer acquisition metrics).

**Task:**
Query all unique customer names from the table.

**Learning objective:** Removing duplicates with DISTINCT

**Hints:**
- Use `SELECT DISTINCT column_name`
- Consider adding `ORDER BY` for readable output

**Expected result:**
- 10 unique customer names (Alice, Bob, Charlie, David, Eve, Frank, Grace, Hannah, Ian, Judy)
- No duplicates

**Questions to consider:**
- How would you count the number of distinct customers? (use `COUNT(DISTINCT ...)`)
- Can you use DISTINCT on multiple columns?

---

### Exercise 4: Group By and Average

**Real-world context:**
Management wants to understand customer behavior—who are the high-value vs low-value customers?

**Task:**
Show the average transaction amount per customer.

**Learning objective:** Aggregations with GROUP BY

**Hints:**
- Use `GROUP BY customer`
- Use `AVG(amount)` to calculate average
- Consider adding `COUNT(*)` to see transaction count per customer
- Use `ROUND(AVG(amount), 2)` for cleaner output

**Expected result:**
- One row per customer
- Average amount for each customer
- Sorted by average (use `ORDER BY avg_amount DESC`)

**Questions to consider:**
- Which customer has the highest average purchase?
- Is there a correlation between transaction count and average amount?
- How would you filter to show only customers with avg > $300?

---

### Exercise 5: Filter by Date Range

**Real-world context:**
Quarter-end reporting—you need to extract Q1 (January-March) transactions.

**Task:**
Get all sales between `2025-01-01` and `2025-01-10`.

**Learning objective:** Date filtering with BETWEEN

**Hints:**
- Use `WHERE date BETWEEN 'start_date' AND 'end_date'`
- Dates as strings work if format is `YYYY-MM-DD`
- For broader range, adjust end date (e.g., `2025-03-31` for Q1)

**Expected result:**
- Only rows with dates in specified range
- Dates should be within first 10 days of January

**Questions to consider:**
- How would you get all Q1 data? (change end date to '2025-03-31')
- What if dates were in different formats (MM/DD/YYYY)?
- How to count transactions per month in Q1?

---

### Exercise 6: Use LIKE for Pattern Matching

**Real-world context:**
Sales team wants to analyze all customers whose names start with 'C' for a targeted campaign.

**Task:**
Find all customers whose names start with `C`.

**Learning objective:** Text pattern matching with LIKE

**Hints:**
- Use `WHERE customer LIKE 'C%'`
- `%` = wildcard (matches any characters)
- `'C%'` = starts with C
- `'%son'` = ends with "son"
- `'%ar%'` = contains "ar"

**Expected result:**
- Rows with customer names: Charlie
- Depending on data, might include other C names

**Questions to consider:**
- How to find names ending in 'e'? (`LIKE '%e'`)
- How to find names containing 'ar'? (`LIKE '%ar%'`)
- How to exclude certain names? (`NOT LIKE 'C%'`)

---

### Exercise 7: Combine Aggregates

**Real-world context:**
Executive dashboard needs customer summary: min, max, average, and count of transactions.

**Task:**
Show `MIN(amount)`, `MAX(amount)`, `AVG(amount)`, and `COUNT(*)` per customer.

**Learning objective:** Multiple aggregations in one query

**Hints:**
- Put all aggregate functions in SELECT clause
- Use `GROUP BY customer`
- Use `AS` to rename columns for clarity
- Use `ROUND()` for cleaner averages

**Expected result:**
- One row per customer
- Four metric columns per customer
- Sortable by any metric

**Example output structure:**
```
customer | min_amount | max_amount | avg_amount | tx_count
Alice    | 5          | 980        | 245.50     | 150
Bob      | 10         | 1200       | 300.25     | 120
...
```

**Questions to consider:**
- Which customer has the highest variance (max - min)?
- Which customer is most consistent?
- How to calculate total revenue per customer? (use `SUM(amount)`)

---

## Section B: Optimize with Partitions (Exercises 8-13)

### Setup: Partitioned Data

**Dataset:** `s3_partitioned_sales.zip` (located in `../assets/`)

**What it contains:**
- Pre-partitioned sales data by year/month
- Structure: `year=2025/month=01/`, `year=2025/month=02/`, etc.
- Same schema as `sales_data.csv`

**Setup steps:**

1. **Unzip the file:**
```bash
cd ../assets/
unzip s3_partitioned_sales.zip
```

2. **Upload to S3 preserving structure:**
```bash
aws s3 cp s3_partitioned_sales/ s3://your-bucket-name/sales_partitioned/ --recursive
```

3. **Verify structure:**
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

**Real-world context:**
You're building a data lake for log analytics. Logs arrive daily and you need daily-level query granularity.

**Task:**
Create a partitioned table structure with `year`, `month`, and `day` partition columns.

**Learning objective:** Multi-level partitioning setup

**Hints:**
- Use `PARTITIONED BY (year STRING, month STRING, day STRING)`
- Partition columns are NOT in main column list
- For this exercise, you can use the month-level data or create your own day-level folders

**Sample CREATE TABLE:**
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS my_datalake_db.sales_partitioned (
  id INT,
  customer STRING,
  amount INT
)
PARTITIONED BY (year STRING, month STRING, day STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
LOCATION 's3://your-bucket-name/sales_partitioned/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

**Expected result:**
- Table created successfully
- Ready to load partitions

**Questions to consider:**
- When is day-level partitioning necessary? (high-frequency logs, need to query specific days)
- What are downsides of too many partition levels? (many small files, metadata overhead)

---

### Exercise 9: Manually Add a Partition

**Real-world context:**
New data arrived for February 2025. You need to make it queryable immediately without running MSCK REPAIR.

**Task:**
Add partition `year=2025, month=02` manually using ALTER TABLE.

**Learning objective:** Manual partition management

**Hints:**
```sql
ALTER TABLE my_datalake_db.sales_partitioned
ADD PARTITION (year='2025', month='02')
LOCATION 's3://your-bucket-name/sales_partitioned/year=2025/month=02/';
```

**Expected result:**
- Partition added successfully
- Verify with: `SHOW PARTITIONS my_datalake_db.sales_partitioned;`

**When to use manual ADD PARTITION:**
- Real-time/streaming data pipelines
- Specific partitions only (not bulk discovery)
- Non-standard partition paths
- Automation scripts (Lambda, Glue jobs)

**Questions to consider:**
- How would you add multiple partitions at once? (multiple ALTER TABLE statements)
- What happens if you add a partition that doesn't exist in S3?

---

### Exercise 10: Drop a Partition

**Real-world context:**
You're implementing data retention policies. Data older than 2 years must be archived.

**Task:**
Drop partition `year=2025, month=01` from your table.

**Learning objective:** Partition lifecycle management

**Hints:**
```sql
ALTER TABLE my_datalake_db.sales_partitioned
DROP PARTITION (year='2025', month='01');
```

**Important warnings:**
- ⚠️ This ONLY removes metadata (partition registration)
- ⚠️ Files in S3 are NOT deleted
- ⚠️ To delete files too, use: `aws s3 rm s3://path/to/partition/ --recursive`

**Expected result:**
- Partition removed from metadata
- `SHOW PARTITIONS` no longer lists it
- Files still exist in S3 (until you manually delete them)

**Questions to consider:**
- How to implement automated data retention? (AWS Glue workflow + S3 lifecycle policies)
- Should you delete S3 files immediately or archive to Glacier first?

---

### Exercise 11: Compare Query Cost With vs Without Partition Filter

**Real-world context:**
You're optimizing costs. Need to prove to management that partitioning saves money.

**Task:**
Run two identical queries—one with partition filter, one without—and compare "Data scanned."

**Learning objective:** Understanding partition pruning impact

**Query 1: Without partition filter (scans all data)**
```sql
SELECT COUNT(*) AS total_transactions
FROM my_datalake_db.sales_partitioned;
```

**Query 2: With partition filter (scans only January)**
```sql
SELECT COUNT(*) AS january_transactions
FROM my_datalake_db.sales_partitioned
WHERE year='2025' AND month='01';
```

**How to compare:**
1. Run Query 1, note "Data scanned" from query details (top-right)
2. Run Query 2, note "Data scanned"
3. Calculate reduction: `(1 - scanned2/scanned1) * 100`%

**Expected result:**
- Query 2 scans ~1/6th of data (if you have 6 months)
- Cost reduction: ~83%

**Questions to consider:**
- If you have 5 years of data partitioned by month, how much savings for querying 1 month?
- What if you query 3 months? Still worth filtering?
- Calculate actual $ savings: `(data_scanned_GB / 1000) * $5`

---

### Exercise 12: Convert to Parquet with Compression

**Real-world context:**
You're migrating from CSV data lake to optimized Parquet format for 10x cost reduction.

**Task:**
Create a CTAS query that converts `sample_sales` (CSV) to Parquet format with Snappy compression.

**Learning objective:** Format conversion and optimization with CTAS

**Hints:**
```sql
CREATE TABLE my_datalake_db.sales_parquet_partitioned
WITH (
  format = 'PARQUET',
  parquet_compression = 'SNAPPY',
  partitioned_by = ARRAY['year','month'],
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

**Compression options:**
- `SNAPPY` - balanced speed/compression (recommended default)
- `GZIP` - better compression, slower
- `ZSTD` - best compression, newest
- `NONE` - no compression

**After running:**
1. Run `MSCK REPAIR TABLE my_datalake_db.sales_parquet_partitioned;`
2. Query both tables with same SELECT statement
3. Compare "Data scanned"

**Expected result:**
- New Parquet table created
- 5-20x smaller file size
- 3-10x less data scanned for queries

**Questions to consider:**
- When would you use GZIP instead of SNAPPY?
- What's the trade-off of heavy compression?
- How often should you convert CSV to Parquet? (during ETL, nightly batch, etc.)

---

### Exercise 13: Show All Partitions

**Real-world context:**
Debugging data pipeline—need to verify which partitions exist and identify gaps.

**Task:**
List all partitions for your partitioned table.

**Learning objective:** Partition metadata inspection

**Hints:**
```sql
SHOW PARTITIONS my_datalake_db.sales_parquet_partitioned;
```

**Expected output:**
```
year=2025/month=01
year=2025/month=02
year=2025/month=03
...
```

**Use cases:**
- Verify new partitions were added after ETL job
- Identify missing months (data gaps)
- Audit partition structure before migrations
- Debugging "no data" issues

**Questions to consider:**
- How to count total partitions? (count output rows)
- How to automate partition gap detection? (query Glue Data Catalog API)

---

## Section C: EC2 → S3 → Athena Lab (Exercises 14-20)

### Overview: Building a Data Pipeline

**Scenario:** You're building a simple data pipeline:
1. Application on EC2 generates data
2. Upload to S3 (data lake)
3. Query with Athena (analytics)

This mimics real-world log collection, IoT data ingestion, or batch data exports.

---

### Prerequisites: Launch and Configure EC2

**Before starting Exercise 14, set up EC2:**

#### Step 1: Launch EC2 Instance

**Via AWS Console:**
1. EC2 Dashboard → Launch Instance
2. **Name:** `athena-data-generator`
3. **AMI:** Amazon Linux 2023 (or Ubuntu 22.04)
4. **Instance type:** `t2.micro` (Free Tier)
5. **Key pair:** Create new or use existing (download .pem file)
6. **Network settings:**
   - Auto-assign Public IP: Enable
   - Security group: Create new
   - Allow SSH (port 22) from your IP
7. **IAM instance profile:**
   - Create IAM role with `AmazonS3FullAccess` policy
   - Attach to instance
8. **Launch instance**

**Via AWS CLI:**
```bash
# Create IAM role (if not exists)
aws iam create-role --role-name EC2-S3-Access --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name EC2-S3-Access --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Launch instance
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --key-name your-key-pair \
  --security-group-ids sg-xxxxxx \
  --iam-instance-profile Name=EC2-S3-Access
```

#### Step 2: Connect to EC2

```bash
# SSH into instance
ssh -i your-key.pem ec2-user@<EC2-Public-IP>

# Or use EC2 Instance Connect in AWS Console
```

#### Step 3: Install Dependencies

**Amazon Linux 2023:**
```bash
sudo yum update -y
sudo yum install -y python3 python3-pip
aws --version  # AWS CLI pre-installed
```

**Ubuntu:**
```bash
sudo apt update
sudo apt install -y python3 python3-pip awscli
```

#### Step 4: Verify S3 Access

```bash
# List your buckets
aws s3 ls

# Try uploading a test file
echo "test" > test.txt
aws s3 cp test.txt s3://your-bucket-name/test.txt
aws s3 ls s3://your-bucket-name/
```

If you see "Access Denied," check IAM role attached to EC2.

---

### Exercise 14: Generate Data with EC2

**Real-world context:**
Your application logs transactions to local files. Need to simulate this for testing.

**Task:**
Create a Python script on EC2 that generates a CSV file with 200 sample sales records.

**Script:** `generate_sales.py`

```python
#!/usr/bin/env python3
import random
import csv
from datetime import datetime, timedelta

# Configuration
NUM_RECORDS = 200
OUTPUT_FILE = 'ec2_generated.csv'

customers = [
    "Alice", "Bob", "Charlie", "David", "Eve",
    "Frank", "Grace", "Hannah", "Ian", "Judy",
    "Zara", "Yasmin"
]

start_date = datetime(2025, 1, 1)

# Generate data
print(f"Generating {NUM_RECORDS} records...")

with open(OUTPUT_FILE, 'w', newline='') as f:
    writer = csv.writer(f)

    # Header
    writer.writerow(['id', 'customer', 'amount', 'date'])

    # Data rows
    base_id = 100000
    for i in range(1, NUM_RECORDS + 1):
        record_id = base_id + i
        customer = random.choice(customers)
        amount = random.randint(5, 1500)
        # Spread over 6 months
        days_offset = (i - 1) % 180
        record_date = (start_date + timedelta(days=days_offset)).strftime('%Y-%m-%d')

        writer.writerow([record_id, customer, amount, record_date])

print(f"✓ Generated {OUTPUT_FILE} with {NUM_RECORDS} records")
```

**Steps:**
1. SSH into EC2
2. Create file: `nano generate_sales.py`
3. Paste script above, save (Ctrl+O, Enter, Ctrl+X)
4. Make executable: `chmod +x generate_sales.py`
5. Run: `python3 generate_sales.py`

**Expected result:**
```bash
Generating 200 records...
✓ Generated ec2_generated.csv with 200 records
```

**Verify:**
```bash
ls -lh ec2_generated.csv   # Should show ~10KB file
head -n 5 ec2_generated.csv  # Preview first 5 rows
wc -l ec2_generated.csv    # Should show 201 lines (header + 200 rows)
```

**Questions to consider:**
- How would you generate larger datasets? (increase NUM_RECORDS)
- How to generate streaming data? (infinite loop with sleep)
- How to add more realistic data? (use faker library)

---

### Exercise 15: Upload Data from EC2 to S3

**Real-world context:**
Batch job completed, now ship data to data lake for analysis.

**Task:**
Use AWS CLI to upload `ec2_generated.csv` to S3.

**Hints:**
```bash
# Upload file
aws s3 cp ec2_generated.csv s3://your-bucket-name/ec2-data/ec2_generated.csv

# Verify upload
aws s3 ls s3://your-bucket-name/ec2-data/

# Check file size in S3
aws s3 ls s3://your-bucket-name/ec2-data/ --human-readable
```

**Expected result:**
```
2025-11-10 10:30:45   10.5 KiB ec2_generated.csv
```

**Troubleshooting:**
- **Error: Access Denied** → Check IAM role attached to EC2
- **Error: Bucket not found** → Verify bucket name and region
- **Upload slow** → Check instance's network bandwidth

**Questions to consider:**
- How to automate daily uploads? (cron job: `0 2 * * * /path/to/upload_script.sh`)
- How to handle large files? (use `aws s3 sync` or multipart upload)
- How to encrypt uploads? (use `--sse AES256` flag)

---

### Exercise 16: Create Athena Table for Uploaded Data

**Real-world context:**
Data landed in S3, now make it queryable for analysts.

**Task:**
Create an Athena external table pointing to the EC2-generated data.

**Hints:**
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

**Expected result:**
- 10 rows displayed
- Columns: id, customer, amount, date
- IDs starting from 100001

**Questions to consider:**
- What if you upload more CSV files to `ec2-data/` folder? (Athena queries all automatically)
- How to handle schema changes? (recreate table or use schema evolution)

---

### Exercise 17: Run Query to Validate Data

**Real-world context:**
Data quality check—verify row count matches what was generated.

**Task:**
Count total rows in `ec2_sales` table.

**Hints:**
```sql
SELECT COUNT(*) AS total_rows
FROM my_datalake_db.ec2_sales;
```

**Expected result:**
```
total_rows
200
```

**Additional validation queries:**

```sql
-- Check date range
SELECT MIN(date) AS earliest_date, MAX(date) AS latest_date
FROM my_datalake_db.ec2_sales;

-- Check for NULL values
SELECT
  COUNT(*) AS total,
  COUNT(id) AS non_null_id,
  COUNT(customer) AS non_null_customer,
  COUNT(amount) AS non_null_amount
FROM my_datalake_db.ec2_sales;

-- Check amount distribution
SELECT
  MIN(amount) AS min_amount,
  MAX(amount) AS max_amount,
  ROUND(AVG(amount), 2) AS avg_amount
FROM my_datalake_db.ec2_sales;
```

**Questions to consider:**
- How to detect data quality issues? (check for NULLs, outliers, duplicates)
- How to automate validation? (AWS Glue DataBrew, Lambda with assertions)

---

### Exercise 18: Transform Data with CTAS (CSV → Parquet)

**Real-world context:**
Raw CSV data loaded, now optimize for repeated querying.

**Task:**
Convert `ec2_sales` (CSV) to Parquet format with partitions using CTAS.

**Hints:**
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

**After running:**
```sql
-- Load partitions
MSCK REPAIR TABLE my_datalake_db.ec2_sales_parquet;

-- Verify
SELECT COUNT(*) FROM my_datalake_db.ec2_sales_parquet;
SHOW PARTITIONS my_datalake_db.ec2_sales_parquet;
```

**Expected result:**
- 200 rows in Parquet table
- Partitions: `year=2025/month=01` through `year=2025/month=06`

**Compare performance:**
```sql
-- CSV query
SELECT AVG(amount) FROM my_datalake_db.ec2_sales WHERE substr(date,1,7) = '2025-01';

-- Parquet query
SELECT AVG(amount) FROM my_datalake_db.ec2_sales_parquet WHERE year='2025' AND month='01';
```

Check "Data scanned" for both—Parquet should be significantly less.

**Questions to consider:**
- Should you keep both CSV and Parquet? (yes for archival, no for cost)
- When to run CTAS? (nightly batch after data loads)

---

### Exercise 19: Partition Uploaded Data by Date

**Real-world context:**
You want to partition data at the source (during upload) rather than via CTAS.

**Task:**
Re-upload EC2 data into partitioned folder structure.

**On EC2:**

Option A - Manual (for one partition):
```bash
# Upload to specific partition
aws s3 cp ec2_generated.csv s3://your-bucket-name/ec2_partitioned/year=2025/month=01/ec2_202501.csv
```

Option B - Script to split by month:

```python
#!/usr/bin/env python3
import csv
from collections import defaultdict

# Read CSV
data_by_month = defaultdict(list)

with open('ec2_generated.csv', 'r') as f:
    reader = csv.DictReader(f)
    header = reader.fieldnames

    for row in reader:
        date = row['date']
        year = date[:4]
        month = date[5:7]
        data_by_month[(year, month)].append(row)

# Write partitioned files
for (year, month), rows in data_by_month.items():
    filename = f'ec2_{year}{month}.csv'
    with open(filename, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=header)
        writer.writeheader()
        writer.writerows(rows)

    # Upload to S3 in partition structure
    s3_path = f's3://your-bucket-name/ec2_partitioned/year={year}/month={month}/{filename}'
    import subprocess
    subprocess.run(['aws', 's3', 'cp', filename, s3_path])
    print(f'✓ Uploaded {filename} to {s3_path}')
```

**Run script:**
```bash
python3 split_and_upload.py
```

**Create table:**
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

**Expected result:**
- Data split into 6 partitions (Jan-Jun 2025)
- Queryable with partition filters

**Questions to consider:**
- When to partition at source vs via CTAS? (streaming data = source, batch = CTAS)
- How to automate? (AWS Glue job, Lambda on S3 PUT events)

---

### Exercise 20: Query Combined Datasets (Join)

**Real-world context:**
Joining internal sales data with external EC2-generated logs for enriched analysis.

**Task:**
Join `sample_sales` with `ec2_sales` on the `customer` column.

**Hints:**

**Simple join to see overlap:**
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

**Aggregated join (customer summary):**
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

**Join with customer lookup table:**

If you have `customers_lookup.csv` (in assets folder):
```csv
customer,email,region
Alice,alice@example.com,US-East
Bob,bob@example.com,EU-West
...
```

Upload to S3 and create table:
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

**Enriched join:**
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
LIMIT 50;
```

**Expected result:**
- Combined data from multiple sources
- Enriched with customer metadata
- Useful for dashboards and reports

**Questions to consider:**
- What join type to use? (LEFT, INNER, FULL OUTER?)
- How to handle NULL values from join? (use COALESCE)
- Performance of large joins? (consider bucketing or pre-aggregation)

---

## Post-Exercise: Clean Up Resources

To avoid unexpected charges, clean up resources you created:

### 1. Delete S3 Data
```bash
aws s3 rm s3://your-bucket-name/ec2-data/ --recursive
aws s3 rm s3://your-bucket-name/ec2_parquet/ --recursive
aws s3 rm s3://your-bucket-name/sales_partitioned/ --recursive
```

### 2. Drop Athena Tables
```sql
DROP TABLE IF EXISTS my_datalake_db.ec2_sales;
DROP TABLE IF EXISTS my_datalake_db.ec2_sales_parquet;
DROP TABLE IF EXISTS my_datalake_db.ec2_partitioned;
DROP TABLE IF EXISTS my_datalake_db.sales_partitioned;
DROP TABLE IF EXISTS my_datalake_db.sales_parquet_partitioned;
```

### 3. Terminate EC2 Instance
```bash
# Find instance ID
aws ec2 describe-instances --filters "Name=tag:Name,Values=athena-data-generator" --query "Reservations[*].Instances[*].InstanceId"

# Terminate
aws ec2 terminate-instances --instance-ids i-xxxxxxxxx
```

Or via Console: EC2 → Instances → Select → Instance State → Terminate

### 4. Delete IAM Role (if created)
```bash
aws iam detach-role-policy --role-name EC2-S3-Access --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam delete-role --role-name EC2-S3-Access
```

---

## Summary & Next Steps

**What you practiced:**
✓ SQL fundamentals (filtering, aggregations, joins)
✓ Creating and managing partitioned tables
✓ CTAS for format conversion and optimization
✓ End-to-end pipeline: EC2 → S3 → Athena
✓ Performance comparison (CSV vs Parquet, partitioned vs non-partitioned)

**Skills gained:**
- Data engineering fundamentals
- Cost optimization techniques
- Real-world pipeline patterns
- Athena best practices

**Next steps:**
1. Review solutions if you got stuck
2. Experiment with larger datasets
3. Build an automated pipeline (Glue + Lambda)
4. Connect Athena to BI tools (QuickSight, Tableau)
5. Explore AWS Glue Crawlers for schema discovery

---

## Additional Challenge Exercises

### Challenge 1: Incremental Data Loading
- Upload new data daily to S3
- Add new partitions automatically with Lambda
- Query only new data since last run

### Challenge 2: Data Quality Checks
- Create SQL queries that detect anomalies
- Set up CloudWatch alarms on query results
- Build data quality dashboard

### Challenge 3: Cost Optimization Competition
- Start with 10 GB CSV dataset
- Apply all optimization techniques
- Measure final cost reduction percentage

### Challenge 4: Multi-Format Data Lake
- Mix CSV, JSON, and Parquet in same table
- Use Glue Crawlers for schema inference
- Compare query performance across formats

---

## Resources

**Sample Datasets:**
- All exercise datasets in `../assets/` folder
- AWS Open Data Registry: https://registry.opendata.aws/
- NYC Taxi Data: https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page

**Documentation:**
- [Athena SQL Reference](https://docs.aws.amazon.com/athena/latest/ug/ddl-sql-reference.html)
- [Partitioning Best Practices](https://docs.aws.amazon.com/athena/latest/ug/partitions.html)
- [CTAS Examples](https://docs.aws.amazon.com/athena/latest/ug/ctas-examples.html)

**Solutions:**
See [solution_enhanced.md](../solutions/solution_enhanced.md) for complete solutions and explanations.
