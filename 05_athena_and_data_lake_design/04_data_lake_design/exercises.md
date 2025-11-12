# Data Lake Design - Practice Exercises

These exercises build on the core concepts from the lecture notes. They're organized by difficulty and topic.

**How to use this guide:**
- Start with Section A if you're comfortable with basics
- Each exercise includes: context, learning goal, step-by-step hints
- Solutions are available in [solutions.md](./solutions.md)
- Don't peek at solutions immediately—struggle is part of learning!

---

## Section A: Data Lake Extensions (Athena + Glue)

### Exercise 1: Partition Athena Table by Date

**Why this matters:** Partitioning is the #1 way to reduce Athena query costs. If you have 1 TB of data and query only January, partitioning means you scan 83 GB (1/12) instead of 1 TB. That's 12× cost savings.

**Real-world context:** E-commerce companies partition sales data by `year/month/day`. When analyzing "yesterday's sales," they scan 1 day of data, not the entire history.

**Learning goal:** Understand how partitions work and how to create partitioned tables in Athena.

**Prerequisites:**
- You have `sales_data.csv` uploaded to S3
- Glue crawler has created a table

**Task:**
1. Reorganize your S3 data with partitions:
   ```
   s3://bucket/sales/year=2025/month=01/sales_jan.csv
   s3://bucket/sales/year=2025/month=02/sales_feb.csv
   ```
2. Create a new Athena table with `PARTITIONED BY (year STRING, month STRING)`
3. Run `MSCK REPAIR TABLE` or `ALTER TABLE ADD PARTITION` to register partitions
4. Query only January data and verify Athena scans less data

**Hints:**
- Use `CREATE EXTERNAL TABLE` with `PARTITIONED BY` clause
- Partitions must be in the format `key=value` in S3 paths
- Check "Data scanned" in query results to verify partition pruning works

**Expected result:** Query for January scans only January data (check "Data scanned" metric).

---

### Exercise 2: Query Only One Customer in Athena

**Why this matters:** Filtering is basic but critical. In production, you rarely want all data—you want specific customers, date ranges, or product categories.

**Real-world context:** Customer support needs to pull transaction history for one customer. Without filtering, you'd scan millions of rows unnecessarily.

**Learning goal:** Practice SQL WHERE clauses and understand how they reduce data scanned.

**Task:**
Write a query to return only sales for customer `Alice`.

**Hints:**
```sql
SELECT * FROM sales_data WHERE customer = ?
```

**Challenge:** Modify the query to show Alice's total spending and order count.

**Expected result:**
| customer | total_spent | order_count |
|----------|-------------|-------------|
| Alice    | 1890        | 5           |

---

### Exercise 3: Count Number of Rows with Athena

**Why this matters:** `COUNT(*)` is the most basic aggregation. In data engineering, you use it constantly to validate data pipelines ("Did all 1 million rows load?").

**Real-world context:** After an ETL job runs, you verify row counts match between source and destination.

**Task:**
1. Count total rows in `sales_data`
2. Count rows per customer
3. Count distinct customers

**Hints:**
```sql
-- Total rows
SELECT COUNT(*) FROM sales_data;

-- Rows per customer
SELECT customer, COUNT(*) as order_count FROM sales_data GROUP BY customer;

-- Distinct customers
SELECT COUNT(DISTINCT customer) FROM sales_data;
```

**Expected result:** You should have 20 total rows, 5 distinct customers.

---

### Exercise 4: Join Two Tables in Athena

**Why this matters:** Real-world data is always split across multiple tables. Sales data doesn't include customer emails—you have to join with a customer table.

**Real-world context:** Marketing wants to email customers who spent >$1000. You join sales (amounts) with customers (emails).

**Learning goal:** Practice SQL JOINs in Athena, understand how Glue catalogs multiple tables.

**Prerequisites:**
- Upload `customers.csv` to S3 (`s3://bucket/customers/`)
- Run Glue crawler to create `customers` table

**Task:**
1. Create `customers` table via Glue crawler
2. Write a JOIN query to combine sales and customer data
3. Find total spending per customer with their email addresses

**Hints:**
```sql
SELECT s.customer, c.email, c.country, SUM(s.amount) as total_spent
FROM sales_data s
JOIN customers c ON s.customer = c.customer
GROUP BY s.customer, c.email, c.country
ORDER BY total_spent DESC;
```

**Challenge:** Filter to show only customers from the USA who spent >$500.

**Expected result:**
| customer | email | country | total_spent |
|----------|-------|---------|-------------|
| Alice    | alice@example.com | USA | 1890 |

