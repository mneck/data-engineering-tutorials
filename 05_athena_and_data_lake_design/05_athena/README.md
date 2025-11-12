# Enhanced Athena Learning Materials

## Overview

This directory contains comprehensive, self-study materials for learning AWS Athena, created by enhancing the original course lecture notes.

**What's enhanced:**
- TL;DR sections for quick understanding
- "Big picture" explanations of WHY, not just HOW
- Real-world use cases and context
- Complete step-by-step instructions with no missing steps
- Troubleshooting guides
- Performance and cost optimization tips
- Practice exercises with detailed solutions

---

## Structure

```
claude_notes/05_athena/
├── README.md                              ← You are here
├── 01_athena_basics_enhanced.md           ← Start here: What is Athena, querying S3
├── 02_Partitioning_&_Schema_enhanced.md   ← Optimization: Partitions, CTAS, Parquet
├── exercises/
│   └── exercise_enhanced.md               ← 20 hands-on exercises (3 sections)
├── solutions/
│   └── solution_enhanced.md               ← Complete solutions with explanations
└── sample_data/                           ← (empty - samples in original assets)
```

**Original course materials:**
- Available in parent directory: `../../05_athena/`
- Assets (CSVs, images) shared between both versions

---

## Learning Path

### Phase 1: Foundations (2-3 hours)
1. Read [01_athena_basics_enhanced.md](./01_athena_basics_enhanced.md)
2. Set up AWS Athena (follow Prerequisites section)
3. Complete the "Hands-On" section with trips.csv data
4. Experiment with the sample queries

**You'll learn:**
- What Athena is and when to use it
- How to create databases and tables
- Basic SQL queries on S3 data
- CSV vs Parquet performance

---

### Phase 2: Optimization (2-3 hours)
1. Read [02_Partitioning_&_Schema_enhanced.md](./02_Partitioning_&_Schema_enhanced.md)
2. Create partitioned tables
3. Run CTAS queries to convert formats
4. Compare costs with and without optimization

**You'll learn:**
- Why partitioning cuts costs by 10-100x
- How to organize data in S3 folders
- CTAS for data transformation
- Schema-on-Read vs Schema-on-Write

---

### Phase 3: Hands-On Practice (4-6 hours)
1. Open [exercises/exercise_enhanced.md](./exercises/exercise_enhanced.md)
2. Work through exercises sequentially:
   - **Section A (1-7):** SQL fundamentals
   - **Section B (8-13):** Partitioning and optimization
   - **Section C (14-20):** End-to-end pipeline (EC2 → S3 → Athena)
3. Check solutions only after attempting each exercise
4. Understand the "why" behind each solution

**You'll build:**
- SQL query skills
- Partition management skills
- Data pipeline from scratch
- Cost optimization expertise

---

## Prerequisites

### AWS Setup
✓ AWS account (Free Tier eligible)
✓ IAM user with policies:
  - `AmazonAthenaFullAccess`
  - `AmazonS3FullAccess` (or specific bucket access)
✓ S3 bucket created
✓ Athena query results location configured

**Estimated cost:** $0-5 for entire course (Free Tier covers most)

### Knowledge Prerequisites
✓ Basic SQL (SELECT, WHERE, GROUP BY)
✓ Familiarity with AWS Console
✓ Basic command line usage
✓ CSV file format understanding

**Nice to have:**
- Python basics (for Exercise Section C)
- Understanding of data warehousing concepts

---

## Datasets

All sample datasets are located in the original assets folder: `../../05_athena/assets/`

**Key datasets:**

1. **trips.csv** (2 KB)
   - NYC taxi trip data
   - For: Basic Athena setup and queries
   - Columns: trip_id, trip_date, vendor_id, passenger_count, trip_distance, fare_amount, payment_type

2. **sales_data.csv** (12 KB)
   - Sample sales transactions (~1000 rows)
   - For: SQL exercises (Section A)
   - Columns: id, customer, amount, date

