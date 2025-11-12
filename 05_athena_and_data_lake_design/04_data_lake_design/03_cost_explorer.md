# AWS Cost Explorer & Billing Dashboard

## TL;DR
Use AWS Cost Explorer to visualize where your money goes, set up budgets with email alerts to prevent surprise bills, and export detailed cost reports for analysis. Essential for avoiding the "oops I left an EC2 running" $500 bill.

**Key takeaway:** You can't optimize what you don't measure. Cost Explorer shows you exactly what's expensive so you can fix it.

---

## Big Picture: What, Why, When

### What are These Tools?

AWS billing is split into several tools, each with a specific purpose:

| Tool | Purpose | Analogy |
|------|---------|---------|
| **Billing Console** | View invoices, payment methods | Your mailbox (bills arrive here) |
| **Cost Explorer** | Visualize spending trends, filter by service/region/tag | Banking app with charts |
| **Budgets** | Set spending limits with alerts | Credit card limit + notifications |
| **Cost & Usage Report (CUR)** | Raw CSV of every transaction | Bank statement export |
| **Cost Anomaly Detection** | ML-based spike alerts | Fraud detection for unusual charges |

### Why This Matters

**The AWS billing horror story:**
- Beginner spins up EC2 instance for testing
- Forgets to stop it
- Uses t3.large instead of t3.micro (Free Tier)
- Runs for 30 days
- Bill: **$62.78** (should have been $0)

**With budgets:**
- Day 2: Email alert "You've spent $5 (50% of budget)"
- Day 4: Alert "You've spent $8 (80% of budget)"
- Immediate action: Stop the instance
- Final bill: **$8** (saved $54)

**Real-world costs can spiral:**
- Accidentally creating NAT Gateway: $32/month
- Leaving RDS instance running: $100-300/month
- Unoptimized S3 storage: $50/month for data you don't need
- Misconfigured Lambda: $1000/month in execution time

### When to Use These Tools?

**Daily/Weekly:**
- Quick check of current month spending (Cost Explorer dashboard)
- Review budget status (are you on track?)

**Monthly:**
- Analyze spending by service (which is most expensive?)
- Identify cost spikes or anomalies
- Review Cost Anomaly Detection alerts

**Quarterly:**
- Deep dive into Cost & Usage Report
- Optimize Reserved Instances / Savings Plans
- Review rightsizing recommendations

**Always:**
- Keep budgets enabled with 50%, 80%, 100% alerts
- Enable Cost Anomaly Detection for production accounts

---

## Real-World Use Case: Freelance Developer

**Scenario:** You're building a SaaS product on AWS.

**Services used:**
- EC2 (web server): $10/month
- RDS (PostgreSQL): $15/month
- S3 (user uploads): $2/month
- Athena (analytics): $1/month
- **Expected total:** $28/month

**Month 1 bill: $147** 😱

**Using Cost Explorer, you discover:**
- EC2: $62 (you used t3.large instead of t3.small)
- RDS: $15 (correct)
- S3: $45 (you stored 2 TB of test data you forgot about)
- Data transfer: $20 (downloading large datasets repeatedly)
- Athena: $5 (querying with `SELECT *` on 100 GB tables)

**Actions taken:**
1. Downsize EC2 to t3.small → save $52/month
2. Delete unused S3 test data → save $43/month
3. Set up lifecycle policy for S3 → save $10/month
4. Limit Athena queries with column selection → save $4/month
5. Use S3 VPC endpoint to avoid data transfer → save $15/month

**New monthly cost:** $23/month (84% reduction)

**This is why Cost Explorer exists.**

---

## Prerequisites

### IAM Permissions

Your IAM user needs billing access (not granted by default):

**Option 1: Root account activates IAM billing access**
1. Log in as root user
2. Go to **Account → IAM User and Role Access to Billing Information**
3. Click **Edit**
4. Check **Activate IAM Access**
5. Update

**Option 2: Attach billing policies to your IAM user**
Your admin needs to attach these policies:
- `Billing` (AWS managed policy) or custom policy with:
  - `ce:*` (Cost Explorer)
  - `budgets:*` (AWS Budgets)
  - `cur:*` (Cost & Usage Reports)
  - `aws-portal:ViewBilling`
  - `aws-portal:ViewUsage`