---

### Exercise 5: Visualize Athena Results in QuickSight

**Why this matters:** Data engineers deliver insights to business users. QuickSight connects directly to Athena, allowing non-technical users to explore data visually.

**Real-world context:** Your CEO wants a dashboard showing "sales by product." Instead of sending Excel exports weekly, you build a QuickSight dashboard that auto-updates.

**Learning goal:** Connect BI tools to Athena, create basic visualizations.

**Prerequisites:**
- Athena table with sales data
- QuickSight account (free trial available)

**Task:**
1. Open QuickSight Console
2. Create new dataset → Athena
3. Select `sales_data` table
4. Create visualizations:
   - Bar chart: Total sales by customer
   - Pie chart: Sales distribution by product
   - Line chart: Sales over time (if you have date field)

**Hints:**
- For bar chart: Drag `customer` to X-axis, `amount` to Y-axis (use SUM aggregation)
- For pie chart: Drag `product` to Group/Color, `amount` to Value
- Choose "Direct Query" to always show latest data (vs importing to SPICE)

**Challenge:** Add a filter to show only sales >$100.

**Expected result:** Interactive dashboard you can share via URL with stakeholders.

---

## Section B: Lifecycle Policy Variations

### Exercise 6: Prefix-Specific Rule

**Why this matters:** You don't want the same retention policy for all data. Logs should be deleted after 90 days, but customer data must be kept forever.

**Real-world context:** GDPR requires deleting personal data after 1 year, but financial records must be kept for 7 years. Use different prefixes (`gdpr/`, `financial/`) with different lifecycle rules.

**Learning goal:** Apply lifecycle rules to specific folders using prefix filters.

**Task:**
1. Create folders in S3: `logs/`, `backups/`, `user-data/`
2. Upload test files to `logs/`
3. Create lifecycle rule that applies ONLY to `logs/` prefix
4. Rule: Delete after 30 days

**Hints:**
- In lifecycle rule creation, choose "Limit the scope of this rule"
- Prefix: `logs/`
- Expiration: 30 days

**Verification:** Upload a file to `logs/` and another to `backups/`. Only the logs file should be scheduled for deletion.

**Expected result:** Lifecycle rule summary shows `Scope: logs/*`.

---

### Exercise 7: Tag-Based Lifecycle

**Why this matters:** Sometimes you can't organize data by folder (e.g., mixed archive/active files in same prefix). Tags let you mark individual objects for archival.

**Real-world context:** Users upload documents. Some are marked "Archive" by the app. You want to move archived docs to Glacier without affecting active docs.

**Learning goal:** Use object tags to selectively apply lifecycle rules.

**Task:**
1. Upload a file to S3
2. Add tag: `Archive=true`
3. Create lifecycle rule that applies only to objects with tag `Archive=true`
4. Rule: Immediately transition to Glacier (0 days)

**Hints:**
- Select file → Properties → Tags → Add tag
- Lifecycle rule scope: "Limit by object tags"
- Tag filter: Key=`Archive`, Value=`true`
- Transition to Glacier Flexible Retrieval after 0 days

**Verification:** Upload two files—tag one with `Archive=true`. Check storage class after 24-48 hours (only tagged file moves to Glacier).

**Expected result:** Tagged file shows storage class "Glacier Flexible Retrieval" in S3 console.

---

### Exercise 8: Non-Current Version Cleanup

**Why this matters:** Versioning protects against accidental deletes, but old versions pile up. A file edited 100 times = 100 versions = 100× storage cost.

**Real-world context:** Your app overwrites `config.json` daily. After 30 days, you have 30 versions of the same file. You only need the latest + last 7 versions for rollback.

**Learning goal:** Use lifecycle rules to delete old versions while keeping current version.

**Prerequisites:**
- Enable bucket versioning

**Task:**
1. Enable versioning on your bucket
2. Upload `data.csv`
3. Modify and re-upload it 3-4 times (creates multiple versions)
4. Create lifecycle rule: "Delete noncurrent versions after 30 days"

**Hints:**
- Bucket → Properties → Bucket Versioning → Enable
- Lifecycle rule actions: "Expire noncurrent versions of objects"
- Days after objects become noncurrent: `30`

**Verification:**
- Upload `test.txt` (version 1)
- Modify and re-upload (version 2 = current, version 1 = noncurrent)
- Check versions: Select file → Versions tab
- After lifecycle rule runs, old versions disappear