3. **s3_partitioned_sales.zip** (7 KB)
   - Pre-partitioned sales data by year/month
   - For: Partitioning exercises (Section B)
   - Structure: year=YYYY/month=MM/ folders

4. **ec2_generated.csv** (5 KB)
   - Sample EC2-generated data
   - For: Pipeline exercises (Section C)
   - You'll generate your own version in exercises

5. **customers_lookup.csv** (500 bytes)
   - Customer metadata for join exercises
   - Columns: customer, email, region

---

## Key Concepts Covered

### Athena Fundamentals
- Serverless query engine for S3
- Schema-on-Read vs Schema-on-Write
- Supported file formats (CSV, JSON, Parquet, ORC)
- SerDe (Serializer/Deserializer)
- External tables (no data movement)

### SQL in Athena
- Basic queries (SELECT, WHERE, ORDER BY, LIMIT)
- Aggregations (COUNT, SUM, AVG, MIN, MAX)
- Grouping (GROUP BY, HAVING)
- Joins (LEFT, INNER, FULL OUTER)
- Pattern matching (LIKE, wildcards)
- Date filtering (BETWEEN, date functions)

### Performance Optimization
- **Partitioning:** Organizing data in S3 folders
- **Partition pruning:** Scanning only relevant folders
- **Columnar formats:** Parquet and ORC
- **Compression:** SNAPPY, GZIP, ZSTD
- **CTAS:** Create Table As Select for transformations
- **Cost calculation:** $5 per TB scanned

### Data Pipeline Patterns
- EC2 data generation
- Upload to S3 (data lake ingestion)
- Athena for analytics (query layer)
- Partition management (add/drop)
- Data validation and quality checks

---

## Tips for Success

### 1. Learn by Doing
Don't just read—actually run the queries in your AWS account. Muscle memory matters.

### 2. Check "Data Scanned"
After each query, look at the query details. This shows how much data was scanned and helps you understand cost implications.

### 3. Compare Before/After
When learning optimizations (Parquet, partitioning), run the same query on both versions and compare:
- Data scanned
- Query time
- Calculated cost

### 4. Break Things
Try to break things intentionally to understand error messages:
- Create table with wrong S3 path
- Query without partition filters
- Use wrong SerDe
- Skip header line vs not skipping

Understanding errors makes you a better troubleshooter.

### 5. Think About Scale
Small datasets (KBs) don't show dramatic improvements. Mentally scale up:
- If 10 KB → 100 MB saved = 10,000x bigger = 1 TB saved
- If query takes 0.5s → at 10,000x = 1.5 hours without optimization

### 6. Real-World Context
For each concept, ask:
- When would I use this in production?
- What problems does this solve?
- What are the trade-offs?

---

## Common Pitfalls

### 1. Not Configuring Query Results Location
**Error:** "Query exhausted resources at this scale factor"
**Fix:** Settings → Set query result location in S3

### 2. Forgetting to Skip Header Row
**Error:** CSV first row appears as data
**Fix:** Add `TBLPROPERTIES ('skip.header.line.count'='1')`

### 3. Wrong Partition Folder Names
**Error:** MSCK REPAIR finds no partitions
**Fix:** Must use `column_name=value` format: `year=2025/month=01/`

### 4. Not Using Partition Filters
**Issue:** High costs despite having partitions
**Fix:** Always filter on partition columns: `WHERE year='2025' AND month='01'`

### 5. Using SELECT * on Large Tables
**Issue:** Scans all columns even if you only need a few
**Fix:** `SELECT col1, col2` instead of `SELECT *` (especially with Parquet)

---

## Cost Management

### Estimated Costs for This Course

| Activity | Data Scanned | Cost |
|----------|-------------|------|
| Basic exercises (A) | ~50 MB | $0.00025 |
| Partitioning exercises (B) | ~100 MB | $0.0005 |
| Pipeline exercises (C) | ~200 MB | $0.001 |
| **Total** | **~350 MB** | **~$0.002** |

