# Enhanced Data Lake Design Notes

These are enhanced, self-study versions of the Week 6 lecture notes. Each file includes:

- **TL;DR** - Quick summary of key concepts
- **Big Picture** - What it is, why it matters, when to use it
- **Real-world use cases** - Practical examples from industry
- **Complete step-by-step instructions** - No missing steps
- **Cost breakdowns** - Understand pricing implications
- **Troubleshooting guides** - Common issues and fixes
- **Best practices** - Production-ready patterns

---

## Files in This Directory

### Core Concepts

1. **[01_data_design.md](./01_data_design.md)** - Data Lake Architecture (S3 + Glue + Athena)
   - Build your first serverless data lake
   - Understand schema-on-read vs schema-on-write
   - Query S3 files with SQL using Athena
   - Real-world e-commerce analytics example
   - Complete cost breakdown (<$0.01 for tutorial)

2. **[02_data_retention_policy.md](./02_data_retention_policy.md)** - S3 Lifecycle Policies
   - Automate data retention and archival
   - Save 50-90% on storage costs
   - Transition through storage tiers (Standard → IA → Glacier)
   - Clean up old versions and incomplete uploads
   - Real-world logging example (83% cost reduction)

3. **[03_cost_explorer.md](./03_cost_explorer.md)** - AWS Billing & Cost Management
   - Visualize where your money goes
   - Set up budgets with email alerts
   - Detect cost anomalies with ML
   - Export detailed cost reports
   - Real-world cost optimization story (77% reduction)

### Practice Materials

4. **[exercises.md](./exercises.md)** - 20 Hands-On Exercises
   - Organized by difficulty and topic
   - Complete context and learning goals for each exercise
   - Hints and verification steps
   - Bonus end-to-end project

5. **[sample_data/](./sample_data/)** - Practice Datasets
   - `sales_data.csv` - E-commerce transactions
   - `customers.csv` - Customer master data
   - `products.csv` - Product catalog
   - `clickstream_events.json` - Web analytics events
   - `server_logs.txt` - Application logs
   - `README.md` - Data documentation and usage guide

---

## How to Use These Notes

### Step 1: Read in Order
Start with the core concept files in sequence:
1. Data Lake Design (understand the architecture)
2. Retention Policies (optimize costs)
3. Cost Explorer (monitor spending)

### Step 2: Hands-On Practice
- Follow the step-by-step tutorials in each file
- Use the sample data files provided
- Don't just read—actually build the data lake in your AWS account

### Step 3: Complete Exercises
- Work through [exercises.md](./exercises.md)
- Start with Section A (Data Lake Extensions)
- Try solving exercises before looking at solutions
- Build the end-to-end project

### Step 4: Build Something Real
Apply concepts to a personal project:
- Analyze your own data (CSV exports, logs, etc.)
- Build a portfolio project for GitHub
- Create a QuickSight dashboard to showcase

---

## Key Differences from Original Lectures

### What's Enhanced?

**Original notes:**
- Library/warehouse analogies (good for initial understanding)
- Basic step-by-step instructions
- Some missing prerequisites and context