**Expected result:** Current version remains, noncurrent versions deleted after 30 days.

---

### Exercise 9: Abort Incomplete Multipart Uploads

**Why this matters:** Multipart uploads (for files >5 GB) can fail mid-upload. Incomplete parts remain in your bucket (invisible but billable). Over time, this waste accumulates.

**Real-world context:** A data pipeline uploads 10 GB files to S3. If the pipeline crashes mid-upload, incomplete parts stay in S3 indefinitely, costing money.

**Learning goal:** Clean up failed uploads automatically.

**Task:**
1. Create lifecycle rule to abort incomplete multipart uploads after 7 days

**Hints:**
- Lifecycle rule actions: "Delete expired object delete markers or incomplete multipart uploads"
- Days after initiation: `7`

**Verification (advanced):**
```bash
# Start a multipart upload (don't complete it)
aws s3api create-multipart-upload --bucket my-bucket --key testfile.txt

# List incomplete uploads
aws s3api list-multipart-uploads --bucket my-bucket
```

After 7 days, re-run the list command—upload should be gone.

**Expected result:** Lifecycle rule prevents accumulation of incomplete upload parts.

---

### Exercise 10: Combine Multiple Actions

**Why this matters:** Production lifecycle rules chain multiple transitions. Data moves through storage tiers as it ages: Standard → IA → Glacier → Delete.

**Real-world context:** Security logs are accessed frequently for 7 days, occasionally for 30 days, archived for 1 year (compliance), then deleted.

**Learning goal:** Create a multi-stage lifecycle rule.

**Task:**
Create a single lifecycle rule with:
- Day 30: Move to Standard-IA
- Day 180: Move to Glacier Flexible Retrieval
- Day 365: Delete

**Hints:**
- One rule can have multiple transitions (add each one separately)
- Transitions must be in ascending order (30 → 180 → 365)

**Verification:** Check rule summary shows all three actions in timeline format.

**Expected result:**
```
Day 0:    S3 Standard
Day 30:   S3 Standard-IA
Day 180:  Glacier Flexible Retrieval
Day 365:  Deleted
```

---

## Section C: Cost Explorer & Budgets Deep Dive

### Exercise 11: Filter by Region in Cost Explorer

**Why this matters:** Different regions have different pricing. If you accidentally launch resources in expensive regions, costs spike.

**Real-world context:** Developer creates EC2 in `ap-southeast-2` (Sydney, 30% more expensive) instead of `us-east-1` (cheapest). Cost Explorer helps identify this.

**Learning goal:** Use Cost Explorer filters to diagnose regional cost issues.

**Task:**
1. Open Cost Explorer
2. Filter costs to show only `us-east-1` region
3. Compare costs across multiple regions

**Hints:**
- Cost Explorer → Filters → Add filter → Region
- Select `US East (N. Virginia)`
- Group by: Region (to compare all regions)

**Challenge:** Identify your most expensive region. If it's not `us-east-1` and you don't need that location, consider migrating.

**Expected result:** Pie chart or bar chart showing costs per region.

---

### Exercise 12: Group by Linked Account Tags

**Why this matters:** Tags enable cost attribution. Without tags, you can't answer "How much does the data lake project cost?" or "Which team is spending the most?"

**Real-world context:** Your company has 3 teams using AWS. Each tags resources with `Team=DataEng`, `Team=WebDev`, etc. Finance uses Cost Explorer to allocate costs back to each team's budget.

**Learning goal:** Use tags for cost tracking and chargeback.

**Prerequisites:**
- Tag your resources (EC2, S3, etc.) with `Project=DataLake`
- Activate tags in Billing Console

**Task:**
1. Tag resources:
   - S3 bucket → Properties → Tags → `Project=DataLake`
   - EC2 instance → Tags → `Project=WebApp`
2. Activate tags:
   - Billing Console → Cost Allocation Tags
   - Select `Project` → Activate
3. Wait 24 hours for data to populate
4. Cost Explorer → Group by → Tag → Project

**Hints:**
- Tags must be activated before they appear in Cost Explorer
- Only costs incurred AFTER activation are tagged

**Challenge:** Create a saved report showing monthly costs per project.

**Expected result:** Cost breakdown showing `Project=DataLake` vs `Project=WebApp`.

---

### Exercise 13: Export Cost Data to CSV Automatically

**Why this matters:** Cost Explorer is good for visualization, but sometimes you need raw data for analysis in Python/Excel/Tableau.

**Real-world context:** Finance team wants monthly cost reports in Excel. Instead of manually downloading each month, set up automatic CSV exports.