**Verify your access:**
1. Go to [Billing Console](https://console.aws.amazon.com/billing/)
2. If you see "You need additional permissions," contact your admin

---

## Step-by-Step Implementation

### Step 1: Enable Cost Explorer

**Important:** Cost Explorer is **disabled by default** and must be activated.

1. Open [Billing & Cost Management Console](https://console.aws.amazon.com/billing/)
2. In left menu: **Cost Explorer**
3. Click **Enable Cost Explorer**
4. Wait 24 hours for data to populate

**What happens:**
- AWS starts processing your historical billing data (up to 12 months)
- Cost Explorer becomes available with pre-built reports
- You're charged **$0.01 per API call** for programmatic access (console use is free)

**First-time note:** If your account is new, you'll only see a few days of data. Come back in a week.

---

### Step 2: Explore the Default Dashboard

Once enabled:

1. Go to **Cost Management → Cost Explorer**
2. You'll see a graph of **daily costs for the last 6 months**

**Default view shows:**
- **X-axis:** Time (daily or monthly)
- **Y-axis:** Cost in USD
- **Breakdown:** By service (colored bars)

**Interpretation example:**
- Blue bars = EC2
- Orange bars = S3
- Green bars = Data Transfer

If you see a sudden spike, click on that day to drill down.

---

### Step 3: Create Your First Custom Report

#### Report 1: Monthly Spending by Service

**Goal:** Understand which AWS services cost the most.

**Steps:**
1. In Cost Explorer, click **Reports** (top right)
2. Time period: **Last 3 months**
3. Granularity: **Monthly**
4. Group by: **Service**
5. Click **Apply**

**Result:** Bar chart showing EC2, S3, RDS, etc. costs per month.

**Save the report:**
1. Click **Save report** (top right)
2. Name: `Monthly-by-Service`
3. Save

**Why save?** You can reload this report instantly instead of reconfiguring filters.

---

#### Report 2: This Month's Daily Breakdown

**Goal:** Track spending in real-time during the month.

**Steps:**
1. Time period: **This month**
2. Granularity: **Daily**
3. Group by: **Service**
4. Save as: `ThisMonth-Daily`

**Use this:** Check every morning to see if yesterday's costs were normal.

---

#### Report 3: Costs by Region

**Goal:** Identify if you're using expensive regions.

**Steps:**
1. Time period: **Last month**
2. Granularity: **Monthly**
3. Group by: **Region**
4. Save as: `LastMonth-by-Region`

**Why this matters:**
- `us-east-1` (N. Virginia): Cheapest region for most services
- `eu-central-1` (Frankfurt): 10-15% more expensive
- `ap-southeast-2` (Sydney): 20-30% more expensive

If you see significant spend in expensive regions and don't need the location, migrate resources.

---

#### Report 4: Costs by Resource Tag

**Goal:** Track spending per project or environment.

**Prerequisites:**
1. Tag your resources:
   - `Project=DataLake`
   - `Environment=Prod` or `Dev`
2. Activate tags in **Billing Console → Cost Allocation Tags**

**Steps:**
1. Time period: **This month**
2. Group by: **Tag → Project**
3. Filter: Only show tagged resources
4. Save as: `ThisMonth-by-Project`

**Result:** See how much each project costs (e.g., DataLake = $50, WebApp = $120).

**Best practice:** Tag everything! Untagged resources = "mystery spend" you can't attribute.

---

### Step 4: Create a Budget with Alerts

**Why budgets?** Prevent surprise bills by getting alerts before costs spiral.

#### Budget 1: Monthly Cost Budget

**Goal:** Alert if total monthly spend exceeds $50.

**Steps:**

1. Go to **Billing Console → Budgets**
2. Click **Create budget**

**Page 1: Choose budget type**
- Select **Customize (advanced)**
- Budget type: **Cost budget**
- Click **Next**

**Page 2: Set budget details**
- **Name:** `Monthly-Total-Budget`
- **Period:** Monthly
- **Budget renewal type:** Recurring budget
- **Start month:** Current month
- **Budgeting method:** Fixed
- **Budgeted amount:** `$50.00`
- Click **Next**

**Page 3: Configure alerts**
Click **Add an alert threshold** (repeat for each threshold):

**Alert 1: 50% threshold (actual spend)**
- **Threshold:** 50% of budgeted amount
- **Trigger:** Actual costs
- **Email recipients:** your-email@example.com
- Click **Add**

**Alert 2: 80% threshold (actual spend)**
- **Threshold:** 80%
- **Trigger:** Actual
- **Email:** your-email@example.com

**Alert 3: 100% threshold (forecasted)**
- **Threshold:** 100%
- **Trigger:** **Forecasted costs**
  - AWS predicts if you'll exceed budget by month-end
- **Email:** your-email@example.com

**Page 4: Review and create**
- Review all details
- Click **Create budget**

**What happens:**
- Day 10: You've spent $25 → Email: "50% of budget reached"
- Day 20: You've spent $40 → Email: "80% of budget reached"
- Day 22: AWS forecasts you'll hit $52 by month-end → Email: "Forecasted to exceed 100%"

**Important:** Budgets send emails but **don't stop resources**. You must manually take action.

---

#### Budget 2: EC2 Usage Budget

**Goal:** Alert if EC2 instance-hours exceed 100 hours/month.

**Why?** Free Tier includes 750 hours/month of t2.micro or t3.micro. This budget ensures you stay within it.

**Steps:**
1. Create budget
2. Type: **Usage budget**
3. Service: **Amazon Elastic Compute Cloud**
4. Usage type: Search for `BoxUsage:t2.micro` (or your instance type)
5. Amount: `750 hours` (Free Tier limit)
6. Alert thresholds: 80%, 100%
7. Create

**Result:** Email when you've used 600 hours (80% of Free Tier).

---

### Step 5: Enable Cost Anomaly Detection

**What is this?** Machine learning that detects unusual spending patterns.

**Example anomalies:**
- EC2 costs jumped from $10/day to $150/day (forgot to stop instance)
- S3 costs increased 500% (someone uploaded massive files)
- Data transfer spiked (possible misconfigured Lambda)

**Setup:**

1. Go to **Cost Management → Cost Anomaly Detection**
2. Click **Create monitor**

**Monitor settings:**
- **Monitor name:** `EC2-Anomaly-Monitor`
- **Monitor type:** AWS Service
- **Service:** Amazon Elastic Compute Cloud (EC2)
- Click **Next**

**Alert subscription:**
- **Subscription name:** `EC2-Cost-Alerts`
- **Threshold:** $5 (alert if anomaly exceeds $5)
- **Frequency:** Daily summary
- **Email:** your-email@example.com
- Click **Create monitor**

**What happens:**
- AWS analyzes your EC2 spending patterns
- If tomorrow's EC2 cost is 3× higher than normal, you get an email
- Email includes: What service, how much, when it started

**Best practice:** Create monitors for your top 3 services (EC2, RDS, S3).

---

### Step 6: (Optional) Export Cost & Usage Report to S3

**Why?** Cost Explorer shows aggregated data. CUR gives you **line-item details** for every API call.

**Use cases:**
- Build custom dashboards in Athena/QuickSight
- Integrate with data warehouse for cross-company analysis
- Compliance/audit requirements

**Setup:**

1. Go to **Billing Console → Cost & Usage Reports**
2. Click **Create report**

**Report settings:**
- **Report name:** `MyCURExport`
- **Time granularity:** Daily
- **Include:** Resource IDs
- **Data refresh settings:** Automatically refresh (if invoice changes)
- Click **Next**

**Delivery options:**
- **S3 bucket:** Create new bucket `my-cur-export-<your-name>`
- **Report path prefix:** `cur/`
- **Compression:** GZIP
- **Format:** Parquet
  - **Why Parquet?** 10× faster queries in Athena, smaller file sizes
- **Versioning:** Overwrite existing report
- Click **Next**

**Review and create**

**What happens:**
- Every day, AWS writes a Parquet file to `s3://my-cur-export-<name>/cur/`
- File contains every line item: EC2 instance-hours, S3 GET requests, data transfer bytes
- You can query with Athena:

```sql
SELECT line_item_resource_id, SUM(line_item_blended_cost)
FROM cur_data
WHERE line_item_product_code = 'AmazonEC2'
GROUP BY line_item_resource_id
ORDER BY SUM(line_item_blended_cost) DESC;
```

**Advanced:** Connect CUR to QuickSight for interactive dashboards.

---

## Understanding Your Bill

### AWS Bill Structure

**Key concepts:**

1. **Blended cost:** What you actually pay (after discounts, credits)
2. **Unblended cost:** List price before discounts
3. **Amortized cost:** Reserved Instances spread over time

**Example:**
- You bought a 1-year Reserved Instance for $100 upfront
- Unblended: $100 in January, $0 for remaining months
- Amortized: $8.33 per month for 12 months (more accurate)

**Use amortized costs** for budgeting and reports.

### Common Cost Drivers

| Service | What You Pay For | Optimization Tips |
|---------|------------------|-------------------|
| **EC2** | Instance-hours (even if stopped = EBS storage) | Use t3.micro, stop when idle, Reserved Instances |
| **S3** | Storage (GB-month) + requests (GET, PUT) | Lifecycle policies, delete unused data, compress files |
| **RDS** | Instance-hours + storage + backup storage | Use t3.micro, delete old snapshots, Aurora Serverless |
| **Data Transfer** | Outbound data (to internet) | Use CloudFront, VPC endpoints, avoid cross-region |
| **Athena** | Data scanned per query ($/TB) | Use Parquet, partitions, `LIMIT` clauses |
| **Glue** | DPU-hours (crawler + ETL jobs) | Reduce crawler frequency, optimize ETL code |
| **Lambda** | Invocations + duration (GB-seconds) | Reduce memory allocation, optimize code, cache results |

### Hidden Costs to Watch

1. **Stopped EC2 instances:** You still pay for attached EBS volumes
   - Solution: Create AMI, terminate instance, launch later from AMI
2. **Idle Load Balancers:** $18/month even with zero traffic
   - Solution: Delete unused ALB/NLB
3. **NAT Gateway:** $32/month + data processing fees
   - Solution: Use VPC endpoints for AWS services, avoid public subnet routing
4. **Old EBS snapshots:** $0.05/GB-month adds up
   - Solution: Delete snapshots older than 90 days (keep what you need)
5. **Elastic IPs not attached:** $0.005/hour ($3.60/month)
   - Solution: Release unused Elastic IPs

---

## Practical Exercises

### Exercise 1: Find Your Most Expensive Service

1. Open Cost Explorer
2. Last month, monthly granularity, group by Service
3. Identify the top 3 services
4. For each, ask: "Is this expected? Can I optimize?"

**Example findings:**
- EC2 = $80 → Check instance types, consider Reserved Instances
- S3 = $15 → Check storage class distribution, add lifecycle policies
- Data Transfer = $20 → Identify what's transferring, use CloudFront/VPC endpoints

---

### Exercise 2: Simulate a Budget Alert

**Goal:** Test that your budget emails work.

**Method:**
1. Create budget: $1/month
2. Spin up a t3.micro instance (costs ~$0.01/hour)
3. Run for 2-3 hours ($0.02-0.03)
4. Wait 24 hours for budget to update
5. You should receive "80% of budget" email

**Cleanup:** Stop the instance, delete the test budget.

---

### Exercise 3: Tag Resources and Filter Costs

**Goal:** See how tagging enables cost attribution.

**Steps:**

1. Tag an S3 bucket:
   - Bucket → Properties → Tags
   - Add: `Project=DataLake`, `Owner=YourName`
2. Tag an EC2 instance:
   - Instance → Tags tab → Add tags
   - Add: `Project=WebApp`, `Environment=Dev`
3. Go to **Billing Console → Cost Allocation Tags**
4. Activate `Project` and `Owner` tags (takes 24 hours)
5. Next day: Cost Explorer → Group by Tag → Project

**Result:** See spending split by project.

---

### Exercise 4: Set Up Cost Anomaly Detection

1. Create monitor for S3
2. Upload a large file to trigger anomaly (e.g., 10 GB)
3. Wait 24-48 hours
4. Check email for anomaly alert

**Expected alert:** "S3 costs increased by 500% (from $0.50 to $3.00)"

---

## Best Practices for Cost Management

### 1. Enable Budgets on Day 1
Don't wait until you get a surprise bill.
- Set conservative budgets initially ($10-20/month)
- Adjust as you understand your usage

### 2. Use Tags Religiously
**Mandatory tags:**
- `Project` (DataLake, WebApp, etc.)
- `Environment` (Dev, Staging, Prod)
- `Owner` (your email)
- `CostCenter` (if multiple teams)

**Enforcement:** Use AWS Organizations SCP to require tags.

### 3. Review Costs Weekly
Make it a habit:
- Monday morning: Check Cost Explorer for last week
- Investigate any spikes
- Verify budgets are on track

### 4. Automate Shutdowns
**Dev/test environments:**
- Use EventBridge to stop EC2 instances at 6 PM daily
- Start at 8 AM on weekdays
- **Savings:** 66% (12 hours/day instead of 24)

**Example Lambda function:**
```python
import boto3

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    # Stop all instances tagged Environment=Dev
    instances = ec2.describe_instances(
        Filters=[{'Name': 'tag:Environment', 'Values': ['Dev']}]
    )
    instance_ids = [i['InstanceId'] for r in instances['Reservations'] for i in r['Instances']]
    ec2.stop_instances(InstanceIds=instance_ids)
    return f"Stopped {len(instance_ids)} instances"
```

### 5. Use AWS Free Tier Wisely
**Always Free:**
- Lambda: 1M requests/month
- DynamoDB: 25 GB storage
- CloudWatch: 10 custom metrics

**12-Month Free Tier:**
- EC2: 750 hours/month of t2.micro or t3.micro
- S3: 5 GB storage
- RDS: 750 hours/month of db.t2.micro or db.t3.micro

**Stay within limits** to keep costs near $0 during learning.

### 6. Leverage Rightsizing Recommendations

1. Go to **Cost Explorer → Recommendations**
2. View **Rightsizing recommendations**
3. AWS analyzes your EC2 usage (CPU, memory, network)
4. Suggests downsizing: "Your t3.large averages 5% CPU → use t3.small"

**Example:**
- Current: t3.large ($60/month)
- Recommended: t3.small ($15/month)
- **Savings:** $45/month

**Warning:** Verify performance won't degrade before downsizing.

### 7. Consider Savings Plans / Reserved Instances

**If you have steady workloads:**
- **Compute Savings Plans:** 1-year commitment → 20-30% discount
- **EC2 Reserved Instances:** 3-year commitment → 40-60% discount

**Use Cost Explorer RI/SP recommendations** to see potential savings.

**Example:**
- Current on-demand EC2: $100/month
- 1-year Savings Plan: $70/month
- 3-year Reserved: $50/month

**Caution:** Only commit if you're sure you'll use the resources.

---

## Troubleshooting

### Issue 1: Cost Explorer shows no data
**Cause:** Cost Explorer not enabled or account is too new.
**Fix:** Enable Cost Explorer, wait 24 hours.

### Issue 2: Budget alerts aren't arriving
**Causes:**
- Email not confirmed (check spam folder for confirmation email)
- Budget is set too high (you haven't reached threshold)
- Billing data updates daily (alerts delayed by 24 hours)

**Fix:**
- Confirm email subscription
- Lower budget to $1 for testing
- Wait 24 hours after spending

### Issue 3: Tags don't appear in Cost Explorer
**Cause:** Tags not activated in Cost Allocation Tags.
**Fix:**
1. Billing Console → Cost Allocation Tags
2. Select your custom tags
3. Click **Activate**
4. Wait 24 hours for data to populate

### Issue 4: CUR export file is empty
**Cause:** No spending occurred yet, or bucket permissions incorrect.
**Fix:**
- Verify S3 bucket policy allows `billingreports.amazonaws.com` to write
- Wait until end of day (CUR generates once daily)
- Check CloudTrail for permission errors

---

## Real-World Cost Optimization Story

**Startup: Image Processing SaaS**

**Initial architecture:**
- 5× t3.large EC2 instances (24/7): $300/month
- RDS db.m5.large (24/7): $150/month
- 10 TB S3 Standard storage: $230/month
- NAT Gateway: $50/month
- **Total:** $730/month

**After cost analysis with Cost Explorer:**

**Findings:**
1. EC2 instances averaging 15% CPU utilization
2. RDS instance averaging 20% CPU
3. 8 TB of S3 data not accessed in 90 days
4. NAT Gateway processing mostly AWS service traffic

**Optimizations:**
1. **EC2:** Downsize to 3× t3.medium, use Auto Scaling → $90/month (70% savings)
2. **RDS:** Switch to Aurora Serverless (scales to zero) → $50/month (67% savings)
3. **S3:** Lifecycle policy (move 8 TB to Glacier) → $62/month (73% savings)
4. **NAT:** Replace with VPC endpoints for S3/DynamoDB → $5/month (90% savings)
5. **Bonus:** 1-year Savings Plan on EC2 → additional 20% off

**New total:** $167/month (77% reduction)
**Annual savings:** $6,756

**Time invested:** 4 hours of analysis + 2 hours of implementation
**ROI:** $1,126/hour

---

## Summary

**What you learned:**
- ✅ Enable and navigate AWS Cost Explorer
- ✅ Create custom reports to analyze spending
- ✅ Set up budgets with email alerts
- ✅ Use Cost Anomaly Detection to catch spikes
- ✅ Export Cost & Usage Reports for deep analysis
- ✅ Apply tagging for cost attribution
- ✅ Optimize costs using rightsizing recommendations

**Key concepts:**
- **Cost Explorer:** Visualize and filter spending
- **Budgets:** Proactive alerts to prevent overspend
- **Tags:** Enable cost attribution by project/team
- **CUR:** Raw data for advanced analysis
- **Anomaly Detection:** ML-based spike alerts

**This is how you avoid surprise AWS bills and optimize for cost.**

---

**Next:** [Exercises & Practice →](./exercises/exercise.md)
