# S3 Lifecycle Policies for Data Retention

## TL;DR
Automate data retention by creating S3 lifecycle rules that move objects to cheaper storage classes over time or delete them after a specified period. This cuts storage costs by 70-90% without manual intervention.

**Key takeaway:** Set it once, forget it. AWS automatically manages your data lifecycle based on age, saving you money and ensuring compliance.

---

## Big Picture: What, Why, When

### What are Lifecycle Policies?
Rules that automatically transition or delete S3 objects based on:
- **Age** (30 days old → move to cheaper storage)
- **Prefix/folder** (apply only to `logs/` folder)
- **Object tags** (apply only to objects tagged `Archive=true`)
- **Versioning state** (delete old versions after 90 days)

### Why Use Lifecycle Policies?

**The problem:** Not all data has equal value over time.
- Day 1: Sales data from today → needs instant access
- Day 90: Sales data from 3 months ago → accessed rarely
- Day 365: Year-old data → kept for compliance, never accessed

But if you store everything in **S3 Standard**, you pay premium pricing for data you never touch.

**The solution:** Automatically move data through storage tiers as it ages.

| Storage Class | Cost/GB/month | Use Case | Retrieval Time |
|---------------|---------------|----------|----------------|
| Standard | $0.023 | Frequently accessed | Instant |
| Standard-IA | $0.0125 | Monthly access | Instant |
| Glacier Flexible | $0.004 | Quarterly access | Minutes-hours |
| Glacier Deep Archive | $0.00099 | Yearly/compliance | 12-48 hours |

**Example savings:**
- 1 TB in Standard: $23/month
- 1 TB in Glacier: $4/month
- **Savings:** $228/year (83% reduction)

### When to Use This?

**Perfect for:**
- **Log files:** Keep recent logs in Standard, archive old logs
- **Backup data:** Immediate backups in IA, older backups in Glacier
- **Compliance data:** Legal/audit data that must be retained but rarely accessed
- **Data lake archives:** Historical data for occasional analytics
- **Media files:** Old videos/images that are no longer popular

**Not ideal for:**
- Data that's accessed unpredictably (retrieval fees can be high)
- Very small files (<128 KB)—minimum billable size makes it inefficient
- Frequently updated data (lifecycle transitions happen once, not continuously)

---

## Real-World Use Case: Application Logging

**Scenario:** You run a web application that generates logs.

**Requirements:**
- Last 7 days: Keep logs instantly available for debugging
- 7-30 days: Keep for compliance, occasional access
- 30-90 days: Archive, rare access (security audits)
- 90+ days: Delete (GDPR/retention policy)

**Without lifecycle policies:**
- Manual weekly cleanup scripts
- Risk of deleting wrong data
- Pay $23/month per TB for all logs

**With lifecycle policies:**
```
Day 0-7:   S3 Standard ($0.023/GB)
Day 7-30:  S3 Standard-IA ($0.0125/GB) - 46% cheaper
Day 30-90: Glacier Flexible ($0.004/GB) - 83% cheaper
Day 90+:   Deleted - 100% savings
```

**Result:**
- 100 GB logs/month generated
- Month 1: 100 GB × $0.023 = $2.30
- Month 2: 30 GB Standard + 70 GB IA = $1.59
- Month 3: 10 GB Standard + 20 GB IA + 70 GB Glacier = $0.76
- **Average cost:** $1.50/month vs $2.30 (35% savings)

For 10 TB/month, this saves **$9,600/year**.

---

## Prerequisites

- AWS account (Free Tier eligible)
- IAM user with `AmazonS3FullAccess` or:
  - `s3:PutLifecycleConfiguration`
  - `s3:GetLifecycleConfiguration`
- Understanding of S3 storage classes (covered in Week 3)

---

## Step-by-Step Implementation

### Step 1: Create a Demo Bucket

