# Glue Crawler Table Naming - Quick Reference

This guide explains why Glue creates tables with unpredictable names and how to get consistent results.

---

## The Problem

You and your classmate got different table names:
- **You:** `sales_sales` table (correct - updated with new files)
- **Classmate:** `sales_sales_data` + `sales_0_tsv` tables + deprecated `sales_sales` (different schemas)

**Why?** Glue's table naming depends on file structure and schema compatibility.

---

## How Glue Names Tables

**Algorithm:**
1. Scan all files in the S3 path
2. Group files by compatible schemas
3. For each group, create a table:
   - `{prefix}` + `{inferred_name}`
   - Inferred name comes from folder or file names

**Result varies based on:**
- Whether files have the same schema
- File naming patterns
- File formats (CSV vs TSV vs JSON)

---

## Common Scenarios

### ✅ Scenario 1: Same Schema (Best Practice)

**Your S3 structure:**
```
s3://bucket/sales/
  ├── sales_data.csv       (id, customer, product, amount, date)
  └── sales_data_ext.csv   (id, customer, product, amount, date)
```

**What Glue does:**
- Detects: Both files have identical schema ✓
- Creates: ONE table (e.g., `sales_sales` with prefix, or `sales` without)
- When you re-run crawler: Updates the same table with both files
- **This is correct! ✓**

**Your case:** This is what happened to you.

---

### ❌ Scenario 2: Different Schemas (Problem)

**Your classmate's S3 structure:**
```
s3://bucket/sales/
  ├── sales_data.csv    (id, customer, amount, date)
  └── customers.tsv     (customer, email, country)  ← Different schema!
```

**What Glue does:**
- Detects: Files have different schemas ✗
- Creates: MULTIPLE tables
  - `sales_sales_data` (from sales_data.csv)
  - `sales_0_tsv` or `sales_customers` (from customers.tsv)
- If `sales_sales` table existed: Marks it "deprecated"
- **This causes confusion!**

**Your classmate's case:** This is what happened.

---

### ❌ Scenario 3: Different File Formats

**S3 structure:**
```
s3://bucket/sales/
  ├── data.csv
  ├── data.json
  └── data.parquet
```

**What Glue does:**
- Creates separate tables: `sales_0_csv`, `sales_1_json`, `sales_2_parquet`
- Indexed naming because formats are incompatible

---

## The Table Prefix Effect

### Without Prefix (Recommended)
```
Crawler configuration: Table prefix = [empty]
Result: Table name = folder name
Example: sales/ folder → "sales" table
```
**Clean and predictable ✓**

### With Prefix (Confusing)
```
Crawler configuration: Table prefix = "sales_"
Result: Table name = "sales_" + inferred name
Possible results:
  - sales_sales (prefix + folder name)
  - sales_sales_data (prefix + file prefix)
  - sales_data (just file prefix?)
  - sales_0_csv (prefix + indexed format)
```
**Unpredictable ✗**

---

## Best Practices

### 1. One Schema Per Folder ⭐

**Do this:**
```
s3://bucket/
  ├── sales/          ← All sales files (same schema)
  │   ├── jan.csv
  │   ├── feb.csv
  │   └── mar.csv
  ├── customers/      ← All customer files (same schema)
  │   └── customers.csv
  └── products/       ← All product files (same schema)
      └── products.json
```

**Result:**
- `sales` table (contains jan, feb, mar data)
- `customers` table
- `products` table
- **Clean, predictable naming**

### 2. Skip the Table Prefix

When creating a crawler:
- **Table name prefix:** Leave empty
- Table will be named after the folder
- No redundant `sales_sales` names

### 3. Keep Schemas Consistent