**Learning goal:** Export Cost Explorer reports for offline analysis.

**Task:**
1. Create a Cost Explorer report (e.g., "Monthly by Service")
2. Download as CSV
3. Open in Excel/Google Sheets and analyze

**Hints:**
- Save report in Cost Explorer
- Click "Download CSV" (top right)
- For automation: Use Cost & Usage Report (CUR) to S3 (covered in advanced lectures)

**Challenge:** Write a Python script to parse the CSV and calculate month-over-month growth.

**Expected result:** CSV file with columns: TimePeriod, Service, Cost.

---

### Exercise 14: Enable Cost Anomaly Detection

**Why this matters:** Manual cost monitoring doesn't catch spikes in real-time. Cost Anomaly Detection uses ML to alert you within 24 hours of unusual spend.

**Real-world context:** A misconfigured Lambda function calls itself recursively, racking up $500 in 6 hours. Anomaly Detection alerts you immediately instead of waiting for the month-end bill.

**Learning goal:** Set up proactive alerting for cost spikes.

**Task:**
1. Create Cost Anomaly Detection monitor for EC2
2. Set alert threshold to $5
3. Add your email as recipient

**Hints:**
- Cost Management → Cost Anomaly Detection → Create monitor
- Monitor type: AWS Service → EC2
- Subscription threshold: $5
- Notification: Email

**Verification:** Spin up a larger instance (t3.medium) to trigger anomaly. You should get an email within 24-48 hours.

**Expected result:** Email notification when EC2 costs exceed normal patterns by >$5.

---

### Exercise 15: Set a Usage Budget (EC2 Hours)

