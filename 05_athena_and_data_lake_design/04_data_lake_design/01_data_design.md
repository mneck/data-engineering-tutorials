# Data Lake Design on AWS (S3 + Glue + Athena)

## TL;DR
Build a serverless data lake where you store raw data in S3, automatically catalog it with Glue crawlers, and query it with SQL using Athena—all without managing databases or servers. You pay only for storage and queries executed.

**Key takeaway:** This architecture lets you analyze any volume of data using familiar SQL, without ETL pipelines or database administration.

---

## Big Picture: What, Why, When

### What is a Data Lake?
A data lake is centralized storage for **all your data** (structured, semi-structured, unstructured) in its raw format. Unlike traditional databases that require predefined schemas, data lakes accept any data type: CSVs, JSONs, logs, images, videos.

**The AWS Data Lake Stack:**
- **S3** = Storage layer (infinitely scalable, dirt cheap)
- **Glue** = Metadata catalog (understands what's in your S3 files)
- **Athena** = Query engine (SQL interface to S3 data)

### Why Use a Data Lake?
**Traditional approach problems:**
1. Databases are expensive and don't scale well for analytical workloads
2. You must define schemas upfront (rigid)
3. ETL pipelines are complex to build and maintain
4. Storage costs are high for rarely-accessed data

**Data lake benefits:**
1. **Schema-on-read:** Store now, define structure later when you need to query
2. **Cost-effective:** S3 storage is ~$0.023/GB/month (Standard class)
3. **Scalable:** Store petabytes without provisioning capacity
4. **Flexible:** Mix CSV, JSON, Parquet, logs—all in one place
5. **Separation of storage and compute:** Pay for queries only when you run them

### When to Use This Architecture?

**Good fit:**
- Storing application logs, clickstream data, IoT sensor data
- Data warehouse source (S3 as staging before Redshift/Snowflake)
- Analytics on historical data that doesn't need real-time updates
- Exploratory data analysis on diverse data sources
- Compliance/audit logs that must be retained long-term

**Not ideal for:**
- Transactional workloads (use RDS/DynamoDB instead)
- Real-time analytics requiring sub-second latency (use Kinesis + DynamoDB)
- Frequent small updates to individual records (Athena is read-optimized)

---

## Real-World Use Case: E-commerce Analytics

**Scenario:** You run an online store and want to analyze:
- Customer purchase patterns
- Product performance across regions
- Marketing campaign effectiveness

**Traditional approach:**
- Set up PostgreSQL RDS database
- Build ETL jobs to load sales data hourly
- Provision enough capacity for peak query loads
- Cost: ~$200/month minimum (RDS + compute)

**Data lake approach:**
- Application writes sales events as JSON to `s3://sales-data-lake/raw/year=2025/month=11/`
- Glue crawler runs nightly, detects new partitions
- Analysts query with Athena: `SELECT product, COUNT(*) FROM sales WHERE year=2025 AND month=11`
- Cost: $5/month storage + $5/TB scanned (only when querying)

**Savings:** 90%+ for analytical workloads

---

## Prerequisites

### IAM Permissions Required
Your IAM user needs these policies:
- `AmazonS3FullAccess` - Create buckets, upload data
- `AWSGlueConsoleFullAccess` - Create databases, crawlers, view catalog
- `AmazonAthenaFullAccess` - Run SQL queries

**Verify your permissions:**
```bash
# Check your IAM user/groups
aws iam list-attached-user-policies --user-name <your-username>
```

### Cost Expectations
For this tutorial with sample data:
- S3 storage: ~$0.001 (negligible)
- Glue crawler: $0.44/hour (runs for ~1 minute = $0.007)
- Athena queries: $5/TB scanned (sample data = $0.00001)

**Total cost:** < $0.01

---

## Step-by-Step Implementation

### Step 1: Create S3 Buckets

**Why two buckets?**
1. **Raw data bucket:** Stores your actual datasets
2. **Query results bucket:** Athena needs a place to save query outputs

#### Create the Raw Data Bucket

1. Open [S3 Console](https://s3.console.aws.amazon.com/s3/)
2. Click **Create bucket**
3. Configure:
   - **Bucket name:** `my-datalake-raw-<your-name>-<random-number>`
     - Must be globally unique across all AWS accounts
     - Example: `my-datalake-raw-sasan-92847`
   - **Region:** Choose `us-east-1` (or your preferred region)
     - **Important:** Use the same region for S3, Glue, and Athena
   - **Block Public Access:** Keep all checkboxes enabled (data is private)
   - **Bucket Versioning:** Disabled (for now)
   - **Default encryption:** Server-side encryption with Amazon S3 managed keys (SSE-S3)
4. Click **Create bucket**

#### Create the Query Results Bucket

Repeat the same process:
- **Bucket name:** `my-datalake-query-results-<your-name>-<random-number>`
- Same region as raw bucket
- Keep public access blocked

**Why separate buckets?**
- Query results are temporary/cache-like data
- You can apply different lifecycle policies (delete query results after 7 days)
- Clear separation of concerns

---

### Step 2: Upload Sample Data

#### Organize with Folders (S3 Prefixes)

Best practice: Structure your data with meaningful prefixes
```
my-datalake-raw-bucket/
  ├── sales/                    ← Data domain
  │   ├── year=2025/            ← Partition by year
  │   │   ├── month=01/         ← Partition by month
  │   │   │   └── sales.csv
  │   │   └── month=02/
  │   │       └── sales.csv
  ├── customers/
  │   └── customers.csv
  └── products/
      └── products.json
```

**Why partitions?** When you query `WHERE year=2025 AND month=01`, Athena only scans that folder—not the entire bucket. This saves time and money.

#### Upload the Sales Data

1. In your raw bucket, click **Create folder**
2. Folder name: `sales`
3. Click inside `sales/`, then **Upload**
4. Click **Add files** and upload this CSV:

**File: `sales_data.csv`**
```csv
id,customer,product,amount,date
1,Alice,Laptop,1200,2025-01-15
2,Bob,Mouse,25,2025-01-16
3,Charlie,Keyboard,75,2025-01-16
4,Alice,Monitor,300,2025-01-17
5,David,Laptop,1200,2025-01-18
6,Bob,Keyboard,75,2025-01-19
7,Eve,Mouse,25,2025-01-20
8,Charlie,Monitor,300,2025-01-21
```

5. Click **Upload**
6. Verify: You should see `s3://my-datalake-raw-<name>/sales/sales_data.csv`

**File format tip:** CSV is human-readable but inefficient. For production, use Parquet (columnar format, 10x faster queries, lower costs). For learning, CSV is perfect.

---

### Step 3: Create Glue Database

**What's a Glue Database?**
It's a logical container for table metadata—think of it as a namespace. It doesn't store actual data, just the schema information (columns, data types, location).

1. Open [AWS Glue Console](https://console.aws.amazon.com/glue/)
2. In left menu: **Data Catalog → Databases**
3. Click **Add database**
4. Settings:
   - **Name:** `ecommerce_datalake`
   - **Description:** "E-commerce data lake catalog"
   - **Location:** Leave empty (not required)
5. Click **Create**

**You now have a database!** It's empty (no tables yet). The crawler will populate it.

---

### Step 4: Create a Glue Crawler

**What does a crawler do?**
1. Connects to your S3 bucket
2. Reads a sample of files to infer schema (column names, data types)
3. Creates/updates table definitions in the Glue Data Catalog
4. Detects partitions automatically

**Think of it as:** Glue reads the first few rows of your CSV, figures out you have columns `id`, `customer`, `amount`, etc., and registers this as a table.

#### Create the Crawler

1. In Glue Console: **Data Catalog → Crawlers**
2. Click **Create crawler**

**Page 1: Set crawler properties**
- **Name:** `sales-data-crawler`
- Click **Next**

**Page 2: Choose data sources and classifiers**
- **Data source:** S3
- **S3 path:** Browse and select `s3://my-datalake-raw-<name>/sales/`
  - **Important:** Select the `sales/` folder, not the root bucket
- **Subsequent crawler runs:** Crawl all sub-folders
- Click **Next**

**Page 3: Configure security settings**
- **IAM role:**
  - If first time: Choose **Create new IAM role**
  - Role name: `AWSGlueServiceRole-DataLake`
  - Glue automatically creates a role with S3 read permissions
- Click **Next**

**Page 4: Set output and scheduling**
- **Target database:** Select `ecommerce_datalake`
- **Table name prefix:** Leave empty (recommended)
  - **Why skip the prefix?** Glue table naming is confusing with prefixes
  - **Without prefix:** Table name = folder name (`sales`)
  - **With prefix:** Table name could be `sales_sales`, `sales_sales_data`, or `sales_0_csv` depending on your file structure
  - **Result is unpredictable** (see "Understanding Glue Table Naming" section below)
- **Crawler schedule:** On demand (run manually for now)
  - In production, schedule daily/hourly to detect new data
- **Configuration options:** Leave defaults
- Click **Next**

**Page 5: Review and create**
- Review settings
- Click **Create crawler**

#### Run the Crawler

1. In the Crawlers list, select `sales-data-crawler`
2. Click **Run**
3. Status changes: Ready → Running → Completed (~1-2 minutes)
4. Check **Tables added:** Should show `1`

**What just happened?**
Glue read your `sales_data.csv`, inferred:
- Column `id` is `bigint`
- Column `customer` is `string`
- Column `amount` is `bigint`
- Column `date` is `string`

It created a table in your `ecommerce_datalake` database. The table name depends on your configuration (see "Understanding Glue Table Naming" below).

---

### Step 5: Verify the Table in Glue

1. Go to **Data Catalog → Tables**
2. You should see your table (name will vary based on your setup)
3. Click on the table to view:
   - **Schema:** Column names and types
   - **Location:** S3 path
   - **Input format:** CSV
   - **SerDe:** OpenCSVSerDe (how to parse the CSV)

**If the schema is wrong:**
- You can manually edit the table
- Or delete it and re-run the crawler with different settings

---

## Understanding Glue Table Naming (Important!)

**Why is table naming confusing?**

Glue crawler naming logic depends on your file structure and schemas. Here's what actually happens:

### Scenario 1: Single schema in folder (cleanest approach)
```
s3://bucket/sales/
  ├── jan.csv        (schema: id, customer, amount, date)
  ├── feb.csv        (same schema)
  └── mar.csv        (same schema)
```

**Result:** ONE table named `sales` (folder name)
- All files with compatible schemas merged into one table
- When you add more files with the same schema, crawler updates this table
- **This is the recommended approach**

### Scenario 2: Files with similar naming pattern
```
s3://bucket/sales/
  ├── sales_data.csv      (schema: id, customer, amount, date)
  └── sales_data_ext.csv  (same schema)
```

**Result:** ONE table named `sales` or `sales_sales` (if prefix used)
- Common file prefix detected (`sales_data*.csv`)
- Files merged into one table
- Adding more `sales_data_*.csv` files updates the same table ✓

### Scenario 3: Multiple different schemas (causes problems!)
```
s3://bucket/sales/
  ├── sales_data.csv    (schema A: id, customer, amount)
  └── customers.csv     (schema B: customer, email, country)
```

**Result:** MULTIPLE tables created:
- `sales_sales_data` or `sales_data` (from sales_data.csv)
- `sales_customers` or `sales_0_tsv` (from customers.csv)
- Table names become unpredictable
- If you had an existing `sales_sales` table, it may be marked "deprecated"

### Scenario 4: Different file formats in same folder
```
s3://bucket/sales/
  ├── data.csv
  └── data.tsv
```

**Result:** Multiple tables like `sales_0_csv`, `sales_1_tsv` or similar indexed names

### The Impact of Table Prefix

**Without prefix (recommended):**
- Table name = folder name (`sales`)
- Clean, predictable naming

**With prefix `sales_`:**
- Table name = `sales_` + inferred name
- Could be: `sales_sales`, `sales_sales_data`, `sales_data`, or `sales_0_csv`
- **Unpredictable and redundant**

### Best Practices to Avoid Confusion

**1. One schema per folder:**
```
s3://bucket/
  ├── sales/          ← All sales CSVs (same schema)
  ├── customers/      ← All customer CSVs (same schema)
  └── products/       ← All product JSONs (same schema)
```
Result: Clean table names that match folder names

**2. Skip the table prefix:**
- Leave "Table name prefix" empty when creating crawler
- Tables will be named after folders (cleaner)

**3. Keep file schemas consistent:**
- All files in a folder should have the exact same columns
- Use the same data types across files
- If you need different schemas, use different folders

**4. If you get unexpected tables:**
- Check if files in the same folder have different schemas
- Delete the tables in Glue Console
- Reorganize S3 structure (one schema per folder)
- Re-run crawler

### What Happens When You Add More Files?

**Same schema → Updates existing table ✓**
```
# Initial: sales_data.csv
# Crawler creates: sales table

# Add: sales_data_ext.csv (same schema)
# Re-run crawler
# Result: sales table is updated (includes both files)
```

**Different schema → Creates new table**
```
# Initial: sales_data.csv (columns: id, customer, amount)
# Crawler creates: sales_sales_data table

# Add: customers.csv (columns: customer, email, country)
# Re-run crawler
# Result: NEW table created (sales_customers)
# Old table may be marked "deprecated" if schema changed
```

### Real-World Example (Your Case)

You added `sales_data_ext.csv` with the same schema as `sales_data.csv`:
- Crawler correctly updated the existing table
- Both files now appear in the same `sales_sales` table
- **This is correct behavior** ✓

Your friend added a different file with a different schema:
- Crawler detected incompatible schemas
- Created multiple tables: `sales_sales_data`, `sales_0_tsv`, etc.
- Deprecated the original table
- **This happens when schemas don't match**

---

### Step 6: Query with Amazon Athena

**What is Athena?**
Serverless SQL query engine. You write standard SQL, Athena:
1. Reads table metadata from Glue
2. Scans the S3 files
3. Executes the query in-memory
4. Returns results

**No servers to manage. Pay only for data scanned.**

#### First-Time Athena Setup

1. Open [Athena Console](https://console.aws.amazon.com/athena/)
2. If first time, you'll see: "Before you run your first query, you need to set up a query result location in Amazon S3"
3. Click **Edit settings** (or **Manage** in the top banner)
4. Settings:
   - **Query result location:** `s3://my-datalake-query-results-<name>/athena-output/`
   - Click **Save**

**What are query results?**
Athena saves every query output as a CSV in this bucket. You can download them or ignore them.

#### Run Your First Query

1. In Athena Query Editor:
   - **Database:** Select `ecommerce_datalake` from dropdown
   - You'll see your table in the left panel (e.g., `sales`, `sales_sales`, or `sales_sales_data`)
2. In the query editor, type (replace `your_table_name` with your actual table name):

```sql
SELECT * FROM your_table_name LIMIT 10;
```

**Example with different possible table names:**
```sql
-- If your table is named "sales"
SELECT * FROM sales LIMIT 10;

-- If your table is named "sales_sales" (with prefix)
SELECT * FROM sales_sales LIMIT 10;

-- If your table is named "sales_sales_data"
SELECT * FROM sales_sales_data LIMIT 10;
```

3. Click **Run**
4. Results appear below (should show your 8 rows)

**What happened?**
- Athena read the Glue catalog to understand table structure
- Scanned `s3://my-datalake-raw-<name>/sales/sales_data.csv`
- Parsed CSV and returned first 10 rows

**Data scanned:** ~500 bytes (shown in query stats)
**Cost:** $0.000000025 (yes, 25 billionths of a dollar)

#### Run an Analytical Query

```sql
SELECT customer, SUM(amount) as total_spent
FROM your_table_name  -- Replace with your actual table name
GROUP BY customer
ORDER BY total_spent DESC;
```

**Results:**
| customer | total_spent |
|----------|-------------|
| Alice    | 1500        |
| Charlie  | 375         |
| David    | 1200        |
| Bob      | 100         |
| Eve      | 25          |

**Business insight:** Alice is your top customer—maybe send her a loyalty discount!

#### More Example Queries

**Count orders per product:**
```sql
SELECT product, COUNT(*) as order_count, SUM(amount) as revenue
FROM your_table_name  -- Replace with your actual table name
GROUP BY product
ORDER BY revenue DESC;
```

**Filter by date range:**
```sql
SELECT * FROM your_table_name
WHERE date BETWEEN '2025-01-16' AND '2025-01-20';
```

**Average order value:**
```sql
SELECT AVG(amount) as avg_order_value FROM your_table_name;
```

**Pro tip:** To avoid typing the table name repeatedly, you can find it in the Athena left sidebar under your database. Click the three dots next to the table name and select "Insert table name" to auto-populate it in your query.

---

## Understanding the Data Flow

```
[Your Application/ETL]
    ↓ writes data
[S3 Bucket] (raw data storage)
    ↓ Glue Crawler scans
[Glue Data Catalog] (metadata: schema, location)
    ↓ Athena reads catalog
[Athena Query Engine]
    ↓ scans S3 files on-demand
[Query Results] (returned to you, saved to results bucket)
```

**Key insight:** Data never moves. It stays in S3. Glue just describes it. Athena reads it directly.

---

## Common Issues & Troubleshooting

### Issue 1: "Table not found" in Athena
**Cause:** You selected the wrong database in the dropdown.
**Fix:** Select `ecommerce_datalake` from the database dropdown.

### Issue 2: Crawler finds 0 tables
**Cause:**
- S3 path is wrong (pointed to wrong folder)
- No files in the folder
- IAM role doesn't have S3 read permissions

**Fix:**
- Verify S3 path in crawler configuration
- Check IAM role has `s3:GetObject` permission

### Issue 3: Query returns 0 rows but file has data
**Cause:** CSV header row is being treated as data, or SerDe is misconfigured.
**Fix:**
- Edit table in Glue → Table properties → Add: `skip.header.line.count = 1`
- Re-run crawler with "Update all new and existing partitions"

### Issue 4: Data types are wrong (all strings)
**Cause:** Glue crawler's type inference isn't perfect.
**Fix:**
- Manually edit table schema in Glue
- Or convert types in Athena: `CAST(amount AS INTEGER)`

### Issue 5: "Insufficient permissions" when running Athena query
**Cause:** Your IAM user can't write to query results bucket.
**Fix:**
- Add `s3:PutObject` permission for the results bucket
- Or create a new results bucket and update Athena settings

---

## Cost Breakdown

For this tutorial:
- **S3 storage:** 1 KB = $0.00000002/month
- **Glue crawler:** $0.44/DPU-hour, ran for 0.02 hours = $0.009
- **Athena queries:** $5/TB, scanned 0.000001 TB = $0.000005
- **Total:** ~$0.01

**At scale (1 TB of data, 100 queries/day):**
- S3 storage: $23/month
- Glue crawler (daily): $0.44 × 30 = $13/month
- Athena: $5 × 100 queries × 0.01 TB scanned = $50/month
- **Total:** ~$86/month

**Compare to managed database:**
- RDS db.m5.large with 1 TB storage: ~$300/month
- Data warehouse (Redshift): ~$180/month minimum

---

## Best Practices for Production

### 1. Use Partitioning
Organize data by date/region/category:
```
s3://bucket/sales/year=2025/month=01/day=15/data.csv
```

Query only what you need:
```sql
WHERE year=2025 AND month=1  -- Scans only January folder
```

**Benefit:** Query only 1/12 of data = 1/12 the cost

### 2. Use Columnar Formats (Parquet/ORC)
Convert CSV to Parquet:
- 5-10x smaller file size
- 10-100x faster queries (reads only needed columns)
- Lower Athena costs

**Example:** 1 GB CSV → 100 MB Parquet → 90% cost reduction

### 3. Compress Your Data
Use Snappy or GZIP compression:
- Smaller S3 storage costs
- Faster network transfer
- Athena decompresses automatically

### 4. Limit Athena Queries
- Use `LIMIT` for exploratory queries
- Avoid `SELECT *` in production—specify columns
- Set workgroup query limits (max bytes scanned)

### 5. Set Up Lifecycle Policies
Move old data to cheaper storage:
- Day 0-30: S3 Standard
- Day 30-90: S3 Standard-IA (40% cheaper)
- Day 90+: S3 Glacier (80% cheaper)

### 6. Monitor Costs
- Enable **S3 Storage Lens** for usage analytics
- Set up **AWS Budgets** with alerts ($50/month threshold)
- Tag resources: `Project=DataLake`, `Env=Prod`

---

## Next Steps

1. **Add more datasets:** Upload `customers.csv`, `products.json`
2. **Practice joins:** Combine sales + customers data
3. **Set up partitions:** Reorganize data by date
4. **Convert to Parquet:** Use Glue ETL jobs or AWS SDK
5. **Build dashboards:** Connect Athena to QuickSight for visualization
6. **Automate data ingestion:** Use Lambda to trigger on S3 uploads

---

## Summary

**What you built:**
- ✅ S3 data lake with organized folders
- ✅ Glue Data Catalog with automated schema discovery
- ✅ Athena SQL queries on S3 files
- ✅ Serverless architecture (no servers to manage)
- ✅ Pay-per-query pricing (cost-effective)

**Key concepts:**
- **S3** = Cheap, scalable storage
- **Glue** = Schema registry + ETL
- **Athena** = SQL interface to S3

**This is the foundation of modern data engineering on AWS.**

---

**Next:** [Data Retention Policies →](./02_data_retention_policy.md)