**In the same folder:**
- All files must have the exact same columns
- Same data types (don't mix integer and string for same column)
- Same column order (not strictly required but recommended)

**If you need different schemas:**
- Put them in different folders
- Create separate crawlers for each folder

### 4. Same Format Per Folder

**Don't mix:**
```
sales/
  ├── data.csv   ✗
  └── data.json  ✗
```

**Instead:**
```
sales_csv/
  └── data.csv   ✓

sales_json/
  └── data.json  ✓
```

---

## When You Get Unexpected Tables

### Problem: Multiple tables when you expected one

**Diagnosis:**
1. Check if files have different schemas
   - Different columns?
   - Different data types?
   - Different formats (CSV vs TSV)?

**Solution:**
1. Go to Glue Console → Data Catalog → Tables
2. Delete the unwanted tables
3. Reorganize S3:
   - Move incompatible files to different folders
   - Ensure all files in a folder have the same schema
4. Re-run the crawler

### Problem: Table marked "deprecated"

**Cause:** Crawler detected schema changed

**Solution:**
1. Check what changed (columns added/removed/renamed?)
2. If intentional: Delete deprecated table, use new one
3. If unintentional: Fix the schema, delete tables, re-run crawler

---

## Adding More Files Later

### Same Schema → Updates Table ✓

```bash
# Initial state
s3://bucket/sales/
  └── jan.csv    (id, customer, amount)

# Run crawler → creates "sales" table

# Add more files
s3://bucket/sales/
  ├── jan.csv
  ├── feb.csv    (same schema)
  └── mar.csv    (same schema)

# Re-run crawler → updates "sales" table
# Now queries on "sales" include all three files ✓
```

### Different Schema → Creates New Table ✗

```bash
# Initial state
s3://bucket/sales/
  └── sales_data.csv    (id, customer, amount)

# Run crawler → creates "sales_sales_data" table

# Add different schema
s3://bucket/sales/
  ├── sales_data.csv
  └── customers.csv     (customer, email, country)  ← Different!

# Re-run crawler → creates "sales_customers" table
# May deprecate old table ✗
```

---

## Troubleshooting Commands

### Check what tables exist
```sql
-- In Athena
SHOW TABLES IN ecommerce_datalake;
```

### Check table schema
```sql
-- In Athena
DESCRIBE ecommerce_datalake.sales_sales;
```

### Check S3 file structure
```bash
# AWS CLI
aws s3 ls s3://my-bucket/sales/ --recursive
```

### Delete a table (Glue Console)
1. Go to Data Catalog → Tables
2. Select the table
3. Actions → Delete table
4. Re-run crawler to recreate correctly

---

## Quick Decision Tree

```
Do all files in the folder have the SAME schema?
  ├─ YES → Glue creates ONE table ✓
  │         (Updates this table when you add more files)
  │
  └─ NO  → Glue creates MULTIPLE tables ✗
            (One per schema or format)
            (May deprecate old tables)

Solution: Put different schemas in different folders!
```

---

## Real-World Recommendations

### For Learning / Small Projects
- Skip the table prefix (leave empty)
- One folder per data type
- Keep schemas simple and consistent

### For Production
- Use partitioned folders:
  ```
  sales/year=2025/month=01/day=15/data.csv
  sales/year=2025/month=01/day=16/data.csv
  ```
- Schedule crawlers to run daily/hourly
- Set up crawler to "Add new partitions" only (don't update existing)
- Use AWS Glue ETL jobs to enforce schema consistency

---

## Summary

**Key Takeaways:**
1. **Same schema in folder** → ONE table (good!)
2. **Different schemas in folder** → MULTIPLE tables (confusing!)
3. **Skip the prefix** → Cleaner table names
4. **One schema per folder** → Predictable results

**Your case (sales_sales table):**
- You did it correctly! ✓
- Files had the same schema
- Crawler updated one table
- This is the expected behavior

**Your classmate's case (multiple tables):**
- Mixed different schemas in one folder
- Crawler created separate tables for each schema
- Deprecated old table when schema changed
- Solution: Separate schemas into different folders

---

**For detailed step-by-step instructions, see:** [01_data_design.md](./01_data_design.md#understanding-glue-table-naming-important)
