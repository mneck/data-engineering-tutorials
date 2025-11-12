# Enhanced AWS Data Engineering Course Notes

Self-study enhanced version of your AWS Data Engineering course materials.

## What's Different?

**Original lectures:** Basic analogies, incomplete steps, missing context
**Enhanced notes:** Real-world use cases, complete instructions, cost breakdowns, troubleshooting, best practices

## Module 4: Data Lake Design

**Location:** [claude_notes/04_data_lake_design/](./04_data_lake_design/)

### Core Lessons

1. **[Data Lake Architecture](./04_data_lake_design/01_data_design.md)** (518 lines)
   - Build serverless data lake with S3 + Glue + Athena
   - TL;DR: Store data in S3, catalog with Glue, query with SQL
   - Real use case: E-commerce analytics (90% cost savings vs RDS)
   - Complete tutorial: <$0.01 total cost

2. **[S3 Lifecycle Policies](./04_data_lake_design/02_data_retention_policy.md)** (566 lines)
   - Automate data retention and archival
   - TL;DR: Move old data to cheap storage automatically
   - Real use case: Application logs (70-90% storage cost reduction)
   - Best practices for production environments

3. **[Cost Explorer & Budgets](./04_data_lake_design/03_cost_explorer.md)** (696 lines)
   - Monitor spending, set alerts, prevent surprise bills
   - TL;DR: Track costs, set budgets, get email alerts
   - Real use case: Startup reduces AWS bill from $730 to $167/month
   - Complete guide to cost optimization

### Practice Materials

4. **[20 Hands-On Exercises](./04_data_lake_design/exercises.md)** (620 lines)
   - Organized by difficulty: Beginner → Advanced
   - Each includes: context, learning goals, hints, verification steps
   - Topics: Athena queries, lifecycle policies, cost management, EC2 optimization
   - Bonus end-to-end project

5. **[Sample Data Files](./04_data_lake_design/sample_data/)** (5 datasets)
   - `sales_data.csv` - E-commerce transactions (20 rows)
   - `customers.csv` - Customer master data (5 customers)
   - `products.csv` - Product catalog (5 products)
   - `clickstream_events.json` - Web analytics (10 events)
   - `server_logs.txt` - Application logs (15 log entries)
   - Complete documentation in [sample_data/README.md](./04_data_lake_design/sample_data/README.md)

## Quick Start

### Option 1: Read First, Practice Later
1. Read [01_data_design.md](./04_data_lake_design/01_data_design.md)
2. Read [02_data_retention_policy.md](./04_data_lake_design/02_data_retention_policy.md)
3. Read [03_cost_explorer.md](./04_data_lake_design/03_cost_explorer.md)
4. Work through [exercises.md](./04_data_lake_design/exercises.md)

### Option 2: Learn by Doing
1. Upload [sample_data/sales_data.csv](./04_data_lake_design/sample_data/sales_data.csv) to S3
2. Follow step-by-step tutorial in [01_data_design.md](./04_data_lake_design/01_data_design.md)
3. Set up lifecycle policy from [02_data_retention_policy.md](./04_data_lake_design/02_data_retention_policy.md)
4. Configure budgets from [03_cost_explorer.md](./04_data_lake_design/03_cost_explorer.md)
5. Complete exercises

## Key Learning Outcomes

After completing this module:
- ✅ Build a serverless data lake (S3 + Glue + Athena)
- ✅ Write SQL queries on S3 files
- ✅ Automate data retention with lifecycle policies
- ✅ Monitor and optimize AWS costs
- ✅ Understand storage class economics
- ✅ Set up production-ready data pipelines

## Cost Expectations

**For all tutorials combined:** ~$0.05/month
- S3 storage: $0.01
- Glue crawlers: $0.03
- Athena queries: $0.0001

**Staying in Free Tier:**
- Use t2.micro/t3.micro (not larger instances)
- Delete resources after practice
- Set budget alert at $5

## Prerequisites

- AWS account with admin IAM user
- Basic SQL knowledge
- Understanding of S3, EC2 from previous weeks

## File Statistics

- **Total content:** 2,665 lines of enhanced documentation
- **Sample data:** 5 practice datasets with complete schemas
- **Exercises:** 20 hands-on exercises with solutions
- **Real-world examples:** 10+ industry use cases
- **Cost breakdowns:** Detailed pricing for every service

## Next Steps

1. Complete this module first
2. Build a personal project using these skills
3. Move to Week 7: Advanced Athena (when ready)
4. Build portfolio project for GitHub

## Comparison: Original vs Enhanced

| Aspect | Original | Enhanced |
|--------|----------|----------|
| Length | ~400 lines | 2,665 lines |
| Real-world context | Minimal | 10+ industry examples |
| Cost info | Basic | Complete breakdowns + optimization |
| Troubleshooting | None | Dedicated sections |
| Sample data | Links only | 5 complete datasets |
| Exercises | Task list | 20 exercises with context |
| Best practices | Few | Production-ready patterns |

## Feedback

These notes were created by Claude Code to enhance your self-study experience.

**If you want to enhance other modules**, just ask!

---

**Location:** `/Users/sasan/spicy_projects/de-week-6-AWS-Foundations-and-Data-Lake/claude_notes/`

**Last updated:** 2025-11-10