**Why this matters:** Cost budgets are reactive (you already spent money). Usage budgets are proactive (you're about to exceed Free Tier limits).

**Real-world context:** Free Tier includes 750 EC2 hours/month. If you run 2× t2.micro instances 24/7, that's 1,440 hours = exceeding Free Tier. Usage budget alerts you at 600 hours (80% of 750).

**Learning goal:** Create usage-based budgets to stay within Free Tier.

**Task:**
1. Create a budget for EC2 usage
2. Set limit: 750 hours (Free Tier)
3. Alert at 80% (600 hours)

**Hints:**
- Budgets → Create budget → Usage budget
- Service: Amazon EC2
- Usage type: `BoxUsage:t2.micro` (or your instance type)
- Budgeted amount: 750 hours

**Challenge:** Create a similar budget for S3 storage (5 GB Free Tier limit).

**Expected result:** Email alert when you've used 600 of 750 Free Tier hours.

---

## Section D: EC2 Cost Optimization

### Exercise 16: Identify Idle EC2 with CloudWatch

**Why this matters:** Idle instances (low CPU, no network traffic) waste money. Identifying and stopping them can save 50-90% of EC2 costs.

**Real-world context:** Developer launches EC2 for testing, forgets about it. It runs at 2% CPU for 3 months = $180 wasted.

**Learning goal:** Use CloudWatch metrics to find underutilized instances.

**Task:**
1. Launch a t3.micro instance
2. Let it run idle (don't SSH, don't run apps)
3. Check CloudWatch metrics after 1 hour
4. Identify if CPU utilization is <5%

**Hints:**
- CloudWatch Console → Metrics → EC2 → Per-Instance Metrics
- Select your instance → CPUUtilization
- Set time range: Last 1 hour
- If average <5%, instance is idle

**Challenge:** Set up a CloudWatch alarm to notify you if CPU stays <5% for 24 hours.

**Expected result:** Graph showing consistent low CPU usage.

---

### Exercise 17: Use Spot Instance Pricing Page

**Why this matters:** Spot Instances cost 60-90% less than On-Demand. For non-critical workloads (batch jobs, dev environments), Spot is a massive cost saver.

**Real-world context:** Data pipeline runs nightly for 2 hours. On-Demand t3.large costs $0.10/hour × 2 × 30 days = $6/month. Spot costs $0.02/hour = $1.20/month (80% savings).

**Learning goal:** Understand Spot pricing and when to use it.

**Task:**
1. Go to EC2 Console → Spot Requests → Pricing History
2. Select instance type: `t3.micro`
3. Compare Spot price vs On-Demand price
4. Check Spot interruption rate (how often AWS reclaims the instance)

**Hints:**
- Spot prices fluctuate based on demand
- Interruption rate <5% = safe for most workloads
- Don't use Spot for production databases (use for batch jobs, CI/CD runners)

**Expected result:** Spot price is 60-90% lower than On-Demand.

---

### Exercise 18: Compare Reserved vs On-Demand in Calculator

**Why this matters:** If you run an instance 24/7 for a year, Reserved Instances save 30-60% compared to On-Demand.

**Real-world context:** Production web server runs continuously. On-Demand t3.small costs $180/year. 1-year Reserved costs $108/year (40% savings).

**Learning goal:** Use AWS Pricing Calculator to model cost scenarios.

**Task:**
1. Open [AWS Pricing Calculator](https://calculator.aws/)
2. Add EC2 instance: `t3.small`, `us-east-1`, 24/7
3. Compare:
   - On-Demand pricing
   - 1-year Reserved (No Upfront)
   - 3-year Reserved (All Upfront)

**Hints:**
- Under "Pricing Strategy," toggle between options
- Calculator shows monthly and annual costs
- Reserved Instances require commitment (can't cancel)

**Challenge:** Calculate ROI if you commit to 3-year Reserved but only use it for 2 years.

**Expected result:**
- On-Demand: $15/month
- 1-year Reserved: $10/month (33% savings)
- 3-year Reserved: $6/month (60% savings)

---

### Exercise 19: Check Instance Right-Sizing Recommendations

**Why this matters:** You might be over-provisioned (t3.large when t3.small is enough). AWS analyzes your usage and suggests downsizing.

**Real-world context:** You launched a t3.large "just in case," but it averages 10% CPU. Downsize to t3.small and save 50%.

**Learning goal:** Use AWS recommendations to optimize instance sizes.

**Task:**
1. Run an EC2 instance for 1 week
2. Go to Cost Explorer → Rightsizing Recommendations
3. Review AWS suggestions

**Hints:**
- Recommendations based on CloudWatch metrics (CPU, memory, network)
- AWS suggests downsizing if utilization is consistently low
- Test recommended size in dev before applying to production

**Expected result:** AWS recommends smaller instance type based on actual usage.

---

### Exercise 20: Stop Instances on Schedule

**Why this matters:** Dev/test environments don't need to run 24/7. Stopping them at night and weekends saves 50-75% of EC2 costs.

**Real-world context:** Development server runs 8 AM - 6 PM weekdays (50 hours/week instead of 168). Savings: 70%.

**Learning goal:** Automate start/stop with EventBridge (formerly CloudWatch Events).

**Task:**
1. Create EventBridge rule to stop EC2 at 6 PM daily
2. Create EventBridge rule to start EC2 at 8 AM weekdays

**Hints:**
- EventBridge Console → Rules → Create rule
- Schedule: Cron expression `0 18 * * ? *` (6 PM UTC daily)
- Target: EC2 StopInstances API
- Select instance IDs to stop

**Cron expressions:**
- Stop at 6 PM UTC: `0 18 * * ? *`
- Start at 8 AM UTC weekdays: `0 8 ? * MON-FRI *`

**Important:** Use UTC time. Berlin is UTC+1 (CET) or UTC+2 (CEST), so adjust accordingly.

**Challenge:** Add a Lambda function that checks if instance is idle before stopping (avoid stopping active work).

**Expected result:** Instance automatically stops at 6 PM, starts at 8 AM weekdays.

---

## Bonus: End-to-End Project

**Scenario:** Build a complete e-commerce data lake.

**Requirements:**
1. Upload sales, customers, products data to S3 (partitioned by month)
2. Create Glue crawlers for each dataset
3. Set up lifecycle policies:
   - Sales data: 30 days Standard, 180 days IA, 365 days delete
   - Customer data: Never delete
4. Create Athena queries:
   - Top 10 customers by revenue
   - Monthly revenue trends
   - Product performance by category
5. Build QuickSight dashboard with 3 charts
6. Set up Cost Explorer budget ($20/month) with 80% alert
7. Enable Cost Anomaly Detection for S3 and Athena

**Time estimate:** 3-4 hours

**Deliverables:**
- S3 bucket structure screenshot
- Glue catalog tables list
- Athena query results (top 10 customers)
- QuickSight dashboard link
- Budget configuration screenshot

**This project demonstrates real-world data engineering skills.**

---

## Solutions

Detailed solutions for all exercises are available in [solutions.md](./solutions.md).

**Don't peek until you've tried!** Struggling is how you learn.

---

**Next Steps:**
- Complete exercises in order
- Document your solutions (screenshots, SQL queries)
- Build a portfolio project using these skills
- Share your data lake on GitHub for job applications

**Good luck!**