**Enhanced notes:**
- TL;DR sections (get the point immediately)
- Real-world industry context (e-commerce, SaaS, startups)
- Complete cost breakdowns (know what you'll pay)
- Troubleshooting sections (fix common issues)
- Best practices for production (not just tutorials)
- Detailed explanations of "why" not just "how"

### Example Comparison

**Original:** "S3 is like a warehouse. Glue is like a librarian."

**Enhanced:** "S3 is object storage at $0.023/GB/month. It's the foundation of AWS data architecture because it separates storage from compute—you can run Athena, EMR, Redshift Spectrum, or SageMaker against the same S3 data without moving it. This eliminates ETL pipelines and reduces costs by 10-100× compared to traditional data warehouses. In a real e-commerce company with 1 TB of clickstream data, this architecture saves $230/month compared to RDS + $500/month compared to Redshift for the same analytical workload."

---

## Cost Expectations

For all tutorials combined:
- **S3 storage:** ~$0.01/month (sample data is <1 MB)
- **Glue crawlers:** ~$0.03 (3 crawlers × $0.01)
- **Athena queries:** ~$0.0001 (tiny datasets)
- **Total:** ~$0.05/month

**Staying in Free Tier:**
- S3: 5 GB free
- Athena: 10 GB scanned free
- Glue: No free tier but crawlers are cheap (<$0.01 each)

**Cost optimization tips:**
- Delete resources after learning (S3 buckets, EC2 instances)
- Set up budget alerts at $5 to catch mistakes
- Use Free Tier eligible services (t2.micro, not t3.large)

---

## Prerequisites

### AWS Account Setup
- ✅ AWS account activated
- ✅ Admin IAM user created (not using root)
- ✅ AWS CLI installed (optional, but recommended)

### AWS Knowledge
From previous weeks, you should understand:
- S3 buckets, objects, storage classes
- IAM users, policies, permissions
- EC2 instances (basic familiarity)
- AWS Management Console navigation

### Technical Skills
- Basic SQL (SELECT, WHERE, GROUP BY, JOIN)
- CSV/JSON file formats
- Command line basics (for AWS CLI exercises)

---

## Learning Outcomes

After completing these notes and exercises, you'll be able to:

**Technical Skills:**
- ✅ Design and build a serverless data lake on AWS
- ✅ Use Glue crawlers for automated schema discovery
- ✅ Write SQL queries on S3 data with Athena
- ✅ Set up S3 lifecycle policies for cost optimization
- ✅ Monitor AWS costs with Cost Explorer and budgets
- ✅ Join multiple datasets and create analytics queries
- ✅ Build basic BI dashboards with QuickSight

**Conceptual Understanding:**
- ✅ Schema-on-read vs schema-on-write architectures
- ✅ Separation of storage and compute
- ✅ Data lake vs data warehouse trade-offs
- ✅ Storage class economics (Standard vs IA vs Glacier)
- ✅ Partitioning for query performance and cost reduction

**Production Skills:**
- ✅ Cost estimation and optimization
- ✅ Setting up monitoring and alerts
- ✅ Troubleshooting common issues
- ✅ Following AWS best practices
- ✅ Building cost-effective data pipelines

---

## Additional Resources

### AWS Documentation
- [Amazon S3 User Guide](https://docs.aws.amazon.com/s3/)
- [AWS Glue Developer Guide](https://docs.aws.amazon.com/glue/)
- [Amazon Athena User Guide](https://docs.aws.amazon.com/athena/)

### AWS Free Tier
- [Free Tier Details](https://aws.amazon.com/free/)
- [Free Tier FAQs](https://aws.amazon.com/free/free-tier-faqs/)

### Cost Management
- [AWS Pricing Calculator](https://calculator.aws/)
- [AWS Cost Management](https://aws.amazon.com/aws-cost-management/)

### Community
- [AWS re:Post](https://repost.aws/) - AWS Q&A forum
- [r/aws](https://reddit.com/r/aws) - Reddit community
- [AWS Workshops](https://workshops.aws/) - Hands-on tutorials

---

## Troubleshooting

### Issue: "Access Denied" errors in Glue/Athena
**Cause:** IAM permissions missing
**Fix:** Ensure your IAM user has:
- `AmazonS3FullAccess`
- `AWSGlueConsoleFullAccess`
- `AmazonAthenaFullAccess`

### Issue: Athena shows "Table not found"
**Cause:** Wrong database selected
**Fix:** In Athena query editor, select correct database from dropdown

### Issue: Glue crawler finds 0 tables
**Cause:** S3 path is wrong or IAM role lacks permissions
**Fix:**
- Verify S3 path in crawler configuration
- Check Glue IAM role has `s3:GetObject` permission

### Issue: Budget alerts not arriving
**Cause:** Email not confirmed
**Fix:** Check spam folder for AWS Budgets confirmation email

### More troubleshooting guides in each individual file.

---

## Feedback & Questions

These notes were enhanced by Claude Code based on your original AWS course materials.

**If you find issues:**
- Missing explanations
- Incorrect information
- Confusing sections
- Outdated pricing

Feel free to edit and improve the notes yourself—they're Markdown files!

---

## Next Steps After This Module

1. **Week 7: Advanced Athena**
   - Partitioning strategies
   - Parquet/ORC formats
   - Query optimization
   - Federated queries

2. **Week 8: AWS Glue ETL**
   - Glue jobs and scripts
   - Data transformations
   - Apache Spark on Glue
   - Scheduling and orchestration

3. **Real-World Projects**
   - Build a data pipeline for your own data
   - Integrate with Redshift or Snowflake
   - Set up CI/CD for data engineering
   - Create a portfolio project for job applications

---

**Good luck with your data engineering journey!**