1. Open [S3 Console](https://s3.console.aws.amazon.com/s3/)
2. Click **Create bucket**
3. Settings:
   - **Name:** `my-lifecycle-demo-<your-name>-<random>`
   - **Region:** `us-east-1` (or your preferred region)
   - **Block Public Access:** Enabled (keep data private)
   - **Versioning:** Disabled (for now—we'll cover versioned objects later)
4. Click **Create bucket**

---

### Step 2: Upload Test Files

Upload files with different "ages" to simulate real-world scenarios.

1. Open the bucket
2. Click **Upload → Add files**
3. Upload 3-4 test files:
   - `recent_log.txt` (pretend this is today's log)
   - `old_backup.csv` (pretend this is 60 days old)
   - `archive_data.json` (pretend this is 200 days old)

**Note:** In reality, S3 tracks object creation time automatically. For testing, we'll pretend these files have different ages.

---

### Step 3: Create a Lifecycle Rule

#### Navigate to Lifecycle Configuration

1. Click on your bucket
2. Go to **Management** tab
3. Under **Lifecycle rules**, click **Create lifecycle rule**

#### Step 3.1: Define Rule Name and Scope

**Rule name:** `AutoArchiveAndDelete`

**Choose a rule scope:**
- Option 1: **Apply to all objects in the bucket**
  - Use this if you want the same policy for everything
- Option 2: **Limit the scope of this rule using filters**
  - Use this to apply rules only to specific folders or tagged objects

**For this demo:** Select **Apply to all objects in the bucket**

**Acknowledge the warning:** Check the box to confirm.

---

#### Step 3.2: Configure Lifecycle Rule Actions

You'll see checkboxes for different actions. Select:

✅ **Transition current versions of objects between storage classes**
- Move objects to cheaper storage as they age

✅ **Expire current versions of objects**
- Delete objects after retention period

**Leave unchecked for now:**
- Transition noncurrent versions (only needed if versioning is enabled)
- Permanently delete noncurrent versions
- Delete expired object delete markers

---

#### Step 3.3: Define Transition Rules

This is where the magic happens. You define the aging timeline.

**Transition 1: Move to Standard-IA**
- **Storage class transition:** Standard-IA
- **Days after object creation:** `30`

**Explanation:** After 30 days, move objects from Standard ($0.023/GB) to Standard-IA ($0.0125/GB).

Click **Add transition**

**Transition 2: Move to Glacier Flexible Retrieval**
- **Storage class transition:** Glacier Flexible Retrieval
- **Days after object creation:** `90`

**Explanation:** After 90 days, move objects to Glacier ($0.004/GB). Retrieval takes minutes to hours.

**Why not Glacier Deep Archive?**
- Deep Archive is cheaper ($0.00099/GB) but requires 12-48 hours for retrieval
- Use it only for compliance data you'll never need quickly

---

#### Step 3.4: Define Expiration Rule

**Expire current versions of objects**
- **Days after object creation:** `365`

**Explanation:** After 1 year, permanently delete the object.

**Important:** There's no "recycle bin" in S3. Deletion is permanent (unless versioning is enabled).

---

#### Step 3.5: Review the Timeline

AWS shows a summary:

```
Day 0:    Object created → S3 Standard
Day 30:   Transition → S3 Standard-IA
Day 90:   Transition → Glacier Flexible Retrieval
Day 365:  Expiration → Object deleted
```

**Verify this matches your retention policy.**

Click **Create rule**

---

### Step 4: Verify the Rule

1. Go to **Management** tab → **Lifecycle rules**
2. You should see `AutoArchiveAndDelete` with status **Enabled**
3. Click on the rule to view details

**How long before it takes effect?**
- Lifecycle rules run daily (usually at midnight UTC)
- Transitions/deletions happen within 24-48 hours of reaching the threshold

**Testing tip:** You can't speed this up in the console, but you can verify the rule is correctly configured.

---

## Advanced Scenarios

### Scenario 1: Apply Rule to Specific Folder (Prefix)

**Use case:** Only archive logs, not user uploads.

**Steps:**
1. Create rule: `LogsArchiveRule`
2. **Scope:** Limit the scope
3. **Prefix:** `logs/`
4. Configure transitions: 7 days → IA, 30 days → Glacier, 90 days → Delete

**Result:** Only objects in `s3://bucket/logs/` are affected.

---

### Scenario 2: Tag-Based Lifecycle

**Use case:** Let users mark files for archival by adding a tag.

**Steps:**

**Upload a file and tag it:**
1. Upload `old_report.pdf`
2. Select the file → **Actions → Edit tags**
3. Add tag: `Archive=true`
4. Save

**Create tag-based rule:**
1. Create rule: `TagArchiveRule`
2. **Scope:** Limit the scope
3. **Filter by object tags:**
   - Key: `Archive`
   - Value: `true`
4. Transitions: Immediately move to Glacier (0 days)

**Result:** Only files tagged `Archive=true` are moved to Glacier.

---

### Scenario 3: Versioning + Lifecycle (Clean Up Old Versions)

**Problem:** If versioning is enabled, every file update creates a new version. Old versions pile up, costing money.

**Solution:** Delete old versions after 30 days.

**Steps:**

**Enable versioning:**
1. Bucket → **Properties → Bucket Versioning → Edit**
2. Select **Enable**
3. Save

**Create lifecycle rule for versions:**
1. Create rule: `CleanOldVersions`
2. Scope: Apply to all objects
3. Actions:
   - ✅ **Expire noncurrent versions of objects**
   - Days after objects become noncurrent: `30`
4. Create rule

**Result:**
- Current version: Kept indefinitely
- Previous versions: Deleted after 30 days

**Example:**
- Upload `data.csv` (version 1)
- Update `data.csv` (version 2 becomes current, version 1 becomes noncurrent)
- After 30 days: Version 1 is deleted, version 2 remains

---

### Scenario 4: Abort Incomplete Multipart Uploads

**Problem:** When uploading large files (>5 GB), S3 uses multipart uploads. If the upload fails or is canceled, incomplete parts remain in the bucket (invisible but billable).

**Solution:** Automatically clean up incomplete uploads.

**Steps:**
1. Create rule: `AbortIncompleteUploads`
2. Scope: Apply to all objects
3. Actions:
   - ✅ **Delete expired object delete markers or incomplete multipart uploads**
   - Days after initiation: `7`
4. Create rule

**Result:** Any multipart upload not completed within 7 days is aborted and parts are deleted.

**How to check for incomplete uploads:**
```bash
aws s3api list-multipart-uploads --bucket my-bucket
```

---

## Cost Implications

### Storage Class Pricing (us-east-1)

| Class | Price/GB/month | Retrieval Fee | Min Storage Duration | Min Billable Size |
|-------|----------------|---------------|----------------------|-------------------|
| Standard | $0.023 | None | None | None |
| Standard-IA | $0.0125 | $0.01/GB | 30 days | 128 KB |
| Glacier Flexible | $0.004 | $0.03/GB | 90 days | 40 KB |
| Glacier Deep Archive | $0.00099 | $0.02/GB | 180 days | 40 KB |

**Important notes:**

**1. Minimum storage duration:**
If you transition to Glacier and delete after 50 days, you're charged for 90 days (the minimum).

**2. Retrieval fees:**
Glacier is cheap to store but costs money to retrieve. If you access data frequently, you'll pay more in retrieval fees than you save in storage.

**3. Small file penalty:**
A 10 KB file in Standard-IA is billed as 128 KB. Use Standard for small files.

**4. Early deletion fees:**
If you delete a file from Standard-IA before 30 days, you pay for 30 days anyway.

### Example Cost Calculation

**Scenario:** 1,000 GB of log files, 100 GB added monthly

**Without lifecycle policy (all in Standard):**
- Month 1: 100 GB × $0.023 = $2.30
- Month 12: 1,200 GB × $0.023 = $27.60/month

**With lifecycle policy:**
- 0-30 days: 100 GB Standard = $2.30
- 30-90 days: 200 GB Standard-IA = $2.50
- 90+ days: 900 GB Glacier = $3.60
- **Total:** $8.40/month (70% savings)

**Annual savings:** $230

---

## Best Practices

### 1. Test with a Prefix First
Don't apply lifecycle rules to your entire bucket immediately.
- Create a `test/` folder
- Apply rule to `test/` prefix only
- Verify behavior for a week
- Then roll out to production

### 2. Monitor Lifecycle Actions
- Enable **S3 Storage Lens** to see transitions/expirations
- Set up **CloudWatch Events** to log lifecycle actions
- Review monthly cost reports to verify savings

### 3. Document Your Retention Policy
Lifecycle rules should match your business/legal requirements:
- GDPR: Delete personal data after X days
- SOC2: Retain audit logs for Y years
- Internal: Keep backups for Z months

Create a document:
```
Data Type         | Retention Period | Storage Class Timeline
------------------|------------------|-------------------------
Application Logs  | 90 days          | 7d Standard → 30d IA → 90d Delete
Database Backups  | 7 years          | 30d Standard → 365d Glacier → 2555d Deep Archive
User Uploads      | Indefinite       | No lifecycle (or 365d → IA)
```

### 4. Avoid Over-Optimizing Small Buckets
If your bucket has <10 GB total, lifecycle policies add complexity for minimal savings.
- 10 GB Standard = $0.23/month
- Effort to set up lifecycle rules > $0.10 saved

**Rule of thumb:** Use lifecycle policies for buckets >100 GB or with high growth rates.

### 5. Combine with Intelligent-Tiering for Unpredictable Access
If you don't know the access pattern, use **S3 Intelligent-Tiering**:
- AWS automatically moves objects between access tiers
- Small monthly fee ($0.0025/1000 objects)
- No retrieval fees (unlike Glacier)

**When to use:**
- Unknown access patterns
- Mix of hot and cold data in the same prefix

### 6. Beware of Versioning Costs
Versioning is great for protection against accidental deletes, but old versions pile up.

**Best practice:**
- Enable versioning on critical buckets
- Add lifecycle rule to delete noncurrent versions after 30-90 days
- Or transition old versions to cheaper storage

---

## Monitoring and Validation

### Check Lifecycle Actions in S3 Metrics

1. Open your bucket
2. Go to **Metrics** tab
3. View:
   - **Storage bytes** (total size per storage class)
   - **Number of objects** (count per storage class)

After lifecycle rules run, you'll see objects move from Standard → IA → Glacier.

### CloudWatch Metrics

1. Open [CloudWatch Console](https://console.aws.amazon.com/cloudwatch/)
2. **Metrics → S3 → Storage Metrics**
3. Monitor:
   - `BucketSizeBytes` by storage class
   - `NumberOfObjects` by storage class

Set up alarms:
- Alert if Glacier storage grows unexpectedly (might indicate a misconfigured rule)
- Alert if objects aren't transitioning (check IAM permissions)

### S3 Storage Lens

Free dashboard showing:
- Cost per storage class
- Lifecycle rule effectiveness
- Incomplete multipart uploads

Enable in **S3 Console → Storage Lens**.

---

## Troubleshooting

### Issue 1: Objects aren't transitioning
**Symptoms:** Created rule 3 days ago, objects still in Standard.

**Causes:**
- Lifecycle rules run daily, not immediately (wait 24-48 hours)
- Rule scope doesn't match objects (check prefix/tags)
- Objects are too small (<128 KB for IA)

**Debug:**
- View rule → Check scope/filters
- Check object metadata (creation date)
- Wait 2-3 days and re-check

### Issue 2: Unexpected costs after enabling lifecycle
**Symptoms:** Bill increased instead of decreased.

**Causes:**
- **Retrieval fees:** You're accessing Glacier objects frequently
- **Minimum duration charges:** Transitioned to Glacier and deleted within 90 days
- **Small file overhead:** 1 million 10 KB files billed as 128 KB each

**Fix:**
- Review access patterns (use Standard-IA for frequently accessed data)
- Don't use Glacier for short-lived data (<90 days)
- Use Standard for files <128 KB

### Issue 3: Deleted objects still appearing in bill
**Explanation:** S3 bills based on storage-day. If you delete on day 15, you pay for 15 days of that month.

**This is normal.** Savings appear in future months.

### Issue 4: Can't retrieve Glacier object immediately
**Symptoms:** Clicked on Glacier file, got error "object is not accessible."

**Explanation:** Glacier requires restore request.

**How to retrieve:**
1. Select object → **Actions → Restore from Glacier**
2. Choose tier:
   - **Expedited:** 1-5 minutes ($0.03/GB)
   - **Standard:** 3-5 hours ($0.01/GB)
   - **Bulk:** 5-12 hours ($0.0025/GB)
3. Wait for restore notification
4. Download within 1-7 days (you specify)

---

## Real-World Example: Startup Data Lake

**Company:** E-commerce startup
**Data:** 500 GB/month clickstream logs

**Initial setup (no lifecycle):**
- Year 1: 6 TB stored
- Cost: 6,000 GB × $0.023 = $138/month
- Annual: $1,656

**After lifecycle policy:**
```
Rule: ClickstreamRetention
- 0-7 days: Standard (for real-time analytics)
- 7-90 days: Standard-IA (for ad-hoc queries)
- 90-365 days: Glacier (for compliance)
- 365+ days: Delete
```

**Steady-state costs (after 12 months):**
- 1 TB Standard (last 2 months): $23
- 3 TB Standard-IA (2-6 months): $37.50
- 2 TB Glacier (6-12 months): $8
- **Total:** $68.50/month (50% savings)
- **Annual savings:** $834

**ROI:** 10 minutes to set up rule → $834/year savings = $5,000/hour value

---

## Summary

**What you learned:**
- ✅ Automate data retention with lifecycle policies
- ✅ Reduce storage costs by 50-90%
- ✅ Apply rules to specific folders, tags, or versions
- ✅ Clean up incomplete uploads and old versions
- ✅ Avoid pitfalls (small files, retrieval fees, minimum durations)

**Key concepts:**
- **Transitions:** Move objects to cheaper storage classes over time
- **Expirations:** Automatically delete objects after retention period
- **Scope:** Target specific prefixes, tags, or versions
- **Storage tiers:** Standard → IA → Glacier → Deep Archive

**This is how data engineers manage petabytes cost-effectively.**

---

**Next:** [AWS Cost Explorer & Budgets →](./03_cost_explorer.md)