Plus:
- S3 storage: ~$0.001/month (negligible)
- EC2 (t2.micro): Free Tier or ~$0.10/day

**Total course cost: < $1** (within Free Tier limits)

### Cost Optimization Checklist

When you move to production:

✓ **Always partition** by date/region/category
✓ **Use Parquet/ORC** instead of CSV/JSON
✓ **Enable compression** (SNAPPY for balance)
✓ **Filter on partition columns** in every query
✓ **SELECT specific columns**, not `SELECT *`
✓ **Avoid small files** (<128 MB) - compact them
✓ **Set up S3 lifecycle policies** for old data
✓ **Use workgroups** to track team costs
✓ **Monitor with CloudWatch** and set billing alarms

---

## Next Steps After Completing

### Intermediate
1. **AWS Glue:** Automate ETL jobs and schema crawling
2. **Lake Formation:** Data lake governance and access control
3. **QuickSight:** Connect Athena to BI dashboards
4. **CloudWatch:** Set up monitoring and alerts
5. **Athena Workgroups:** Manage team permissions and budgets

### Advanced
1. **Federated Queries:** Query across S3, RDS, DynamoDB
2. **User-Defined Functions (UDFs):** Custom SQL functions with Lambda
3. **ML Functions:** Use built-in ML models in SQL
4. **ACID Transactions:** Use Apache Iceberg/Hudi tables
5. **Cross-Account Access:** Share data across AWS accounts

### Real Projects
1. Build a log analytics dashboard (CloudTrail → S3 → Athena → QuickSight)
2. Create a data lake for business intelligence
3. Implement a cost monitoring system (CUR data → Athena)
4. Build a streaming data pipeline (Kinesis → S3 → Athena)

---

## Resources

### AWS Documentation
- [Athena User Guide](https://docs.aws.amazon.com/athena/latest/ug/)
- [SQL Reference](https://docs.aws.amazon.com/athena/latest/ug/ddl-sql-reference.html)
- [Best Practices](https://docs.aws.amazon.com/athena/latest/ug/best-practices.html)

### Practice Datasets
- [AWS Open Data Registry](https://registry.opendata.aws/)
- [NYC Taxi Data](https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page)
- [Amazon Customer Reviews](https://s3.amazonaws.com/amazon-reviews-pds/readme.html)
- [COVID-19 Data Lake](https://aws.amazon.com/blogs/big-data/a-public-data-lake-for-analysis-of-covid-19-data/)

### Blog Posts
- [Top 10 Performance Tuning Tips](https://aws.amazon.com/blogs/big-data/top-10-performance-tuning-tips-for-amazon-athena/)
- [Athena Cost Optimization](https://aws.amazon.com/blogs/big-data/10-things-you-should-know-about-amazon-athena/)

### Community
- [AWS re:Post (Athena tag)](https://repost.aws/tags/TA4IHN2rz9QiySuIr7rF7Tuw/amazon-athena)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/amazon-athena)

---

## Feedback and Improvements

These materials were created to address common gaps in lecture notes:
- Missing "why" explanations
- Incomplete instructions
- Lack of real-world context
- No troubleshooting guidance

If you find areas that could be improved, consider:
- Adding your own notes in margins
- Creating additional practice exercises
- Building real projects to reinforce learning
- Teaching concepts to someone else (best way to solidify understanding)

---

## Summary

**What you'll achieve:**
- Deep understanding of Athena and when to use it
- Ability to query data lakes with SQL
- Cost optimization skills (10-100x savings)
- Hands-on experience building data pipelines
- Confidence to implement Athena in production

**Time investment:** 8-12 hours
**Cost:** < $1 (Free Tier)
**Value:** Skills used by data engineers at top companies processing petabytes daily

**Ready to start?** Open [01_athena_basics_enhanced.md](./01_athena_basics_enhanced.md) and dive in!

Good luck! 🚀
